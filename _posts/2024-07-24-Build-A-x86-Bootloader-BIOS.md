---
title: 'Build a bootloader for your x86 Operating System: BIOS (Legacy)'
description: >-
  Bootloader is a piece of code that is executed once the system is booted. Let see how we use BIOS to load our bootloader and boot system.
  
author: Cong
date: 2024-07-24 7:00:00 +0800
categories: [Bootloader-x86, BIOS]
tags: [Bootloader, BIOS, x86, Assembly]
image:
  path: assets/img/Legacy_BIOS_boot_process_fixed.png
  alt: BIOS Boot Process.
---

## 1. Fundamental concepts

### 1.1. POST

When a computer is switched on or reset, it runs through a series of diagnostics called POST - **Power-On Self-Test**. It makes sure all hardware components are working properly. And finally locates a bootable device, such as a Floppy Disk, CD-ROM or a Hard disk in the order that the firmware is configured to.

POST routines are part of a computer's **pre-boot sequence**. If they complete successfully, the **bootstrap loader** code is invoked to load an OS.

In IBM PIC compatible computers, the main duties of POST are handled by the **BIOS/UEFI**.

The principal duties of the main BIOS during POST include:

- Verify CPU registers.
- Verify the integrity of the BIOS code itself.
- Verify some basic components like DMA, timer, interrupt controller.
- Initialize, size, and verify system main memory (check corruptions).
- Initialize BIOS.
- Pass control to other specialized extension BIOSes (if installed).
- **Identify, organize, and select which devices are available for booting**.

In later BIOS versions, POST will also:

- Initialize chipset.
- Discover, initialize, and catalog all system buses and devices.
- Provide a User Interface for system's configuration.
- Construct whatever system environment is required by the target OS.

Once BIOS finds the boot sector it loads the image in memory and execute it. If a valid boot sector is not found, BIOS check for next drive in boot sequence until it find valid boot sector. If BIOS fails to get valid boot sector, generally it stops the execution and gives an error message "Disk boot failure".

> **It is boot sectors responsibility to load the operating system in memory and execute it.**
{: .prompt-tip }

### 1.2. BIOS

BIOS (Basic Input/Output System) was created to offer generalized low-level services to early PC system programmers. The basic aims:

- To hide (as much as possible) variations in PC models and hardware from the OS and applications. -> Try to **abstract**, **de-couple** upper layers like OS and apps from hardwares, PC models. So OS and apps development be more easier (Because the BIOS services handled most of the hardware level interface).

#### 1.2.1. BIOS Services

These BIOS services are still used (especially during boot-up), and are often named `BIOS functions`. In real mode, they can easily accessed through **Software Interrupts**, using Assembly language.

Generally, you can access a BIOS function by setting the `AH` CPU register (or `AX` or `EAX`) to a particular value, and then do an `INT` opcode. The value in `AH`, combined with the particular interrupt number selected requests a specific BIOS function. (Another CPU registers hold any **arguments** to the function, and often the return values.)

For example: `INT 0x13` with `AH=0` is a BIOS function that resets hard disks or floppy disk.

Some common BIOS services:

- `INT 0x10, AH = 1` -- set the cursor.
- `INT 0x10, AH = 3` -- cursor position.
- `INT 0x10, AH = 0xE` -- display char.
- `INT 0x10, AH = 0x13` -- display string.

ATA using BIOS (Disk access using BIOS `INT 0x13`)

- `INT 0x13, AH = 0x2` -- read floppy/hard disk in CHS mode.
- `INT 0x13, AH = 0x3` -- write floppy/hard disk in CHS mode.
- `INT 0x13, AH = 0x42` -- read hard disk in LBA mode.
- `INT 0x13, AH = 0x43` -- write hard disk in LBA mode.

Memory detection

- `INT 0x12` -- get low memory size.
- `INT 0x15, EAX = 0xE820` -- get complete memory map.
- `INT 0x15, AX = 0xE801` -- get contiguous memory size.

> Each BIOS function has a specific set of "result" registers. For errors handling, almost always set the carry flag (`JC`), sometimes return `AH = 0x86` (Unsupported), `AH = 0x80` (Invalid Command).
{: .prompt-info }
> In Protected Mode and Long Mode, almost BIOS services become unavailable.
{: .prompt-info }

#### 1.2.2. BIOS (Legacy) and UEFI

In this topic, we'll talk about BIOS, but let see some main differences between BIOS and UEFI.

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

It seems UEFI more powerful than BIOS, but BIOS is still widely used because offering simplicity and compatibility with older hardware and OSes.

### 1.3. Master Boot Record

A master boot record (MBR) is a type of **boot sector** in the first few blocks of partitioned computer mass storage devices (hard disk, etc.). The MBR contains:

- Information on **how the disc's sectors are divided into partitions**, each partition notionally containing a file system.
- **Executable code to function as a loader** for the installed OS - usually by passing control over to the loader's second stage.

> The MBR code is usually referred to as a bootloader.
{: .prompt-info }

#### 1.3.1. Why is BIOS supported partition limited to 2TB?

The organization of the partition table in MBR limits the maximum addressable storage space of a partitioned disk to 2 TiB (2^32 * 512 bytes). Approaches to slightly raise this limit utilizing 32-bit arithmetic or 4096-byte sectors are not officially supported.
Therefore, the MBR-based partition scheme is in the process of being superseded by the **GUID Partition Table (GPT)** scheme in new computers. A GPT can co-exist with an MBR in order to provide some limited form of backward compatibility for older systems.

## 2. Boot Process

![Bios Boot Process](assets/img/Legacy_BIOS_boot_process_fixed.png)

### 2.1. System startup

#### 2.1.1. Where is BIOS firmware stored?

Originally, BIOS firmware was stored in a ROM chip on the PC motherboard. In later computer systems the BIOS contents are stored on Flash Memory (or NVRAM) so it can be rewritten without removing the chip from mother board.

#### 2.1.2. How does BIOS start running?

Early Intel processors started at physical address 000FFFF0h. Systems with later processors provide logic to start running the BIOS from the system ROM.

- `cold boot`: system has been powered up or the reset button was pressed.
- `warm boot`: Ctrl + Alt + Delete was pressed.

If the system has a cool boot, the full POST is run. Otherwise, a special flag value stored in **Nonvolatile BIOS memory** tested by the BIOS allows bypass of the lengthy POST and memory detection.

### 2.2. Boot process

After POST, the BIOS calls `INT 0x19` to start booting processing. When `INT 0x19` is called, the BIOS attempts to locate the `boot loader` software (Master Boot Record) on a `boot device` such as a hard disk, a floppy disk, CD or DVD. It loads and executes the first boot software it finds, giving the control to it.

The BIOS uses the boot devices set in the **Nonvolatile BIOS memory**. It checks each device in order to see if it is bootable by **attempting to load the first sector (boot sector)**. If the sector cannot be read, the BIOS proceeds to the next device. If the sector is read successfully, BIOS checks for the boot sector signature `0x55AA` in the end of sector.

> The boot signature number `0x55AA` is also called `magic number`. It's in the last 2 bytes of boot sector.
{: .prompt-info }

When a bootable device is found, the BIOS transfer control to the loaded sector.
