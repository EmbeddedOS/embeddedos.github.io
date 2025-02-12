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

- Disk file systems: based on disk storage media. Some disk file system formats: FAT (FAT12, FAT16, FAT32), exFAT, NTFS, ext2, ext3, ext4, XFS.
- Flash file systems: A file system that is designed for storing files on flash memory.
- Network file systems: A file system that acts as a client for a remote file access protocol, providing access to files on a server.
- Special file systems: Some file system expose elements of the OS as files so they can be acted on via the filesystem API (open()/read()/write()/close()/etc.). For example, these are common in Unix-like OSes:
  - `sysfs` expose special files that can be used to query and configure Linux Kernel information.
  - `udev`, `devfs` expose I/O devices or pseudo-devices as special files.
  - `proc` expose process information as special files.

### 1.4. Virtual file system

A virtual file system (VFS) is an abstraction layer within an OS that provides a unified interface for accessing different types of file systems. By that we can hiding the details of how each file system storages data.

```text
 _____________________________________________________________
| User                                                        |
|       |app1|              |app2|              |app3|        |
|_________||__________________||__________________||__________|
     _____||__________________||__________________||__________
    |unified interface: read(), write(), open(), close(), etc.|
          /\                  /\                  /\
 _________||__________________||__________________||__________
|        _||__________________||__________________||_         |
|       |_  ___________Virtual_Filesystem_________  _|        |
|         ||                  ||                  ||          |
|Kernel   \/                  \/                  \/          |
|    |Ext4 driver|       |NFS module|     |Proc subsystem|    |
|_________/\__________________/\______________________________|
          ||                  ||
 _________||________   _______||_________
|Hw       ||        | |Cloud  ||         |
|         \/        | |       \/         |
|   Ext4 filesystem | |   Network fs     |
|___________________| |__________________|
```

### 1.5. File system implementation

Unix-like OSes create a VFS which makes all the files on all the devices appear to exist in a single hierarchy. That means, in those systems, there is one **root directory**, and every file existing on the system is located under it somewhere.

To access the files on another device, the OS must first be informed where in the directory tree those files should appear. This process is called **mounting** a file system. For example, to access the files on a CD-ROM, one must tell the OS *Take the file system from this CD-ROM and make it appear under such-and-such directory*. The directory given to the OS is called **mount point**, it might, for example, the `/media` directory exists as a mount point for removable media such as CDs, DVDs, USB drives. Or `/mnt` for temporary mounts initiated by the user. They may be empty, or it may contain subdirectories for mounting individual devices.

## 2. Root file system

In previous sections, we mentioned the **root directory** that is first or top-most directory in a hierarchy. It can be likened to the trunk of a tree, as the starting point where point where all branches originate from. The **root file system** is the file system contained on the same **disk partition** on which the root directory is located.

In Unix-like systems, the root directory is abstracted and denoted by the `/` (slash) sign. All filesystem entries, including mounted file systems are **branches** of this root.

> On many Unixes, there is also a directory named `/root` that is the home directory of the root user.
{: .prompt-info }

### 2.1. Root file system structure

A rootfs structure refers to the hierarchical organization of directories. In Unix-like systems, there are some entries are required in `/`.

