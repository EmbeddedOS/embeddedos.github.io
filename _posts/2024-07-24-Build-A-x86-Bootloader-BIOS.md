---
title: 'Build a bootloader for your x86 Operating System: BIOS (legacy)'
description: >-
  Bootloader is a piece of code that is executed once the system is booted. Let see how we use BIOS to load our bootloader and boot system.
  
author: Cong
date: 2024-07-24 7:00:00 +0800
categories: [Bootloader-x86, BIOS]
tags: [Bootloader, BIOS, x86, Assembly]
---

## 1. Booting Process

### 1.1. POST

When a computer is switched on or reset, it runs through a series of diagnostics called POST - **Power-On Self-Test**. It makes sure all hardware components are working properly. And finally locates a bootable device, such as a Floppy Disk, CD-ROM or a Hard disk in the order that the firmware is configured to.

POST routines are part of a computer's **pre-boot sequence**. If they complete successfully, the **bootstrap loader** code is invoked to load an OS.

In IBM PIC compatible computers, the main duties of POST are handled by the **BIOS/UEFI**.

The principal duties of the main BIOS during POST include:

- Verify CPU registers.
- Verify the integrity of the BIOS code itself.
- Verify some basic components like DMA, timer, interrupt controller.
- Initialize, size, and verify system main memory.
- Initialize BIOS.
- Pass control to other specialized extension BIOSes (if installed).
- **Identify, organize, and select which devices are available for booting**.

In later BIOS versions, POST will also:

- Initialize chipset.
- Discover, initialize, and catalog all system buses and devices.
- Provide a **User Interface** for system's configuration.
- Construct whatever system environment is required by the target OS.

Once BIOS finds the boot sector it loads the image in memory and execute it. If a valid boot sector is not found, BIOS check for next drive in boot sequence until it find valid boot sector. If BIOS fails to get valid boot sector, generally it stops the execution and gives an error message "Disk boot failure".

**It is boot sectors responsibility to load the operating system in memory and execute it.**

### 1.2. BIOS

### 1.2.1. BIOS (legacy) and UEFI

```text
|            |          BIOS             |             UEFI             |
|------------|---------------------------|------------------------------|
|Release Date| 1981                      | 2002. Intel developed to     |
|            |                           | replace legacy BIOS arch.    |
|------------|---------------------------|------------------------------|
|    User    | Text-based.               | GUI, mouse, keyboard.        |
|  Interface |                           |                              |
|------------|---------------------------|------------------------------|
| Operating  | 16-bit, limited to 1MB of | 32-bit, 64-bit, networking   |
|    Mode    | addressable space.        | booting, remote diagnostics. |
|------------|---------------------------|------------------------------|
| Partition  | MBR (Master Boot Record), | GPT (GUID Partition Table),  |
|  Support   | up to 2TB.                | over 2TB.                    |
|------------|---------------------------|------------------------------|
|  Security  | Basic, no inherent        | Support Secure boot, prevent |
|            | security feature.         | unauthorized OS.             |
|------------|---------------------------|------------------------------|
|Performance | Slower boot times, limited| Faster boot times, optimized |
|            | hardware support          | for modern hardware.         |
|------------|---------------------------|------------------------------|
```

- It seems UEFI more powerful than BIOS, but BIOS is still widely used because offering simplicity and compatibility with older hardware and OSes.
