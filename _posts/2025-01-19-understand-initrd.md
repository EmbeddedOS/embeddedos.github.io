---
title: "Understand initrd, build a minimum initrd image for Linux system."
description: >-
  Let's dig deeper into of initrd concept in Linux system and the differences between initrd, ramfs, initramfs, and rootfs.

author: Cong
date: 2025-01-19 00:00:00 +0800
categories: [Linux, initrd]
tags: [initrd]
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

### 5.1. The cpio images

Why the `initrd` images are built as cpio images,

<https://www.linuxfromscratch.org/blfs/view/systemd/postlfs/initramfs.html>

cpio images.

<https://docs.kernel.org/admin-guide/initrd.html>
