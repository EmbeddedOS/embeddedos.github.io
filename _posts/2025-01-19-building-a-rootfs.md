---
title: "Building a root filesystem for Linux system"
description: >-
  Understand the rootfs concept, how to build a minimum rootfs for Linux system.

author: Cong
date: 2025-01-19 00:01:00 +0800
categories: [Linux, rootfs]
tags: [rootfs]
---

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
