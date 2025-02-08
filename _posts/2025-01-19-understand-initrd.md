---
title: "Discover the initrd concept in Linux system."
description: >-
  Let's dive deeper into the concept of initrd, explore why it's needed, and how the Linux Kernel boots with this `temporary rootfs`.

author: Cong
date: 2025-01-19 00:00:00 +0800
categories: [Linux, initrd]
tags: [initrd]
image:
  path: assets/img/rootfs.jpeg
  alt: Rootfs.
---

## 1. What is initrd?

First time heard of that, I have to admit that I have thought `initrd` was some programs that are used to initialize ram and disk drives 😛. So let's analysis this concept to avoid silly mistakes like me.

The wikipedia mentions `initrd` concept: in Linux systems, `initrd` (initial ramdisk) is a scheme for loading temporary root filesystem into memory, to be used as part of Linux startup process. Let's delve into this concept.

The ramdisk, in short, still is a block of Physical Memory, but in this case, the computer's software is treating it as a disk drive. In other words, the ramdisk is a virtual storage location that can be accessed the sam as a disk drive, but actually working on Physical Memory. The main purpose of ramdisk is to provide high performance temporary storage for demanding tasks.

The second thing was mentioned is *temporary root filesystem*. A root filesystem is the file system contained on the same disk partition on which the root directory (`/`) is located. The rootfs must contain everything needed to support a full Linux System, for example, the minimum set of directories: `/dev`, `/proc`, `/bin`, `/etc`, etc. The basic set of utilities: `sh`, `ls`, `mv`, etc. Devices: `/dev/hd*`, `/dev/tty*`, etc. And runtime libs to provide basic functions.

I have other blog that goes into the rootfs and building a simplest one, take a look here: [building rootfs](/posts/building-a-rootfs/).

Back to the `initrd` concept, let's summarize, this rootfs will be loaded into RAM for temporary and as a part of booting flow. *temporary* means finally we still need to load the real one?. So the questions are why we need to load a temporary rootfs into RAM? why don't we mount the real rootfs and using it directly? Let's go to the next section to find out the answers.

## 2. Why we need initrd?

To answer this question, let's discuss this scenario. After kernel is booted, and the rootfs (/) mounted, programs can run and further runtime-kernel-modules can be loaded. But to mount the rootfs, some conditions need to be met:

- The kernel needs the corresponding drivers to access the device on which the rootfs is located (especially SCSI drivers). Some solutions are suggested:
  - Kernel will contain all these drivers, but those drivers might be conflict to each other.
  - Also if the kernel contains all of drivers, it becomes very large.
  - The idea about different kernels for each kind of these things appear, but the problem is it becomes so hard to maintain and optimize, the kernel becomes specific when it should be generic.
- The rootfs might be encrypted. In this case a password or key is needed to decrypt and mount the filesystem.

All of these things can be resolved by the concept of `initrd`: running user space programs even before the rootfs is mounted. By that, the system startup is able to occur in two phases, where the kernel comes up with a minimum set of compiled-in drivers whereas additional modules are loaded from initrd.

## 3. Boot flow with initrd

1. The bootloader loads the kernel and `initrd` to memory and pass the control to kernel. Loading `initrd` actually has no different with loading a kernel that uses boot services to load data to RAM from the boot medium. If the bootloader can load kernel, it is also able to load `initrd` image. Special drivers are not required.
2. The bootloader informs the kernel that an `initrd` exists and where it is located in memory (through kernel parameters).
3. If the `initrd` was compressed, the kernel decompresses and mounts it as a temporary rootfs.
4. The `init` (or `linuxrc`) program is then started, that call now do all the things necessary to mount the real rootfs (using `pivot_root()` syscall).
5. As soon as the `init` program finishes, it execs the `init` on the new rootfs. The temporary rootfs is unmounted and the boot process continues as normal with the mount of the real rootfs.

> The `init` program in temporary rootfs (from `initrd` image) is different with the real rootfs. The one in `initrd` takes care loading real rootfs part, whereas the one in real rootfs is up to user's applications.
{: .prompt-info }
> The filesystem under `initrd` can continue to be accessible if mount point `/initrd` is there.
{: .prompt-info }

## 4. Other concepts

Let's take a look to other concepts that might confuse you:

