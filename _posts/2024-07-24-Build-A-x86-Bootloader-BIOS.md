---
title: 'Build a x86 bootloader from scratch with BIOS'
description: >-
  Bootloader is a piece of code that is executed once the system is booted.
  
author: Cong
date: 2024-07-14 7:00:00 +0800
categories: [Bootloader, x86]
tags: [Bootloader, BIOS, x86, Assembly]
---

## 1. Booting Process

### 1.1. POST

When a computer is switched on or reset, it runs through a series of diagnostics called POST - **Power-On Self-Test**. It makes sure all hardware components are working properly. And finally locates a bootable device, such as a Floppy Disk, CD-ROM or a Hard disk in the order that the firmware is configured to.

POST routines are part of a computer's **pre-boot sequence**. If they complete successfully, the **bootstrap loader** code is invoked to load an OS.

In IBM PIC compatible computers, the main duties of POST are handled by the **BIOS/UEFI**.

### 1.2. BIOS
