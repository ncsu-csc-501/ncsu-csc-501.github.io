# How to set up your VM for eBPF

This document is the step-by-step manual for getting eBPF working on your VCL VM. You will use eBPF two ways. The first is through command-line tools someone else already wrote: bpftrace, the BCC tools, and bpftool. The second is through C programs you write yourself, compiled with clang and loaded with libbpf. bpftrace and BCC compile and load the BPF code for you. With libbpf you do each of those steps yourself, and that is what this lab needs. 

Use the same VCL environment, XINU+QEMU (CSC 501 new), running Ubuntu 24.04, but we recommend reserving a new VM for eBPF instead of reusing the one you built your kernel on. If your VM boots a custom kernel like `6.8.0-csc501`, the eBPF tools do not work on it. Ubuntu builds `linux-tools` (where bpftool lives) and `linux-headers` (which BCC needs) only for its own kernels, so on a custom kernel Step 2 fails with `Unable to locate package`. A fresh reservation boots the stock kernel (e.g., `6.8.0-124-generic`), and everything below assumes that kernel.

**Note**: the number in the middle depends on when the VM image was built, so your stock kernel may be `6.8.0-54-generic` or some other `6.8.0-NNN-generic`. That is fine. It is still Ubuntu's own kernel, and every command below uses `$(uname -r)` rather than a fixed version. Wherever this document shows `6.8.0-124-generic`, read it as your number.

All your eBPF work goes in `~/ebpf-lab`. Remember that when your VM reservation expires, its filesystem is deleted along with it. To keep your files, save them under `ncsudrive`.

---

## Step 1: Check the VM before you start

Run these in your VM terminal. All five, before you install anything.

```bash
uname -r                          # 6.8.0-124-generic, the stock kernel
uname -m                          # x86_64
ls -l /sys/kernel/btf/vmlinux     # must exist
grep -E "^CONFIG_(BPF_SYSCALL|BPF_JIT|DEBUG_INFO_BTF)=" /boot/config-$(uname -r)
sudo -v                           # you must be able to run sudo
```

The `grep` should print three lines, all ending in `=y`. What those options are:

- `BPF_SYSCALL` is the `bpf()` system call itself. Every tool below goes through it to load programs and create maps.
- `BPF_JIT` turns verified BPF bytecode into native x86 instructions, so your program runs about as fast as compiled kernel code.
- `DEBUG_INFO_BTF` makes the kernel describe its own types, every struct and every field offset, in `/sys/kernel/btf/vmlinux`. bpftrace reads that file to understand `task_struct`, and your C programs get `vmlinux.h` from it in Step 5.

If `uname -r` says `6.8.0-csc501`, you are on the VM that you built. Stop here and reserve a new one.

## Step 2: Install the command-line tools

```bash
sudo apt update
sudo apt install -y bpftrace bpfcc-tools python3-bpfcc \
     linux-headers-$(uname -r) linux-tools-$(uname -r)
```

What you just installed:

- `bpftrace`: a small awk-like language for one-liners. It compiles your one-liner to BPF and loads it, all in one command. It gets kernel types from BTF, so it does not need kernel headers.
- `bpfcc-tools`: ready-made tracing tools from the BCC project, installed as `/usr/sbin/*-bpfcc`. Ubuntu adds that suffix, so the tool the BCC docs call `execsnoop` is `execsnoop-bpfcc` here.
- `python3-bpfcc`: the `from bcc import BPF` module, for writing your own Python tools with the BPF code embedded as a C string. 
- `linux-headers-$(uname -r)`: BCC compiles its embedded C every time a tool starts, against these headers. Without them every BCC tool fails.
- `linux-tools-$(uname -r)`: `bpftool`, built for this exact kernel. `/usr/sbin/bpftool` is only a wrapper that runs `/usr/lib/linux-tools/$(uname -r)/bpftool`.



## Step 3: Try the command-line tools

Open a second SSH session to the VM. You run the tracer in the first terminal and do things in the second.

### bpftrace

Run:

```bash
sudo bpftrace -e 'BEGIN { printf("eBPF OK\n"); exit(); }'
```

If it prints `eBPF OK`, the kernel accepted a BPF program and ran it. 

Now, this prints every program that starts, anywhere on the machine:

```bash
sudo bpftrace -e 'tracepoint:sched:sched_process_exec { printf("%-7d %s\n", pid, comm); }'
```

Type `ls` and `date` in the second terminal:

```
Attaching 1 probe...
20481   ls
20482   date
```

