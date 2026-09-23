# How to build and boot your own Linux kernel

This document is the step-by-step manual for building and booting your own Linux kernel. You can download the Linux 6.8 source on your VCL VM, make some changes, compile the whole source tree, install the new kernel beside the kernel you already have, and boot into it. Follow the steps in order, top to bottom.

The VCL environment you reserve for this course, XINU+QEMU (CSC 501 new), boots Ubuntu on the stock `6.8.0-124-generic` kernel. That is your starting point, and `uname -r` will confirm it the moment you log in. If yours prints a different version, everything below still works, just substitute your version wherever `6.8.0-124-generic` appears.

---

## Step 1: Check the VM before you start

Run these in your VM terminal. All five, before you download anything.

```bash
uname -r        # 6.8.0-124-generic, the stock kernel
nproc           # 4 or more vCPUs
free -h         # 4 GB RAM minimum, 8 GB is comfortable
df -h /         # you need at least 30 GB free
sudo -v         # you must be able to run sudo
```

## Step 2: Install the build toolchain

```bash
sudo apt update
sudo apt install -y build-essential libncurses-dev bison flex \
     libssl-dev libelf-dev libdw-dev dwarves zstd
```

What you just installed:

- `build-essential`: gcc, make, and the C headers.
- `bison`, `flex`: the parser generators that the kernel's own config and DTC tools are
  built with.
- `libncurses-dev`: the `menuconfig` text UI.
- `libssl-dev`: module and kernel image signing.
- `libelf-dev`, `dwarves`: pahole, which generates the BTF type information.
- `zstd`: compresses modules and the initramfs.

If apt says the `dwarves` package does not exist, you are on an older release: install `pahole` instead. If it cannot find anything at all, you skipped `apt update` or the VM is offline. 

## Step 3: Download and extract the 6.8 source

Work in your home directory. 

```bash
cd ~
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.8.tar.xz
tar xf linux-6.8.tar.xz        
cd linux-6.8
ls            # Makefile init/ kernel/ mm/ fs/ drivers/ arch/
```

What you downloaded is mainline 6.8, which is what Ubuntu's `6.8.0-124` is based on. Ubuntu carries its own patches on top, so the two trees are close but not identical. That is fine for this lab.

Four directories carry most of an OS course: `init/` for early boot, `kernel/` for the scheduler and system calls, `mm/` for memory, `fs/` for filesystems.

## Step 4: Start from the config your VM already boots

Do not try to answer the config questions from scratch. Copy the config Ubuntu built your running kernel with, then change four options.

```bash
cp /boot/config-6.8.0-124-generic .config
make olddefconfig                                    # accept defaults for new 6.8 options
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
scripts/config --disable DEBUG_INFO
scripts/config --set-str LOCALVERSION "-csc501"
make olddefconfig                                    # re-resolve after the edits
```

What those four options are:

- `SYSTEM_TRUSTED_KEYS` holds the certificates the kernel trusts for module signatures, and `SYSTEM_REVOCATION_KEYS` holds the ones it rejects. In Ubuntu's config both point at Canonical's key files.
- `DEBUG_INFO` builds DWARF debug symbols into every object file.
- `LOCALVERSION` is the string appended to the kernel version, so `uname -r` reads `6.8.0-csc501`.

## Step 5: Make your change

Open `init/main.c` and find `start_kernel()`. In vim, that is `vim init/main.c` then
`/start_kernel(void)` and Enter to jump to it.

```c
asmlinkage __visible __init __no_sanitize_address __noreturn __no_stack_protector void start_kernel(void)
{
        ...
        pr_notice("%s", linux_banner);
        pr_info("CSC501: hello from YOURNAME's kernel\n"); //Add this line
        early_security_init();
```

Four things to know about that one line:

- `pr_info` is `printk` at the KERN_INFO level. The kernel links against no libc, so `printf` does not exist here.
- Put it right after the banner. Your line then lands next to the "Linux version ..." line in `dmesg`, where it is easy to find.
- Keep the trailing `\n`. printk holds a partial line in its buffer until it sees a newline.
- Use your own name. By the end of practice every student's `dmesg` line should read differently. Whatever string you use, use the same one when you grep in Step 8.

