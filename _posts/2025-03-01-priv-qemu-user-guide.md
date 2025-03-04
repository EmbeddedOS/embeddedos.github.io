---
title: "QEMU user guide"
description: >-
  Build custom QEMU, understand useful options.

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [QEMU, user-guide]
tags: [QEMU]
published: false
---

## 1. Device Emulation

- Device frontend -- the way device exposed to the guest.
- Device buses -- Most devices will exist on a BUS of some sort.
- Device backend - How the data from emulated device will be processed by QEMU.

You can specify multiple devices to your guest with multiple `--device` option.

### 3. Device backend

For serial device wll be backed by a `--chardev`, you can redirect data output to a file or a socket or some other system.
Storage devices are handled by `--blockdev` which specify how blocks are handled, for example, being stored in a qcow2 file or accessing a raw host disk partition.

## 2. QEMU options

### `bios` option

Using this option means you are trying to emulate the earliest code in your architecture (normally ROM code). The given image normally is in binary format which will get loaded into some flash or ROM memory where the CPU start executing. Because you're try to emulate ROM code, QEMU will try to act like the CPU, pass the first control to you.

### `kernel` option

Refer to emulating kernel code (mostly Linux kernel). Kernel running means, the bootloader already there, so the QEMU try to load various kernel format like the way bootloader do.

For example, in case ARM virt machine, QEMU will try to load if kernel is Linux kernel, if the kernel is other formats like ELF, PE, etc. QEMU will still try to load it.

If the firmware was started with `bios` option, the `kernel` option might be ignored.

### `initrd` option

This option normally comes along with `kernel` option, because you are trying to emulate kernel, QEMU will act like a bootloader, and the bootloader also can load both initrd and kernel. It works exactly the way UBoot do: Load `initrd` into RAM and then pass info into kernel via parameters.

### `device` option

### `net` option

## 3. Network emulation

By default, QEMU provide a user mode network stack for every guest (If no `-net` option is specified).

```text
guest (10.0.2.15)  <------>  Firewall/DHCP server <-----> Internet
                      |          (10.0.2.2)
                      |
                      ---->  DNS server (10.0.2.3)
                      |
                      ---->  SMB server (10.0.2.4)
```

So to access the internet outside (via your host):

- Configure guest IP: `10.0.2.15`.
- Configure default gateway to: `10.0.2.2`.
- Point DNS to: `10.0.2.3`.

## Debugging