Ctrl-C stops it. 

The following counts system calls per process name for as long as it runs:

```bash
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

Give it a few seconds, then press Ctrl-C. `@` is a map, a key-value table that lives in the kernel. The counting happens inside the kernel on every syscall, and bpftrace copies the table out once, when you stop it. That is why counting every syscall on the machine stays cheap.

### BCC tools

```bash
sudo execsnoop-bpfcc
```

It takes a few seconds to start, because BCC is compiling its C code against the kernel headers right then. Once the header line appears, type `ls` in the second terminal:

```
PCOMM            PID     PPID    RET ARGS
ls               20483   1493      0 /usr/bin/ls --color=auto
```

Ctrl-C stops it. `ls /usr/sbin/*-bpfcc` shows everything else you got. `opensnoop-bpfcc`, `syscount-bpfcc` and `biolatency-bpfcc` are good ones to try next.

### bpftool

bpftool does not trace anything by itself. It shows you what BPF is loaded in the kernel right now, and in Step 6 it also generates code for you.

```bash
bpftool version
sudo bpftool prog show
```

The `sd_*` entries belong to systemd, which uses BPF for cgroup device and firewall rules. Now start the syscall-counting one-liner again in the first terminal and run `sudo bpftool prog show` in the second. bpftrace's program shows up at the bottom of the list. Stop bpftrace, run it again, and the program is gone.

## Step 4: Install the C toolchain

```bash
sudo apt install -y build-essential clang llvm gcc-multilib \
     libbpf-dev libelf-dev zlib1g-dev pkg-config
```

What you just installed:

- `clang`: the compiler that turns your C into BPF bytecode, with `-target bpf`. The gcc on this VM only targets x86.
- `llvm`: `llvm-objdump`, `llvm-readelf`, and other tools for looking inside the `.bpf.o` files clang produces.
- `libbpf-dev`: the libbpf library your loader links against, plus the headers your BPF code includes (`bpf_helpers.h`, `bpf_tracing.h`, `bpf_core_read.h`), all under `/usr/include/bpf/`.
- `libelf-dev`, `zlib1g-dev`: a `.bpf.o` is an ELF file, and libbpf needs these to read it.
- `gcc-multilib`: fixes `'asm/types.h' file not found` when BPF code includes kernel UAPI headers such as `<linux/bpf.h>`.
- `pkg-config`: tells you which libbpf you have and the flags to link it.

Check it:

```bash
clang --version                   # Ubuntu clang version 18.x
pkg-config --modversion libbpf    # 1.3.0
ls /usr/include/bpf/              # bpf_helpers.h  libbpf.h  ...
```

## Step 5: Write your first eBPF program in C

A C eBPF program has two halves. The kernel half, `hello.bpf.c`, is compiled to BPF bytecode and runs inside the kernel every time its event fires. The user half, `hello.c`, is an ordinary Linux program that loads the kernel half, attaches it, and shows you what it reports. This one prints a line every time any process on the machine calls `execve`.

### Generate vmlinux.h

```bash
mkdir -p ~/ebpf-lab && cd ~/ebpf-lab
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

This turns the kernel's BTF into one big C header that declares every type in the running kernel. Your BPF code includes it instead of the kernel's own headers. No sudo needed, and you only do it once per directory.

A program compiled against this header still runs on a different kernel version. At load time libbpf compares the struct layouts your program was compiled with against the running kernel's BTF and patches the field offsets to match. This is called CO-RE, compile once, run everywhere.

### The kernel half: hello.bpf.c

Create `~/ebpf-lab/hello.bpf.c`:

```c
// hello.bpf.c: runs inside the kernel
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "GPL";

SEC("tp/syscalls/sys_enter_execve")
int handle_execve(struct trace_event_raw_sys_enter *ctx)
{
    char comm[16];
    char file[64];
    u32 pid = bpf_get_current_pid_tgid() >> 32;    // upper 32 bits hold the pid

    bpf_get_current_comm(comm, sizeof(comm));
    // args[0] is execve's filename, a pointer into user memory
    bpf_probe_read_user_str(file, sizeof(file), (const char *)ctx->args[0]);

    bpf_printk("CSC501: %s (pid %d) -> %s", comm, pid, file);
    return 0;
}
```

Four things to know about this file:

- `SEC("tp/syscalls/sys_enter_execve")` puts the function in an ELF section with that name. libbpf reads the name to decide where to attach it, here the tracepoint that fires on entry to `execve`. Every tracepoint you can use is a directory under `/sys/kernel/tracing/events/`.
- The license has to be GPL-compatible. `bpf_printk` and the probe-read helpers are GPL-only, and the kernel refuses to load a program that calls them without one.
- `ctx->args[0]` is a user-space address. You cannot dereference it from kernel code, and the verifier rejects the program if you try. `bpf_probe_read_user_str` copies the string safely, and if the page is not in memory it returns an error instead of crashing.
- `bpf_printk` writes to the kernel's trace buffer, which you read from `/sys/kernel/tracing/trace_pipe`. It is a debugging tool, like `pr_info` in the kernel lab. Real tools send their data up through maps and ring buffers.

### The user half: hello.c

Create `~/ebpf-lab/hello.c`:

```c
// hello.c: user space loader
#include <stdio.h>
#include <string.h>
#include <bpf/libbpf.h>
#include "hello.skel.h"

int main(void)
{
    struct hello_bpf *skel;
    char line[256];
    FILE *trace;

    skel = hello_bpf__open_and_load();      // the verifier runs here
    if (!skel) {
        fprintf(stderr, "load failed, did you use sudo?\n");
        return 1;
    }
    if (hello_bpf__attach(skel)) {          // hook it to the tracepoint
        fprintf(stderr, "attach failed\n");
        hello_bpf__destroy(skel);
        return 1;
    }

    trace = fopen("/sys/kernel/tracing/trace_pipe", "r");
    if (!trace) {
        perror("trace_pipe");
        hello_bpf__destroy(skel);
        return 1;
    }
    printf("attached, run commands in another terminal, Ctrl-C stops\n");
    while (fgets(line, sizeof(line), trace))    // blocks until the kernel writes
        if (strstr(line, "CSC501"))
            fputs(line, stdout);

    fclose(trace);
    hello_bpf__destroy(skel);
    return 0;
}
```

Three things to know about this file:

- `hello.skel.h` does not exist yet. bpftool generates it from the compiled `hello.bpf.o` in Step 6. It embeds the whole BPF object inside your program and gives you functions named after the file: `hello_bpf__open_and_load()`, `hello_bpf__attach()`, `hello_bpf__destroy()`.
- `open_and_load` is where the verifier checks your BPF code. If the code is unsafe, this call fails and libbpf prints the verifier's log right above your error message.
- There is no cleanup on Ctrl-C, and none is needed. When the process dies its file descriptors close, and the kernel detaches your program and frees it.

## Step 6: Build and run it

```bash
cd ~/ebpf-lab
# 1) kernel half: C into BPF bytecode
clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I. -c hello.bpf.c -o hello.bpf.o
# 2) skeleton header, generated from the object file
bpftool gen skeleton hello.bpf.o > hello.skel.h
# 3) user half: a normal C program linked against libbpf
clang -g -O2 -Wall -I. hello.c -lbpf -lelf -lz -o hello
```

The flags on the first command:

- `-target bpf`: emit BPF bytecode instead of x86 code.
- `-O2`: BPF code has to be optimized. At `-O0` the verifier rejects code it accepts at `-O2`.
- `-g`: keeps BTF type information in `hello.bpf.o`. libbpf needs it for CO-RE, and bpftool needs it to write the skeleton.
- `-D__TARGET_ARCH_x86`: tells `bpf_tracing.h` which registers hold function arguments. `hello` does not use it, but the kprobes and uprobes in the labs do, so put it on every build. It must match `uname -m`.


Now run it:

```bash
sudo ./hello
```

Type `ls` and `date` in the second terminal:

```
attached, run commands in another terminal, Ctrl-C stops
            bash-20481   [002] ...21  3521.401234: bpf_trace_printk: CSC501: bash (pid 20481) -> /usr/bin/ls
            bash-20482   [000] ...21  3523.118070: bpf_trace_printk: CSC501: bash (pid 20482) -> /usr/bin/date
```

Reading the result:

- Everything before `bpf_trace_printk:` is the trace buffer's own prefix: task name and pid, CPU, flags, timestamp. Your message comes after it. Your pids and timestamps will differ.
- It says `bash`, not `ls`. To run `ls`, your shell forks a child, a copy of bash with a new pid, and that child calls `execve`. The name changes to `ls` only when `execve` succeeds, which is after your probe fired. 

While `hello` is running, `sudo bpftool prog show name handle_execve` in the second terminal shows your program sitting in the kernel. 

### A build script for the labs

For convenience, the script below runs the three build commands for you. It is optional, and typing the commands by hand works just as well.

```bash
cat > ~/ebpf-lab/build.sh <<'EOF'
#!/bin/bash
# usage: ./build.sh NAME   builds NAME.bpf.c, then NAME.c if there is one
set -e
clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I. -c "$1.bpf.c" -o "$1.bpf.o"
if [ -f "$1.c" ]; then
    bpftool gen skeleton "$1.bpf.o" > "$1.skel.h"
    clang -g -O2 -Wall -I. "$1.c" -lbpf -lelf -lz -o "$1"
fi
EOF
chmod +x ~/ebpf-lab/build.sh
cd ~/ebpf-lab && ./build.sh hello
```


## In-Class Tutorial: Who gets the CPU

This one ties eBPF to scheduling. You pin two CPU-bound processes to the same CPU so they have to share it, and watch how the scheduler splits that CPU between them, once a second. Then you change one process's nice value and watch the split move.

The kernel half hooks `sched_switch`, the tracepoint that fires every time a CPU switches from one task (`prev`) to another (`next`). Each time, it charges the time since the last switch to `prev`. The user half prints the table once a second and clears it.

### The shared header: sched_share.h

Both halves include this, so they agree on the layout of one table entry.

```c
// sched_share.h: shared by the kernel half and the user half
#ifndef SCHED_SHARE_H
#define SCHED_SHARE_H

struct share_stat {
    unsigned long long runtime_ns;  // total time on the CPU
    unsigned long long runs;        // how many times it was switched out
    char comm[16];
};

#endif
```

### The kernel half: sched_share.bpf.c

```c
// sched_share.bpf.c: charge every run on one CPU to the task that ran
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "sched_share.h"

char LICENSE[] SEC("license") = "GPL";

// globals use plain C types: bpftool copies them into the skeleton,
// and user space has no u32 or u64
const volatile unsigned int targ_cpu = 3;   // user space sets this before load
unsigned long long start_ns;                // when the task now on targ_cpu got it

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, u32);                   // pid of the task
    __type(value, struct share_stat);
} stats SEC(".maps");

SEC("tp_btf/sched_switch")
int BPF_PROG(on_switch, bool preempt, struct task_struct *prev,
             struct task_struct *next, unsigned int prev_state)
{
    u64 now = bpf_ktime_get_ns();
    struct share_stat *s, fresh = {};
    u32 pid;

    if (bpf_get_smp_processor_id() != targ_cpu)    // watch one CPU only
        return 0;

    if (start_ns) {                     // skip the very first switch
        pid = BPF_CORE_READ(prev, pid);
        s = bpf_map_lookup_elem(&stats, &pid);
        if (!s) {                       // prev's first run this second
            BPF_CORE_READ_STR_INTO(&fresh.comm, prev, comm);
            bpf_map_update_elem(&stats, &pid, &fresh, BPF_NOEXIST);
            s = bpf_map_lookup_elem(&stats, &pid);
        }
        if (s) {                        // NULL only if the map is full
            s->runtime_ns += now - start_ns;
            s->runs++;
        }
    }
    start_ns = now;                     // next starts running now
    return 0;
}
```

Things to know about this file:

- `tp_btf/sched_switch` runs inside the scheduler, on the CPU that is switching. That is why checking `bpf_get_smp_processor_id()` is all it takes to watch one CPU. Every other CPU runs the program too, but returns at that first check.
- Only the watched CPU ever writes `start_ns`, and a CPU cannot be in two context switches at once. So a plain global variable is safe here, with no map and no lock.
- `targ_cpu` is `const volatile`. User space sets it between open and load, and from then on the verifier treats it as a constant. 
- The two globals are declared with plain C types, not `u32` and `u64`. `bpftool gen skeleton` copies every global's declaration into `sched_share.skel.h`, and the user half compiles that header without `vmlinux.h`, so a kernel type there fails with `unknown type name 'u32'`. Inside functions and map definitions, kernel types are fine.
- The idle task counts too. When the CPU has nothing to run it switches to `swapper/3`, pid 0, and that shows up as a row like any other task.

### The user half: sched_share.c

```c
// sched_share.c: once a second, print who used one CPU
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <signal.h>
#include <unistd.h>
#include <bpf/bpf.h>
#include <bpf/libbpf.h>
#include "sched_share.h"
#include "sched_share.skel.h"

#define MAX_TASKS 1024      // same as max_entries in the BPF map
#define TOP       5         // rows printed per second

struct row {
    unsigned int pid;
    struct share_stat s;
};

static struct row rows[MAX_TASKS];
static volatile bool stop;
static void on_sigint(int sig) { stop = true; }

// biggest runtime first
static int by_runtime(const void *a, const void *b)
{
    const struct row *x = a, *y = b;

    if (x->s.runtime_ns == y->s.runtime_ns)
        return 0;
    return x->s.runtime_ns < y->s.runtime_ns ? 1 : -1;
}

int main(int argc, char **argv)
{
    struct sched_share_bpf *skel;
    int cpu = argc > 1 ? atoi(argv[1]) : 3;
    int fd;

    skel = sched_share_bpf__open();
    if (!skel)
        return 1;
    skel->rodata->targ_cpu = cpu;           // set between open and load
    if (sched_share_bpf__load(skel) || sched_share_bpf__attach(skel)) {
        fprintf(stderr, "load or attach failed, did you use sudo?\n");
        sched_share_bpf__destroy(skel);
        return 1;
    }
    fd = bpf_map__fd(skel->maps.stats);
    signal(SIGINT, on_sigint);
    printf("watching CPU %d, Ctrl-C stops\n", cpu);

    while (!stop) {
        unsigned int *key = NULL;
        unsigned long long total = 0;
        int n = 0, i;

        sleep(1);

        // 1) collect every key, 2) copy each entry out and delete it,
        //    so the next second starts from zero
        while (n < MAX_TASKS &&
               bpf_map_get_next_key(fd, key, &rows[n].pid) == 0) {
            key = &rows[n].pid;
            n++;
        }
        for (i = 0; i < n; i++) {
            if (bpf_map_lookup_and_delete_elem(fd, &rows[i].pid, &rows[i].s))
                memset(&rows[i].s, 0, sizeof(rows[i].s));
            total += rows[i].s.runtime_ns;
        }
        if (!total)
            continue;

        qsort(rows, n, sizeof(rows[0]), by_runtime);
        printf("\n%-7s %-16s %6s %6s %9s\n", "PID", "COMM", "CPU%", "RUNS", "AVG RUN");
        for (i = 0; i < n && i < TOP && rows[i].s.runs; i++) {
            struct share_stat *s = &rows[i].s;

            printf("%-7u %-16s %5.1f%% %6llu %6.2f ms\n",
                   rows[i].pid, s->comm, 100.0 * s->runtime_ns / total,
                   s->runs, s->runtime_ns / 1e6 / s->runs);
        }
    }

    sched_share_bpf__destroy(skel);
    return 0;
}
```

Things to know about this file:

- Each second it first collects every key, then copies each entry out and deletes it with `bpf_map_lookup_and_delete_elem()`. It takes two loops because deleting the key you are standing on makes `bpf_map_get_next_key()` start over from the beginning.
- `CPU%` is each task's share of the time the table accounted for in that second. `RUNS` is how many times the task was switched out, and `AVG RUN` is how long it stayed on the CPU each time.

### Run it

CPUs are numbered from 0, so CPU 3 exists only if `nproc` prints 4 or more. In the first terminal, build it the same way as `hello` in Step 6:

```bash
cd ~/ebpf-lab
# 0) kernel struct definitions, only if vmlinux.h is not here yet (Step 5)
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
# 1) kernel half: C into BPF bytecode
clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I. -c sched_share.bpf.c -o sched_share.bpf.o
# 2) skeleton header, generated from the object file
bpftool gen skeleton sched_share.bpf.o > sched_share.skel.h
# 3) user half: a normal C program linked against libbpf
clang -g -O2 -Wall -I. sched_share.c -lbpf -lelf -lz -o sched_share
```

Then start watching CPU 3:

```bash
sudo ./sched_share 3
```

Nothing is competing for CPU 3 yet, so the top row is `swapper/3`, the idle task, at close to 100%.

In the second terminal, start two CPU hogs, both pinned to CPU 3:

```bash
taskset -c 3 yes > /dev/null &
taskset -c 3 yes > /dev/null &
pgrep -a yes                  # note the two pids
```

The first terminal now shows something like this. Run lengths vary from VM to VM.  The ratios are what matter.

```
PID     COMM               CPU%   RUNS   AVG RUN
20501   yes               49.9%    166   3.00 ms
20502   yes               49.8%    166   3.00 ms
34      ksoftirqd/3        0.1%      3   0.31 ms
```

If only `swapper/3` shows up, the hogs and the watcher are on different CPUs. The number after `taskset -c` and the argument to `sched_share` must match.

### Change the nice value

The kernel turns each nice value into a weight, from the `sched_prio_to_weight[]` table in `kernel/sched/core.c`: nice 0 is 1024, nice 5 is 335, nice 10 is 110. A task's share of the CPU is its weight divided by the sum of the weights of the tasks competing for it. Before you run the next command, work out what split `renice -n 5` should give.

```bash
renice -n 5 -p 20502          # use the second pid from pgrep
ps -o pid,ni,comm -p 20501,20502
```

1024 / (1024 + 335) is about 75%, so expect roughly 75/25. Then push it further:

```bash
renice -n 10 -p 20502         # 1024 / (1024 + 110): about 90/10
```

```
PID     COMM               CPU%   RUNS   AVG RUN
20501   yes               90.2%     34  26.50 ms
20502   yes                9.7%     34   2.85 ms
34      ksoftirqd/3        0.1%      3   0.31 ms
```

Reading the result:

- `CPU%` follows the weights, as predicted.
- `RUNS` stays equal. With only two tasks on the CPU, every context switch goes from one to the other, so they get the same number of turns no matter what. Nice changes how long each turn lasts.

If there is time left, try these:

- `sudo renice -n -5 -p 20501` (going below 0 needs root), after first putting 20502 back with `sudo renice -n 0 -p 20502`. Weight 3121 against 1024 gives the same 75/25 as before, just from the other side. Only the ratio of the weights matters.
- Start a third hog on CPU 3 with all three at nice 0, and the split becomes three ways, about 33% each.

When you are done, stop the hogs and then `sched_share` (Ctrl-C):

```bash
pkill -x yes
```

---
<!-- 
## When it does not work

- `Unable to locate package linux-tools-6.8.0-csc501` (or `linux-headers-...`): you are on the VM from the kernel lab. Reserve a new VM and start over from Step 1.
- `WARNING: bpftool not found for kernel ...`: the `/usr/sbin/bpftool` wrapper found no bpftool for the running kernel. Either you are on the kernel lab VM, or you skipped `linux-tools-$(uname -r)` in Step 2.
- `Unable to locate package linux-tools-6.8.0-124-generic` on a fresh VM: Ubuntu's archive has moved on to a newer kernel and no longer carries tools for this one. Check that `df -h /boot` has a few hundred MB free, run `sudo apt install -y linux-generic linux-headers-generic linux-tools-generic`, reboot (the newer kernel boots by default), and redo Step 2.
- BCC tools die with `Unable to find kernel headers`: install `linux-headers-$(uname -r)`.
- `fatal error: 'vmlinux.h' file not found`: you generated it in another directory. Run the `bpftool btf dump` line again in the directory you compile in.
- `fatal error: 'bpf/bpf_helpers.h' file not found`: `libbpf-dev` is missing (Step 4).
- `fatal error: 'asm/types.h' file not found`: `gcc-multilib` is missing (Step 4).
- `hello.skel.h: No such file or directory`: you compiled `hello.c` before running `bpftool gen skeleton`. The order in Step 6 matters.
- `unknown type name 'u32'` (or `u64`) inside a `.skel.h`: a global variable in your `.bpf.c` uses a kernel type. Declare globals with plain C types such as `unsigned int` (Step 7 explains why).
- `load failed` or `Operation not permitted`: you forgot `sudo`. Loading BPF programs needs root, or strictly, the `CAP_BPF` and `CAP_PERFMON` capabilities.
- The verifier rejected your program: libbpf prints the verifier's log between `-- BEGIN PROG LOAD LOG --` and `-- END PROG LOAD LOG --`. Read it from the bottom up. The last lines name the instruction it refused, and most of the time the cause is a pointer you used without checking it for NULL.
- `sudo ./hello` runs but prints nothing when you type commands: make sure nothing else is reading `trace_pipe`, such as a forgotten `sudo cat /sys/kernel/tracing/trace_pipe` in another terminal. Each line goes to only one reader, so two readers steal lines from each other.

--- -->

## Cleaning up

On the kernel side, nothing from this document outlives the process that loaded it. A program loaded by `hello`, bpftrace or a BCC tool is freed when that process exits. The exception is a program you pin under `/sys/fs/bpf`, as Lab 1 does. A pinned program stays until you `sudo rm` the pin or reboot. To check that nothing is left over:

```bash
sudo bpftool prog show      # only systemd's sd_* entries should be left
```

The packages from Steps 2 and 4 are harmless to leave installed.