```text
| Directory | Description                                                      |
|-----------|------------------------------------------------------------------|
| `/bin`    | Essential command binaries that can be used by both admins and   |
|           | users. NO subdirectories in `/bin`. Some commands, for example:  |
|           | `cat`, `cp`, `ls`, etc.                                          |
|-----------|------------------------------------------------------------------|
| `/boot`   | Contains everything required for the boot process: kernel,       |
|           | initrd, device tree table binary, etc. These data are used before|
|           | the kernel begins executing user-mode programs.                  |
|           | Kernel must be located in either `/` or `/boot`.                 |
|-----------|------------------------------------------------------------------|
| `/dev`    | Is the location of special or device files.                      |
|-----------|------------------------------------------------------------------|
| `/etc`    | Hierarchy contains configuration files. A `configuration file` is|
|           | local file used to control the operation of a program; it must   |
|           | be static and cannot be an executable binary.                    |
|           | It is recommended that files be stored in subdirectories of      |
|           | `/etc` rather than directly in `/etc`.                           |
|           | Configuration files, for example: `/etc/fstab`: static info about|
|           | filesystems.                                                     |
|-----------|------------------------------------------------------------------|
| `/home`   | User home directories (optional). `/home` is a fairly standard   |
|           | concept, but it is clearly a site-specific filesystem. So, no    |
|           | program should assume any specific location fo a home directory. |
|-----------|------------------------------------------------------------------|
| `/lib`    | Essential shared libraries and kernel modules.                   |
|           | Contains those shared library images needed to boot the system   |
|           | and run the commands in rootfs, ie. By binaries in `/bin` and    |
|           | `/sbin`.                                                         |
|           | `/lib/modules` directory contains loadable kernel module         |
|           | (optional)|.                                                     |
|-----------|------------------------------------------------------------------|
| `/lib<..>`| Alternate format essential shared libraries (optional).          |
|           | There may be one or more variants of the `/lib` directory on     |
|           | systems which support more than one binary format requiring      |
|           | separate libraries.                                              |
|-----------|------------------------------------------------------------------|
| `/media`  | Mount point for removable media                                  |
|           | This directory contains subdirectories which are used as mount   |
|           | points for removable media such as floppy disks, cdrom, etc. For |
|           | example:                                                         |
|           | `/media/floppy` Floppy drive (optional).                         |
|           | `/media/cdrom` CD-ROm drive.                                     |
|           | In case more than one device exists for mounting a certain type  |
|           | of media, mount directories can be created by appending a digit. |
|           | ie. `/media/floppy0`, `/media/floppy1`.                          |
|-----------|------------------------------------------------------------------|
| `/mnt`    | Mount point for a temporary mounted filesystem.                  |
|           | The admin may temporarily mount a file system as needed.         |
|-----------|------------------------------------------------------------------|
| `/opt`    | Is reserved for the installation of add-on application software  |
|           | packages. A package to be installed in `/opt` must locate its    |
|           | static files in a separate `/opt/<package>` or `/opt/<provider>` |
|           | directory tree.                                                  |
|-----------|------------------------------------------------------------------|
| `/root`   | Home directory for the root user (optional).                     |
|-----------|------------------------------------------------------------------|
| `/run`    | Run-time variable data.                                          |
|           | This directory contains system information data describing the   |
|           | since it was booted.                                             |
|           | Process identifies (PID) files must be placed in `/run`. The     |
|           | naming convention for PID files is `<program>.pid`, ie.          |
|           | `/run/crond.pid`.                                                |
|-----------|------------------------------------------------------------------|
| `/sbin`   | System binaries. Utilities used for system admin are stored in   |
|           | `/sbin`, `/usr/bin` and `/usr/local/sbin`.                       |
|           | There must be no subdirectories in `sbin`.                       |
|           | For example, commands: `shutdown` - command to bring system down,|
|           | `ifconfig` - configure a network interface.                      |
|-----------|------------------------------------------------------------------|
| `/srv`    | Contains site-specific data which is served by this system.      |
|           | The users may find the location of the data files for a          |
|           | particular service, and so that services which require a single  |
|           | tree for ro-data, writable-data and scripts can be reasonable    |
|           | placed.                                                          |
|-----------|------------------------------------------------------------------|
| `/tmp`    | Temporary files.                                                 |
|-----------|------------------------------------------------------------------|
| `/usr`    | This Hierarchy is a second major section of the filesystem.      |
|           | - `/usr/bin` - Most user commands.                               |
|           | - `/usr/include` - standard include files. This is where all the |
|           |   system's general-use include files for the C programming should|
|           |   be placed.                                                     |
|           | - `/usr/lib` - Libraries for programming and packages.           |
|           | - `/usr/libexec` - Binaries run by other programs.               |
|           | - `/usr/lib<qual>` - Alternative format libraries (optional).    |
|           | - `/usr/local` - Local hierarchy is for ise by the system admin  |
|           |   when installing software locally.                              |
|           | - `/usr/local/bin` - Local binaries.                             |
|           | - `/usr/local/etc` - Local system configuration file.            |
|           | - `/usr/local/games` - Local game binaries.                      |
|           | - `/usr/local/include` - Local C header files.                   |
|           | - `/usr/local/sbin` - Local system binaries.                     |
|           | - `/usr/local/share` - Local architecture-independent hierarchy. |
|           | - `/usr/sbin/` - Non essential standard system binaries.         |
|           | - `/usr/share/` - This hierarchy is for all read-only arch       |
|           |   independent data files.                                        |
|           | - `/usr/share/man` - Manual pages.                               |
|           | - `/usr/share/color` - Color management information.             |
|-----------|------------------------------------------------------------------|
| `/var`    | Contains variable data files. This includes directories and      |
|           | files, administrative and logging data, and transient and tmp    |
|           | files.                                                           |
|-----------|------------------------------------------------------------------|
```

