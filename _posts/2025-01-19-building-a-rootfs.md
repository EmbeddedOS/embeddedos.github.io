---
title: "Building a root filesystem for Linux system"
description: >-
  Understand the rootfs concept, how to build a minimum one, and discover other common tools like Busybox, Yocto, Buildroot.

author: Cong
date: 2025-01-19 00:01:00 +0800
categories: [Linux, rootfs]
tags: [rootfs]
image:
  path: assets/img/rootfs.jpeg
  alt: Rootfs.
---

## 1. File system

*A File system is a structure used by an OS to organize and manage files on a storage device. It defines how data is stored, accessed and organized on the storage device*. Analysis this concept, we learn two points here. First, storage devices have no idea what are files, directories, etc, they literally storages information and data. To access these data, we normally do that via a controller. This is done either through memory-mapped access to the controller or via input/output ports. Each controller has it's own set of commands, requirements, etc.

Secondly, *the structure is used by an OS to organize and manage files*. The OS, in this sentence, act like a interface, that receives file operation commands from user and operates the storage device follow a well-known format. So not only the OS can access to the storage, others also can operate if follow the same format structure.

```text
 _______________________________
| User space                    |
|             |App1| |App2|     |
|_______________||______________|
                ||
    Read/write/open/close file requests.
                ||
 _______________||_______________
| Kernel space  ||               |
|               \/               |
|     |Virtual File System|      |
|               ||               |
|               \/               |
|     |Block device driver|      |
|_______________||_______________|
                ||
    Organizing data follow the file system format.
 _______________||_______________                     ___________
| Hardware      \/               |   /-------------->|Block 1    |
|   |Storage device controller|  |  /                |Block 2    |
|               \/      _________|_/                 |...        |
|         |Mass Storage|>__________                  |           |
|________________________________| \---------------->|___________|
```

### 1.1. Disk Partitioning

A disk partition is a section of a hard disk drive that is formatted to specific types. So why we have to divide the storage device to multiple partitions? A storage device is divided into partitions to organize data by dedicating specific sections of the drive to different purposes. For example, you want to isolate system files and personal data to different partitions. Or the disk can install multiple Operating systems, allowing you to boot into whichever you want. Accessing to a separate partition also improve high performance due to you don't have to travel across the entire disk. Other use case is that, you want some partitions will be secured from others.

```text
 _______________________________________________________________________
|Disk                                                                   |
||Partition 1||Partition 2||Partition 3||Partition ...||Partition n|____|
              /           \ \__FAT32__/                 \_MS_DOS__/
  ___________/             \________________________
 /__________________________________________________\
| Ext4 format                                        |
|MBR|Block 0|Block 1|Block ...|Block n|Unused sectors|
 ___/       \__________________________________
|Headers + metadata + inode_tables| Data blocks|
```

Storage device stores information about its partition into a partition table that will be accessed by the OS to manages them.

Each partition can be formatted with a file system or as a swap partition.

### 1.2. Attributes

- `filename` - identifies a file. A filename is unique so that an application can refer to exactly one for a particular name.
- `directory` - Filesystem typically support organizing files into directories.
- `metadata` - name, size, time, owner, permissions, file attributes, device types, etc. A file system stores associated metadata separate from the content of the file.

```text
user application

user -> /home/user1/file.txt
                ||
                \/
        Virtual File System---> inode 123
                                    ||
                                    ||
 ___________________________________||____________________________________
disk partition                      ||
      //============================//
      ||                   //==============\\=======================\\
   ___\/_______________   //   data blocks  \/                       \/
  | inode 123 metadata |  ||  |__1_____|_2______|_...____|_______|_n____|
  | owner              |  ||  |________|________|________|_______|______|
  | permissions        |  ||  |________|________|________|_______|______|
  | types: file, dir   |  ||  |________|________|________|_______|______|
  | data blocks        |=//   |________|________|________|_______|______|
  | ...                |      |________|________|________|_______|______|
  |____________________|      |________|________|________|_______|______|

```

### 1.3. file system types

<https://en.wikipedia.org/wiki/File_system>

- Disk file systems: FAT (FAT12, FAT16, FAT32), exFAT, NTFS, ReFS, HFS and HFS+, HPFS, APFS, UFS, ext2, ext3, ext4, XFS, btrfs, Files-11, Veritas File System, VMFS, ZFS, ReiserFS, NSS and ScoutFS.
- Flash file systems.
- Network file systems.
- Special file systems: virtual filesystems.

### 1.4. Virtual file system

<https://en.wikipedia.org/wiki/Virtual_file_system>

### 1.5. File system implementation

<https://en.wikipedia.org/wiki/File_system>

## 1. What is root file system?

<https://tldp.org/HOWTO/Bootdisk-HOWTO/buildroot.html>
<https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html>

Building a simple rootfs with busybox

```bash
cd busybox/
make -j4 ARCH=arm64 CROSS_COMPILE=../toolchain/bin/aarch64-none-linux-gnu- defconfig
make -j4 ARCH=arm64 CROSS_COMPILE=../toolchain/bin/aarch64-none-linux-gnu- menuconfig
make -j4 ARCH=arm64 CROSS_COMPILE=../toolchain/bin/aarch64-none-linux-gnu-
make -j4 ARCH=arm64 CROSS_COMPILE=../toolchain/bin/aarch64-none-linux-gnu- install
cd ../
mkdir -p rootfs/{bin,sbin,etc,proc,sys,usr/{bin,sbin}}
cp -av busybox/_install/* rootfs/
cd rootfs
ln -sf bin/busybox init
mkdir -p etc/init.d/
cat > etc/init.d/rcS << EOF
mount -t sysfs none /sys
mount -t proc none /proc
EOF
chmod -R 777 etc/init.d/rcS
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../rootfs.cpio.gz
```

\Documentation\driver-api\early-userspace\early_userspace_support.rst

```text
How does it work?
=================

The kernel has currently 3 ways to mount the root filesystem:

a) all required device and filesystem drivers compiled into the kernel, no
   initrd.  init/main.c:init() will call prepare_namespace() to mount the
   final root filesystem, based on the root= option and optional init= to run
   some other init binary than listed at the end of init/main.c:init().

b) some device and filesystem drivers built as modules and stored in an
   initrd.  The initrd must contain a binary '/linuxrc' which is supposed to
   load these driver modules.  It is also possible to mount the final root
   filesystem via linuxrc and use the pivot_root syscall.  The initrd is
   mounted and executed via prepare_namespace().

c) using initramfs.  The call to prepare_namespace() must be skipped.
   This means that a binary must do all the work.  Said binary can be stored
   into initramfs either via modifying usr/gen_init_cpio.c or via the new
   initrd format, an cpio archive.  It must be called "/init".  This binary
   is responsible to do all the things prepare_namespace() would do.

   To maintain backwards compatibility, the /init binary will only run if it
   comes via an initramfs cpio archive.  If this is not the case,
   init/main.c:init() will run prepare_namespace() to mount the final root
   and exec one of the predefined init binaries.

Bryan O'Sullivan <bos@serpentine.com>
