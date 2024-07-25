---
title: 'Build a x86-64 bootloader from scratch: BIOS'
description: >-
  Bootloader is a piece of code that is executed once the system is booted.
  
author: Cong
date: 2024-07-24 7:00:00 +0800
categories: [Bootloader, x86]
tags: [Bootloader, BIOS, x86, Assembly]
---

## 1. Booting Process

### 1.1. POST

When a computer is switched on or reset, it runs through a series of diagnostics called POST - **Power-On Self-Test**. It makes sure all hardware components are working properly. And finally locates a bootable device, such as a Floppy Disk, CD-ROM or a Hard disk in the order that the firmware is configured to.

POST routines are part of a computer's **pre-boot sequence**. If they complete successfully, the **bootstrap loader** code is invoked to load an OS.

In IBM PIC compatible computers, the main duties of POST are handled by the **BIOS/UEFI**.

### 1.1.1. POST Purposes

During the POST, the BIOS must integrate multiple competing, changing, and even mutually exclusive standards and initiatives for the matrix of hardware and OSes the PC is expected to support, although at most only simple memory tests and the setup screen are displayed. The principal duties of the main BIOS during POST include:

- Verify CPU registers.
- Verify the integrity of the BIOS code itself.
- Verify some basic components like DMA, timer, interrupt controller.
- Initialize, size, and verify system main memory.
- Initialize BIOS.
- Pass control to other specialized extension BIOSes (if installed).
- Identify, organize, and select which devices are available for booting.

The functions above are served by the POST in all BIOS versions back to the very first. In later BIOS versions, POST will also:

- Initialize chipset.
- Discover, initialize, and catalog all system buses and devices.
- Provide a **User Interface** for system's configuration.
- Construct whatever system environment is required by the target OS.

Once BIOS finds the boot sector it loads the image in memory and execute it. If a valid boot sector is not found, BIOS check for next drive in boot sequence until it find valid boot sector. If BIOS fails to get valid boot sector, generally it stops the execution and gives an error message "Disk boot failure".

It is boot sectors responsibility to load the operating system in memory and execute it.

### 1.2. Master-Boot-Record