- `ramfs` - A simple dynamically resizable RAM-based file system. The data is saved in RAM only and there is no backing store for it. Thus, if we unmount a `ramfs` mounting point every file which is stored there is lost. We can keep writing to the fs until we fill up the entire physical memory, but the VM can't free it because the VM thinks that files should get written to backing store.
- `ramdisk` - An older version of `ramfs` that create a fake block device out af an area of RAM, the device is used as backing store to fs. The problem with this is waste memory and CPU for unnecessary works. It also required a fs driver to format and interpret data.
- `tmpfs` - A newer version of `ramfs` that adds size limits, and the ability to write the data to swap space.
- `rootfs` - A temporary `ramfs` or `tmpfs` that is replaced by the actual rootfs from physical store.
- `initramfs` - A newer version of `initrd` concepts. Starting with kernel 2.5.x, the old `initrd` protocol is getting (replaced/complemented) with the new `initramfs` protocol. So at this moment when we are talking about `initrd` we are actually referring to `initramfs` concept.

## 5. The initrd image

### 5.1. Why the initrd images are built as compressed cpio images?

The idea is that the `initrd` need to unpacked by the kernel during boot, it means, the kernel has to include at least one format extractor to do that. So which format archive should be choose? tar, cpio, or file system types?

Some of the reasons why cpio is chosen:

- This is kernel internal format, kernel have no depends on others, and could easily make something new. Kernel provides its own tools to create and extract this format anyway, so using an existing standard was preferable, but not essential.
- cpio is a standard.
- The code to extract in kernel is simpler and cleaner than any of the various `tar` archive formats. The complete `initramfs` archive format is explained in `buffer-format.rst`, created in `usr/gen_init_cpio.c` and extracted in `init/initramfs.c`. All three together come to less than 26K total text.

But why we need to compress the cpio images?

The answer is quite simple, we need it to be as small as possible to fit within the limited RAM and resource during early boot stage. `gzip` provides a good balance between compression ration and decompression speed, so it might be the best choice for us.

### 5.2. The cpio format

cpio (copy in and out) is a general file archiver utility and its associated file format.

#### 5.2.1. Archive Creation

cpio reads file and directory path names from its standard input channel and writes the resulting archive byte stream to its standard output. Cpio is therefore typically used other utilities that generate the list of files to be archived, such as `find` command.

The resulting cpio archive is a sequence of files and directories concatenated into a single archive, separated by header sections with file meta information, such as filename, inode number, ownership, permissions, and timestamps.

```text
 ________________
| File1 Metadata |
|________________|
| File1 Content  |
|                |
|________________|
| File2 Metadata |
|________________|
| File2 Content  |
|                |
|________________|
| Dir1 Metadata  |
|________________|
| ...            |
|________________|
```

This below example using `find` to list all entries in current folder and pass through `cpio`'s standard input. The `cpio` and then stream bytes to its standard output as a file `/path/archive.cpio`.

```bash
find . -depth -print | cpio -o > /path/archive.cpio
```

> cpio does not compress any content, but resulting archives are often compressed using `gzip`.
{: .prompt-info }

```bash
find . -depth -print | cpio -o | gzip > /path/archive.cpio.gz
```

#### 5.2.2. Extraction

cpio reads an archive from its standard input and recreates the archived files in the OS's file system. In `initrd` case, extracting is already taken care by the kernel itself.

### 5.3. Making an initrd image

#### 5.3.1. How kernel run the init program

After mounting the initrd, the kernel try to run the `init` program. Take a look to this piece of kernel code:

```c
asmlinkage __visible __init __no_sanitize_address __noreturn __no_stack_protector
void start_kernel(void)
{
    ...
    /* Do the rest non-__init'ed, we're now alive */
    rest_init();
}

static noinline void __ref __noreturn rest_init(void)
{
    ...
    pid = user_mode_thread(kernel_init, NULL, CLONE_FS);
    ...
}

static int __ref kernel_init(void *unused)
{
    ...
    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
        if (!ret)
            return 0;
        pr_err("Failed to execute %s (error %d)\n",
               ramdisk_execute_command, ret);
    }
    ...
    if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init") ||
        !try_to_run_init_process("/bin/init") ||
        !try_to_run_init_process("/bin/sh"))
        return 0;

    panic("No working init found.  Try passing init= option to kernel. "
          "See Linux Documentation/admin-guide/init.rst for guidance.");
}
```

The `Arch` kernel boot code run first, and then jump to `start_kernel()` that initialize stuffs, do mount the initrd image, and then run the first user mode thread `kernel_init()`. This function will try to run the `init` program in current rootfs. Start by trying to run `ramdisk_execute_command` program, and then `/sbin/init`, `/etc/init` and so on. To dive deeper into the kernel boot process on AArch64, visit this blog [minimal rootfs](/posts/Linux-Kernel-booting-with-Aarch64/)

> The `ramdisk_execute_command` variable have the default value as `/init`, the user can replace this variable's value by passing kernel parameters `initrd=<path>`. In case `noinitrd` is passed to kernel parameters, this value will be `NULL`. As a result `/sbin/init` will be tried first.
{: .prompt-info }
> Kernel executes the `init` program as its first process that is not expected to exit.
{: .prompt-info }

