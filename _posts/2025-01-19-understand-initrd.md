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

To answer this question, let's discuss these scenarios, where another component outside the kernel is needed:

1. For many Linux distributions, *kernel image is generic* that is created to boot on a wide variety of hardware. The Device Drivers for this generic kernel are included as loadable kernel modules (runtime kernel modules) because so many static drivers can cause larger kernel image, or might lead to other problems like conflicting hardware. But some modules are necessary at boot time, for example, to mount the rootfs, so we can not store these modules in the rootfs. In this situation, a temporary rootfs is a good choice to store various kind of modules.

2. Another situation is that the rootfs might lie on complex or encrypted partition that require preparations to mount. Kernel cannot decrypt or mount the rootfs itself. Because there are so many encrypting algorithm. With an temporary rootfs, we can store something needed to decrypt or mount the necessary components before switching to the actual rootfs.

3. Some root partition is identified by a UUID or label, so now hardcoding these identifies in kernel is terrible, in the case, the `initrd` can be used to look up this information and mount the correct rootfs.

4. Do early system configuration step like setting up network interfaces, can be done before fully mounting the rootfs.

5. The `initrd` is quite small that provide capability to load by bootloader. After kernel take control, it can run user-space helpers, load needed module to mount real rootfs, even from a different device.

So let's summarize, the use of `initrd` is to separate initial boot stages from kernel, by that we can avoid having to hardcode handling for so many special cases into kernel. In other word, `initrd` is also called as an **early user-space**. By that we can keep the kernel generic as much as possible and focus to its jobs. Sound like a bit of single responsibility principle here, right 😛?

## 3. Other concepts

There are other concepts that might confuse you because of same purposes but with a slightly different. Let's discover some of them.

### 3.1. ramfs

A very simple filesystem that

### 3.2. rootfs

Other blog the go deeper into the rootfs concept here: [building rootfs](/posts/building-a-rootfs/)

### 3.3. initramfs

## 4. Build a initrd image

<https://docs.kernel.org/admin-guide/initrd.html>

## 5. How that work?
