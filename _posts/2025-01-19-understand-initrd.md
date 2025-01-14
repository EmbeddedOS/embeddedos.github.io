---
title: "Understand what initrd is, and why it's necessary in Linux system."
description: >-
  Let's dig deeper into of initrd concept in Linux system and the differences between initrd, ramfs, initramfs, and rootfs.

author: Cong
date: 2025-19-01 00:00:00 +0700
categories: [Linux, initrd]
tags: [initrd]
---

## 1. What is initrd?

First time heard of that, I have to admit that I thought `initrd` was some programs to initialize ram and disk drives 😛. So let's analysis this concept to avoid silly mistakes like me.

The wikipedia mentions `initrd` concept: in Linux systems, `initrd` (initial ramdisk) is a scheme for loading temporary root filesystem into memory, to be used as part of Linux startup process. Let's delve into this concept.

The ramdisk, in short, still is a block of Physical Memory, but in this case, the computer's software is treating it as a disk drive. In other words, the ramdisk is a virtual storage location that can be accessed the sam as a disk drive, but actually working on Physical Memory. The main purpose of ramdisk is to provide high performance temporary storage for demanding tasks.

The second thing was mentioned is *temporary root filesystem*. A root filesystem is the file system contained on the same disk partition on which the root directory (`/`) is located. The rootfs must contain everything needed to support a full Linux System, for example, the minimum set of directories: `/dev`, `/proc`, `/bin`, `/etc`, etc. The basic set of utilities: `sh`, `ls`, `mv`, etc. Devices: `/dev/hd*`, `/dev/tty*`, etc. And runtime libs to provide basic functions.

I have other blog that goes into the rootfs and building a simplest one, take a look here: [building rootfs](/posts/2025-01-19-building_a_rootfs).

Back to the `initrd` concept, let's summarize, this rootfs will be loaded into RAM for temporary and as a part of booting flow. *temporary* means finally we still need to load the real one?. So the questions are why we need to load a temporary rootfs into RAM? why don't we mount the real rootfs and using it directly? Let's go to next section to find out the answers.

## 2. Why we need `initrd`?