## Step 6: Compile

Now compile. Be warned that this can take a few hours.

- The build runs far longer than an idle SSH session usually survives, and when the connection drops your shell dies and takes `make` with it. Run it inside `screen`, which keeps the build alive on the VM whether or not you are still connected:

  ```bash
  sudo apt install -y screen     # if it is not there already
  screen -S build                # opens a session named build
  make -j$(nproc) 2>&1 | tee ~/build.log
  # press Ctrl-a then d to detach, and you can close the terminal
  screen -r build                # reattach later to see how far it got. Or you could do `tail -f build.log` instead without reattching to the screen
  ```

- `-j$(nproc)` runs one compile job per core. Without it the build is single threaded and takes hours.
- `tee` keeps the whole output in `~/build.log`. When something fails you grep that file instead of scrolling the terminal: `grep -n -i "error" ~/build.log | head`. The first error is the real one, everything after it is fallout.
- Never run `make` with sudo. Compiling is unprivileged work, only Step 7 needs root, and a root-owned object file will break your next build.

```bash
...
  LD      vmlinux
  OBJCOPY arch/x86/boot/bzImage
Kernel: arch/x86/boot/bzImage is ready  (#1)
```

That last line is your success signal.



## Step 7: Free up /boot, then install

### First, make room in /boot

`make install` writes a new `vmlinuz` plus a freshly generated initrd, and that initrd alone runs 50 to 100 MB. 
  - What is `initrd`? A temporary root filesystem loaded into RAM at boot. It holds the drivers and modules the kernel needs to find and mount the real root disk.
  - What is `vmlinuz`? The compressed kernel image itself, the file GRUB loads and runs. It is a copy of the `bzImage` you just built.
  
`/boot` is a small partition, and on a stock Ubuntu VM it is already carrying initrd images you have no use for. If it fills up mid-install you get a half-written kernel and a GRUB entry that does not boot.

Delete them first:

```bash
sudo rm /boot/initrd.img-5.15.0-106-generic
sudo rm /boot/initrd.img-6.8.0-111-generic
df -h /boot                    # you want a few hundred MB free before you continue
```

The 5.15 image is a leftover from a kernel this VM no longer runs. Neither image does anything for this lab. 

### Then install

This is the only part that needs root. Run the commands in this order.

```bash
sudo make modules_install      # -> /lib/modules/6.8.0-csc501/
sudo make install              # -> /boot/, builds initrd, adds GRUB entry
ls /boot | grep csc501
  config-6.8.0-csc501
  initrd.img-6.8.0-csc501
  vmlinuz-6.8.0-csc501
sudo update-grub               # only if make install did not do it for you
```

- `modules_install` copies every `.ko` into `/lib/modules/6.8.0-csc501`. - `make install` copies the image into `/boot`, generates a matching initramfs, and adds a GRUB menu entry. It never touches `6.8.0-124-generic`.

- Confirm all three `/boot` files exist before you reboot. A missing `initrd.img` is the single most common reason a freshly built kernel will not boot.

## Step 8: Reboot into your kernel and prove it


```bash
sudo reboot

$ uname -r
6.8.0-csc501
$ sudo dmesg | grep CSC501
[    0.000000] CSC501: hello from Yoon's kernel
```

Reading the result:

- If `uname -r` still says `6.8.0-124-generic`, you booted the old entry. 

---


## Undoing all of it

Nothing you did today is permanent. `6.8.0-124-generic` is still in GRUB and still bootable, and everything you produced lives in three places: `~/linux-6.8`, `/lib/modules/6.8.0-csc501`, and `/boot/*6.8.0-csc501*`.

```bash
sudo rm -rf /lib/modules/6.8.0-csc501
sudo rm /boot/*6.8.0-csc501*
sudo update-grub               # boot the distro kernel first, not this one
```

If you also removed the stock initramfs in Step 7 and want it back:

```bash
sudo update-initramfs -c -k 6.8.0-124-generic
sudo update-grub
```