For Linux systems, there are more requirements and recommendations.

```text
| Directory | Description                                                      |
|-----------|------------------------------------------------------------------|
| `/dev`    | These devices must exist:                                        |
|           | `/dev/null` - All data written to this device is discarded.      |
|           | `/dev/zero` - All data written to this device is discarded. A    |
|           | read from this device will return as many bytes containing the   |
|           | as was requested.                                                |
|           | `/dev/tty` - This device is a synonym for the controlling        |
|           | terminal of a process. Once this device is opened, all reads and |
|           | writes will behave as if the actual controlling terminal device  |
|           | had been opened.                                                 |
|-----------|------------------------------------------------------------------|
| `/proc`   | Is the standard Linux method for handling process and system     |
|           | information.                                                     |
|-----------|------------------------------------------------------------------|
| `/sys`    | Is the location where information about devices, drivers, and    |
|           | some kernel features is exposed. Its underlying structure is     |
|           | determined by the particular Linux kernel being used at the      |
|           | moment.                                                          |
|-----------|------------------------------------------------------------------|
```

### 2.2. Build a root file system

Creating the rootfs involves selecting files necessary for the system to run. A rootfs must contain everything needed to support a full Linux system. To be able to do this, the disk must include the minimum requirements for a Linux system:

- The basic file system structure.
- Minimum set of directories: `/dev`, `/proc`, `/bin`, `/etc`, `/lib`, `/usr`, `/tmp`.
- Basic set of utilities: `sh`, `cp`, `ls`, `mv`, etc.
- Minimum set of config files: `rc`, `inittab`, `fstab`, etc.
- Devices: `/dev/hd*`, `/dev/tty*`, `/dev/fd0`, etc.
- Runtime libraries to provide basic functions used by utilities.

#### 2.2.1. Creating the filesystem

Building such a rootfs require a spare device that is large enough to hold all the files. There are several choices:

- Use a *ramdisk*, the memory is used to simulate a disk drive.
- You have unused hard disk partition that is large enough.
- Use a loopback device, which allows a disk file to be treated as a device.

##### 2.2.1.1. Using the RAM disk block device with Linux

The RAM disk driver is a way to use main system memory as a block device. It is required for `initrd`, and initial file system used if you need to load modules in order to access the rootfs, I have a full blog about the `initrd` here, take a look [initrd.](/posts/understand-initrd/). It can also be used for a temporary filesystem, since the contents are erased on reboot.

Make sure we have the ramdisk device node first:

```bash
ls /dev/ram*
```

If not, create a device node with `mknod`, the RAM disk is block device with major number 1 and minor start from 0. So to create a new device node to expose to user space. We run:

```bash
mknod -m 660 /dev/ram0 b 1 0
```