#### 5.3.2. Contents of initrd

An `initrd` is a complete self-contained rootfs for Linux. A rootfs normally includes basic file system structure, minimum set of directories (`/dev`, `/proc`, `/etc`, `/bin`, etc.), utilities, shared libs, devices, etc. Take a look to this blog to explore what is rootfs, and build a simple one: [minimal rootfs](/posts/building-a-rootfs/).

To understand how `initrd` we don't need to build a complete rootfs. Just remember, the kernel will looking for `/init` program, right?. So we build a rootfs with only one file `/init`, and then compress into an compressed cpio image.

```bash
cat > init.c << EOF
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
  printf("Hi initrd!\n");
  sleep(999999999);
}
EOF
aarch64-none-linux-gnu-gcc -static init.c -o init
echo init | cpio -o -H newc | gzip > test.cpio.gz
```

Boot the system with QEMU:

```bash
qemu-system-aarch64 -kernel linux/arch/arm64/boot/Image -initrd test.cpio.gz -machine virt -cpu cortex-a53 -m 1G -nographic
```

> In fact, the `/init` should do more tasks such as loading more drivers, mounting the real rootfs, and run the init program and that.
{: .prompt-info }

## 6. Some tricky questions

### 6.1. Can kernel boot without initrd?

Of course, yes, `initrd` is optional and the kernel can mount rootfs directly by either having fixed rootfs name  built-in kernel or passing the rootfs name via the kernel parameter `root=<path>`. The needed drivers for mounting the rootfs also are required to build into kernel image.

### 6.2. Is it possible to use initrd as the final rootfs?

Well, It depends on your purposes. The `initrd` is based on RAM, so everything you write to this filesystem is temporary, that will lost when you reboot the system. So if your requirements are testing temporary things, or expect higher performance filesystem access, etc, you can use `initrd` as the final rootfs. Another use case is that in embedded systems, [DBAN](https://dban.org/), that could package anything they needs into the `initrd` and just never do the switching. Otherwise the real rootfs that based on a disk device might be better.

### 6.3. How the kernel mount the initrd?

Start by loading `populate_rootfs()` module.

```c
static int __init populate_rootfs(void)
{
    initramfs_cookie = async_schedule_domain(do_populate_rootfs, NULL,
                         &initramfs_domain);
    usermodehelper_enable();
    if (!initramfs_async)
        wait_for_initramfs();
    return 0;
}
rootfs_initcall(populate_rootfs);
```

### 6.4. What if the init program exit?

Kernel panic when we try to call `exit()` in the `init` program.

```text
[    1.622477] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000000
[    1.623562] CPU: 0 UID: 0 PID: 1 Comm: init Not tainted 6.13.0-rc5-00163-gab75170520d4-dirty #4
[    1.625479] Hardware name: linux,dummy-virt (DT)
[    1.626507] Call trace:
[    1.627129]  show_stack+0x18/0x24 (C)
[    1.628325]  dump_stack_lvl+0x60/0x80
[    1.628974]  dump_stack+0x18/0x24
[    1.629515]  panic+0x168/0x360
[    1.629785]  make_task_dead+0x0/0x17c
[    1.630240]  do_group_exit+0x34/0x9c
[    1.630655]  __arm64_sys_exit_group+0x18/0x20
[    1.631410]  invoke_syscall+0x48/0x104
[    1.632025]  el0_svc_common.constprop.0+0x40/0xe0
[    1.632869]  do_el0_svc+0x1c/0x28
[    1.633456]  el0_svc+0x30/0xcc
[    1.633797]  el0t_64_sync_handler+0x10c/0x138
[    1.634498]  el0t_64_sync+0x198/0x19c
[    1.635991] Kernel Offset: disabled
[    1.636681] CPU features: 0x000,00000c00,00800000,0200421b
[    1.637640] Memory Limit: none
[    1.638805] ---[ end Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000000 ]---
```

The code that checking for global init task before exiting a process. The function `is_global_init()` is literally comparing the process id with `1`.

```c
void __noreturn do_exit(long code)
{
    ...
    if (unlikely(is_global_init(tsk)))
        panic("Attempted to kill init! exitcode=0x%08x\n",
            tsk->signal->group_exit_code ?: (int)code);
}
```

### 6.5. Embed an initrd image into a Linux kernel

The config option `CONFIG_INITRAMFS_SOURCE` (in General Setup in `menuconfig`, and living in `usr/Kconfig`) can be used to specify a source for the initramfs archive, which will automatically be incorporated into the resulting binary.
