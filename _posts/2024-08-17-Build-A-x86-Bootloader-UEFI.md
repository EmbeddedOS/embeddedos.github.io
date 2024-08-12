---
title: 'Build your own bootloader for x86 OS: UEFI'
description: >-
  Let's discover what is UEFI and how to use it to load our custom x86 bootloader and boot system.
  
author: Cong
date: 2024-08-17 7:00:00 +0800
categories: [Bootloader-x86, UEFI]
tags: [Bootloader, UEFI, x86, QEMU, Assembly]
image:
  path: assets/img/UEFI_boot_process.png
  alt: UEFI Boot Process.
---

## 1. UEFI basic concepts

**Unified Extensible Firmware Interface** (UEFI) is a specification that defines the architecture of the platform firmware used for booting the computer hardware and its interface for interaction with the operating system.

UEFI replaces the BIOS which was present in the boot ROM, although it can provide **backwards compatibility** with the BIOS using CSM booting.

UEFI is independent of platform and programming language, but C is used for the reference implementation **TianoCore EDKII**.

### 1.1. Advantages

The interface defined by the EFI specification includes data tables that contain:

- Platform information.
- Boot and runtime services to the Bootloader and OS.

Several technical advantages over a BIOS:

- Ability to boot a disk containing large partitions (over 2TB) with a GUID Partition Table (GPT).
- Flexible pre-OS environment: networking, GUI, multi-language, etc.
- 32-bit or 64-bit pre-OS environment.
- **C**.
- **Python**.
- Modular design.
- Backward and forward compatibility.
