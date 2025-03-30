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

- guest - your running system that is emulated by QEMU.
- host - your system that you run QEMU on.

## 1. Device Emulation

- Device frontend -- the way device exposed to the guest.
- Device buses -- Most devices will exist on a BUS of some sort.
- Device backend - How the data from emulated device will be processed by QEMU.

### 1.1. Device frontend

A device front end is how a device is presented to the guest. All devices can be specified with the `--device` option.

Check list all devices supported by the machine: `qemu-system-aarch64 -M virt --device help`. For example:

```bash
$ qemu-system-aarch64 -M virt --device help
Controller/Bridge/Hub devices:
name "pci-bridge", bus PCI, desc "Standard PCI Bridge"
name "pci-bridge-seat", bus PCI, desc "Standard PCI Bridge (multiseat)"
...

USB devices:
name "ich9-usb-ehci1", bus PCI
name "ich9-usb-ehci2", bus PCI
mmm

Storage devices:

name "nvme", bus PCI, desc "Non-Volatile Memory Express"
name "pvscsi", bus PCI
name "scsi-cd", bus SCSI, desc "virtual SCSI CD-ROM"
name "scsi-disk", bus SCSI, desc "virtual SCSI disk or CD-ROM (legacy)"
name "sd-card", bus sd-bus
name "sdhci-pci", bus PCI
name "usb-bot", bus usb-bus
name "usb-storage", bus usb-bus
```

To check specific device's additional configuration options: `qemu-system-aarch64 -M virt --device foo,help`. For example, a `sd-card` device:

```bash
$ qemu-system-aarch64 -M virt --device sd-card,help
sd-card options:
  drive=<str>            - Node name or ID of a block device to use as a backend
  spi=<bool>
  spec_version=<uint8>
```

### 1.2. Device buses

Most devices will exist on a BUS of some sort. Some buses will be created automatically for each machine.

Normally, the BUS a device is attached to can be inferred, for example, PCI devices are generally automatically allocated to the next free address of first bus found. But you can also specify address (`addr=N`), bus (`bus=ID`) yourself.

Some devices, for example, a PCI SCSI host controller, will add an additional bus to the system that other devices can be attached to. For example, the `bar` device will be attached to the first `foo` bus (`foo.0`) at address 1. The `foo` device which provides that bus itself is attached to the PCU bus `pci.0`,

```bash
–device foo,bus=pci.0,addr=0,id=foo –device bar,bus=foo.0,addr=1,id=baz
```

### 1.3. Device backend

For serial device wll be backed by a `--chardev`, you can redirect data output to a file or a socket or some other system.
Storage devices are handled by `--blockdev` which specify how blocks are handled, for example, being stored in a qcow2 file or accessing a raw host disk partition.

### 1.4. Device use

[Doc](https://github.com/qemu/qemu/blob/master/docs/qdev-device-use.txt)

#### 1.4.1. Block devices

A QEMU block device (drive) has a host and a guest part.

To define the host part, we use `-drive` anf guest device(s) with `-device`.

```bash
   -drive if=none,id=DRIVE-ID,HOST-OPTS...
   -device DEVNAME,drive=DRIVE-ID,DEV-OPTS...
```

- Host options (`HOST-OPTS`): file, format, snapshot, cache, aio, readonly, rerror, werror, etc.
- Device options (`DEV-OPTS`): serial, etc.

The `-device` argument depends on each type of drive:

- `if=ide`: `-device DEVNAME,drive=DRIVE-ID,bus=IDE-BUS,unit=UNIT`
- `if=virtio`: `-device virtio-blk-pci,drive=DRIVE-ID,class=C,vectors=V,ioeventfd=IOEVENTFD`

## 2. QEMU options

### `bios` option

Using this option means you are trying to emulate the earliest code in your architecture (normally ROM code). The given image normally is in binary format which will get loaded into some flash or ROM memory where the CPU start executing. Because you're try to emulate ROM code, QEMU will try to act like the CPU, pass the first control to you.

In case of u-boot, QEMU load it into ROM (flash memory), the U-Boot after that relocate itself to RAM and jump to it.

### `kernel` option

Refer to emulating kernel code (mostly Linux kernel). Kernel running means, the bootloader already there, so the QEMU try to load various kernel format like the way bootloader do.

For example, in case ARM virt machine, QEMU will try to load if kernel is Linux kernel, if the kernel is other formats like ELF, PE, etc. QEMU will still try to load it.

If the firmware was started with `bios` option, the `kernel` option might be ignored.

### `initrd` option

This option normally comes along with `kernel` option, because you are trying to emulate kernel, QEMU will act like a bootloader, and the bootloader also can load both initrd and kernel. It works exactly the way UBoot do: Load `initrd` into RAM and then pass info into kernel via parameters.

### `device` option

#### Generic Loader

The `loader` device allows the user to load multiple images or values into QEMU at startup. `-device loader,addr=<addr>,data=<data>,data-len=<data-len>[,data-be=<data-be>][,cpu-num=<cpu-num>]`
An example of loading value 0x8000000e to address 0xfd1a0104 is: `-device loader,addr=0xfd1a0104,data=0x8000000e,data-len=4`
For example testing TF-A with multiple separated BLx firmware and loaded in different addresses.

```bash
qemu-system-aarch64 -M virt -cpu cortex-a57 -nographic \
    -bios build/qemu/release/bl1.bin \
    -device loader,file=build/qemu/release/bl2.bin,cpu-num=0,addr=0x40010000 \
    -device loader,file=build/qemu/release/bl31.bin,cpu-num=0,addr=0x40000000 \
    -device loader,file=u-boot.bin,cpu-num=0,addr=0x42000000
```

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

## 4. Block devices

- [UBoot with QEMU](https://docs.u-boot.org/en/latest/board/emulation/blkdev.html)

- Emulate MMC:

```text
-device sdhci-pci,sd-spec-version=3 \
-drive if=none,file=disk.img,format=raw,id=MMC1 \
-device sd-card,drive=MMC1
```

- Emulate USB block device:

```text
-device qemu-xhci \
-drive if=none,file=disk.img,format=raw,id=USB1 \
-device usb-storage,drive=USB1
```

- Emulate virtio:

```text
-drive if=none,file=disk.img,format=raw,id=VIRTIO1 \
-device virtio-blk,drive=VIRTIO1
```

## Debugging