> Check all the [Linux device list here](https://www.kernel.org/doc/html/latest/admin-guide/devices.html)
{: .prompt-info }

Decide on the RAM disk size that you want, for example, 20MB in this case:

```bash
dd if=/dev/zero of=/dev/ram0 bs=1k count=20480
```

Make a filesystem on it, for example, ext4 file system:

```bash
mkfs.ext4 -vm0 /dev/ram0 20480
```

> `mkfs.ext4` command is equal to `mke2fs -t ext4`. The `-m 0` option here to prevent the command reserving space for root, and hence provides more usable space on the disk.
{: .prompt-info }

Mount it to a temporary mount point, let's say `/mnt/rootfs`, populate it and unmount it again.

```bash
mount /dev/ram0 /mnt/rootfs

# Populating the rootfs ...

umount /mnt/rootfs
```

About how to populate the rootfs, we'll talk more detail in the **Populating the file system** section. After that, you might want to put the RAM disk image onto an drive or a filesystem image.

```bash
# Writing to a file system image.
dd if=/dev/ram0 bs=1k count=20480 of=/tmp/rootfs.img

# Or compressed the file system image (For initrd image).
dd if=/dev/ram0 bs=1k count=20480 | gzip -v9 > /tmp/rootfs.img.gz

# Put onto other drives if any, for example a floppy.
dd if=/tmp/rootfs.img of=/dev/fd0 bs=1k
```

##### 2.2.1.2. Using a loopback device

A loop device is a pseudo-device that makes a computer file accessible as a block device. To make a loop device, there are several choices:

- Associating loop device with an exist file system image.
  1. Create a raw file system image: `dd if=/dev/zero of=/tmp/rootfs.img bs=1k count=20480`
  2. Format file system: `mkfs.ext4 -vm0 /tmp/rootfs.img 20480`
  3. Associate with a loop device: `losetup /dev/loop0 /tmp/rootfs.img`
  4. Mount the loop device to a temporary mount point: `mount /dev/loop0 /mnt/rootfs`
  5. Populate the rootfs and umount.
  6. Delete the loopback device: `losetup -d /dev/loop0`
- Create a loop device node, work on that, and writing the result onto a new file system image:
  1. Make a loop device node: `mknod /dev/loop0 b 7 0`
  2. Decide the size for the image: `dd if=/dev/zero of=/dev/loop0 bs=1k count=20480`
  3. Format the image: `mkfs.ext4 -vm0 /dev/loop0 20480`
  4. Mount the loop device to a temporary mount point: `mount /dev/loop0 /mnt/rootfs`
  5. Populate the rootfs and umount.
  6. Writing into a file system image: `dd if=/dev/loop0 bs=1k of=/tmp/rootfs.img`
  7. Delete the loopback device: `losetup -d /dev/loop0`
- Using `mount` utility to write directly to a file system, passing the create a loop device node step.
  1. Create a raw file system image: `dd if=/dev/zero of=/tmp/rootfs.img bs=1k count=20480`
  2. Format file system: `mkfs.ext4 -vm0 /tmp/rootfs.img 20480`
  3. Mount this image to a temporary mount point: `mount -o loop /tmp/rootfs.img /mnt/rootfs`
  4. Populate the rootfs and umount.

> Check the list of mounted filesystems and information in `/proc/mounts`.
{: .prompt-info }

#### 2.2.2. Populating the file system

After mounting the device into a temporary mount point, let's say `/mnt/rootfs`, now is time to perform populating. The rootfs require a minimum set of basic file system structure. So we make a basic ones:

```bash
mkdir -p /mnt/rootfs/{bin,sbin,etc,proc,sys,usr/{bin,sbin},dev,lib,var/{log,run}}
```

##### 2.2.2.1. /dev directory

This directory contains special that can not be created in a normal way. To create device special files uses `mknod` command instead.

Create basic device nodes:

```bash
# Device physical memory.
mknod -m 660 /mnt/rootfs/dev/mem c 1 1

# Create virtual consoles.
mknod -m 660 /mnt/rootfs/dev/tty2 c 4 2
mknod -m 660 /mnt/rootfs/dev/tty3 c 4 3
mknod -m 660 /mnt/rootfs/dev/tty4 c 4 4

# Create Null device.
mknod -m 660 /mnt/rootfs/dev/null c 1 3
mknod -m 660 /mnt/rootfs/dev/zero c 1 5
```

> Check all the [Linux device list here](https://www.kernel.org/doc/html/latest/admin-guide/devices.html)
{: .prompt-info }

Another way to do that is copying devices files from your existing hard disk directory.

```bash
cp -dpR /dev/fd[01]* /mnt/rootfs/dev
cp -dpR /dev/tty[0-6] /mnt/rootfs/dev
```

##### 2.2.2.2. /etc directory

This should contain depends on what programs you intent to run. On most disk-based rootfs, we have three sets of files:

- MUST HAVE files for boot/root system:
  - `rc.d/*` - system startup and run level change scripts. The standard location may depends on your `init` program.
  - `fstab` - list of file systems to be mounted.
  - `inittab` - parameters for the `init` process.
- SHOULD HAVE files for boot/root system:
  - `passwd` - list of users, home directories, etc. Should contain at least `root` user.
  - `group` - user groups.
  - `shadow` - passwords of users.
- All the rest is nice to have.

##### 2.2.2.3. /bin and /sbin directories

The `/bin` directory is a convenient place for extra utilities you need to perform basic operations, utilities such as `lv`, `mv`, `cat` and `dd`. You simply copy these utilities from outside.

To get the utilities, a simple way to using common tools, `busybox` is one of the best choice for getting these utilities. Go to the section **Common tools: Busybox** for more detail.

##### 2.2.2.4. /lib directory

The `/lib` and `/lib<arch>` you place necessary shared libraries and loaders. Otherwise some programs or system cannot boot due to lack of shared libraries. Nearly every program requires at least the `libc` library.

In `/lib` you must also include a loader for the libraries. The loader will be either `ld.so` (for A.OUT libraries, that is not common now) or `ld-linux.so` (ELF libraries). You can check shared library dependencies with `ldd` command, that also shows the needed loader. For example, the `libc.so` requires ELF loader and also the `vdso` library:

```bash
$ ldd /lib/x86_64-linux-gnu/libc.so.6
    /lib64/ld-linux-x86-64.so.2 (0x00007f88f33a1000)
    linux-vdso.so.1 (0x00007fff16943000)
```

Some chip vendors are providing compiling toolchains to compile programs that run on their architecture. The toolchain might include libraries and library loaders. For example, if you are building the rootfs for Aarch64 platform, ARM provide libraries in their GNU toolchains, let's say `aarch64-none-linux-gnu` 14.2. You can take the libraries and their loader onto your rootfs:

```bash
cp <toolchain_path>/aarch64-none-linux-gnu/libc/lib64/libc.so.6 lib64/
cp <toolchain_path>/arch64-none-linux-gnu/libc/lib/ld-linux-aarch64.so.1 lib/
```

Kernel modules are placed in `lib/modules` folder, if you include them, remember to to include `insmod`, `rmmod`, and `lsmod` commands also. If you want to load these modules automatically, you might also include `modprobe`, `depmod` and `swapout`. Remember to check shared library dependencies of these and place them onto rootfs also.

##### 2.2.2.6. init program

## 3. Common tools

### 3.1. Busybox

Busybox is a software suite that provides several Unix utilities in a **single executable**. It also includes a complete bootstrapping toolchain (ie. `init`). The tricky part is that when you execute a command, it seems it work separately, but in fact, all of these commands are linked into the only one executable `busybox` binary. For example, you run `ls`, it acts like `/bin/ls`, but actually is linked (hard or symbolic links) to the `/bin/busybox`. Busybox would see that its `name` is `ls` and act like the `ls` program. You can also run the command directly with the `busybox` binary by adding the command as an argument, similar thing that we do with sudo. For example, `busybox ls`.

By providing everything, but in only one binary, this can keeps the user-land tools at a minimum. So every time, memory space is a concern, `busybox` is one of the best choices.

> `initrd` normally using `busybox` to provide early-userspace tools, and also the `/init` program to boot up the system.
{: .prompt-info }

#### 3.1.1. Populating a rootfs with Busybox

Building a busybox image is quite similar to building the kernel:

```bash

```

<https://tldp.org/HOWTO/Bootdisk-HOWTO/buildroot.html>

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
```
