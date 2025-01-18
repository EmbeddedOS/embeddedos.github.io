---
title: "Understand what initrd is, and why it's necessary in Linux system."
description: >-
  Let's dig deeper into of initrd concept in Linux system and the differences between initrd, ramfs, initramfs, and rootfs.

author: Cong
date: 2025-01-19 00:00:00 +0800
categories: [Linux, initrd]
tags: [initrd]
---

## 1. What is initrd?

First time heard of that, I have to admit that I thought `initrd` was some programs to initialize ram and disk drives 😛. So let's analysis this concept to avoid silly mistakes like me.

The wikipedia mentions `initrd` concept: in Linux systems, `initrd` (initial ramdisk) is a scheme for loading temporary root filesystem into memory, to be used as part of Linux startup process. Let's delve into this concept.

The ramdisk, in short, still is a block of Physical Memory, but in this case, the computer's software is treating it as a disk drive. In other words, the ramdisk is a virtual storage location that can be accessed the sam as a disk drive, but actually working on Physical Memory. The main purpose of ramdisk is to provide high performance temporary storage for demanding tasks.

The second thing was mentioned is *temporary root filesystem*. A root filesystem is the file system contained on the same disk partition on which the root directory (`/`) is located. The rootfs must contain everything needed to support a full Linux System, for example, the minimum set of directories: `/dev`, `/proc`, `/bin`, `/etc`, etc. The basic set of utilities: `sh`, `ls`, `mv`, etc. Devices: `/dev/hd*`, `/dev/tty*`, etc. And runtime libs to provide basic functions.

I have other blog that goes into the rootfs and building a simplest one, take a look here: [building rootfs](/posts/building-a-rootfs/).

Back to the `initrd` concept, let's summarize, this rootfs will be loaded into RAM for temporary and as a part of booting flow. *temporary* means finally we still need to load the real one?. So the questions are why we need to load a temporary rootfs into RAM? why don't we mount the real rootfs and using it directly? Let's go to the next section to find out the answers.

## 2. Why we need initrd?

The key reasons why the initrd is needed in Linux system:

1. For many Linux distributions, *kernel image is generic* that need to boot on a wide variety of hardware. The Device Drivers for this generic kernel are included as loadable kernel modules (runtime kernel modules) because so many static drivers can cause larger kernel image, or might lead to other problems like conflicting hardware. The static-compiled kernel module approach also leaves modules in kernel memory which are no longer used or needed. by storing these runtime modules in `initrd`, we separate them from kernel, kernel then can load them to access the rootfs.

2. The rootfs might lie on complex or encrypted partition that require preparations to mount. The `initrd` can contains utilities to decrypt or mount the necessary components before switching to the actual rootfs.

3. Some root partition is identified by a UUID or label, the `initrd` can be used to look up this information and mount the correct rootfs.

4. Do early system configuration step like setting up network interfaces, can be done before fully mounting the rootfs.

5. The `initrd` is quite small that provide capability to load by bootloader. After kernel take control, it can run user-space helpers, load needed module to mount real rootfs, even from a different device.

So let's summarize, the use of `initrd` is to separate various kind of initial boot stage from kernel, by that we can avoid having to hardcode handling for so many special cases into kernel. In other word, an **early use-space**, a **temporary rootfs**, is used to separate various kind of hardware booting from kernel, keep the kernel generic as much as possible.

## 3. Other concepts

<https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html>

### 3.1. ramfs

### 3.2. initramfs

### 3.3. rootfs

## 4. Build a initrd image

<https://docs.kernel.org/admin-guide/initrd.html>

## 5. How that work?
