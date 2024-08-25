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

## 1. UEFI Basics

UEFI is a specification for x86, x86-64, ARM and Itanium platforms that defines a **software interface** between the OS and the platform firmware/BIOS.

### 1.1. UEFI vs legacy BIOS

A common misconception is that UEFI and BIOS are mutual exclusion. In reality, both legacy motherboards and UEFI-based motherboards both include BIOS ROMs. The differences are in where they find the bootloader/OS, how they prepare the system before executing it, and what convenience functions they provide.

```text
|                |             Legacy BIOS                  |                    UEFI                    |
|----------------|------------------------------------------|--------------------------------------------|
|Platform        | Performs all the usual platform init(    | Perform same steps like BIOS but it also   |
|Initialization  | memory control config, PCI bus config,   | enables the A20 gate and Protected Mode    |
|                | BAR mapping, Graphic cards, etc.) and    | (for i386 processors) or Long Mode (x64    |
|                | then to drop to real mode env. The boot- | processors).                               |
|                | loader must enable A20, config GDT, IDT, |                                            |
|                | switch to protected mode, config paging  |                                            |
|                | and switch to long mode (x86-64 CPUs).   |                                            |
|----------------|------------------------------------------|--------------------------------------------|
| Boot Mechanism | BIOS loads a 512 byte flat binary blob   | UEFI fw loads an arbitrary sized UEFI      |
|                | from the MBR of the boot device into     | application (e relocatable PE executable   |
|                | memory at physical address 0x7C00 and    | file) from a FAT partition on a GPT or MBR |
|                | jumps to it. The Bootloader CAN'T return | partitioned boot device to some address    |
|                | back to BIOS.                            | selected at run-time. Then it calls that   |
|                |                                          | that application's main entry point.       |
|                |                                          | The application can continue booting or    |
|                |                                          | return control to the UEFI, which continue |
|                |                                          | searching for another boot-device or bring |
|                |                                          | up a diagnostic menu.                      |
|----------------|------------------------------------------|--------------------------------------------|
| System         | A bootloader itself scans memory for     | UEFI fw calls a UEFI application's entry   |
| Discovery      | structures like EBDA, SMBIOS, and ACPI   | point function, it passes a "System Table" |
|                | tables. It uses PIO to talk to the root  | structure, which contains pointers to all  |
|                | PCI controller and scan the PCI bus.     | of the system's ACPI tables, memory map,   |
|                | Redundant tables may be present in       | and other information relevant to an OS.   |
|                | memory and the boot-loader can choose    |                                            |
|                | which to use.                            |                                            |
|----------------|------------------------------------------|--------------------------------------------|
| Convenience    | A legacy BIOS hooks a variety of         | UEFI fw establishes many callable functions|
| Functions      | interrupts which a bootloader can trigger| in memory, which are grouped into sets     |
|                | to access system resources like disks and| called "protocols" and discoverable through|
|                | the screen. These interrupts are NOT     | the "System Table".                        |
|                | STANDARDIZED, each interrupt uses a      | The behavior of each function in each      |
|                | different register passing convention.   | "protocol" is defined by specification.    |
|                |                                          | UEFI applications can define their own     |
|                |                                          | "protocols" and persist them in memory for |
|                |                                          | other UEFI applications to use.            |
|                |                                          | Calling functions follow modern calling    |
|                |                                          | convention that supported by many C        |
|                |                                          | compilers.                                 |
|----------------|------------------------------------------|--------------------------------------------|
| Development    | Can be developed in any env that can     | Can be developed in any language that can  |
| Environment    | generate flat binary images: NASM, GCC,  | be compiled and linked into a PE executable|
|                | etc.                                     | file and supports the calling convention   |
|                |                                          | used to access functions established in    |
|                |                                          | memory by the UEFI fw. That means one of   |
|                |                                          | three dev env:                             |
|                |                                          | - EDK2: is large and complex, yet feature  |
|                |                                          |   filled env with its own build system.    |
|                |                                          |   Can compile UEFI applications and even   |
|                |                                          |   UEFI fw to flash to a BIOS ROM.          |
|                |                                          | - GNU-EFI: is a set of libs and headers for|
|                |                                          |   compiling UEFI applications with a       |
|                |                                          |   system's native GCC. Can't not compile   |
|                |                                          |   UEFI fw.                                 |
|                |                                          | - POSIX-UEFI: is very similar to GNU-EFI,  |
|                |                                          |   but it is distributed mainly as a source,|
|                |                                          |   not as a binary lib.                     |
|----------------|------------------------------------------|--------------------------------------------|
```

You should prefer `UEFI` than `BIOS` except:

- You want to do it with education purpose.
- Your targeting system is legacy and UEFI is not available.

### 1.2. UEFI Classes and Booting

PCs are categorized as UEFI class 0, 1, 2, 3.

- A class 0 machine is legacy system with a legacy BIOS.
- A class 1 machine is a UEFI system, but run in (Compatibility Support Module - a specification let UEFI emulate a legacy BIOS) CSM mode, It's only UEFI "within" the BIOS.
- A class 2 machine is a UEFI system that run **UEFI BOOT** (can launch UEFI applications), also support CSM as a option.
- A class 3 is same class 2 but without CSM supporting.

#### 1.2.1. UEFI Booting

UEFI specification define a **boot manager** that checks boot configurations, settings when a computer is powered on, and then decide to executes Bootloader, Kernel, Applications, etc.

The boot configurations is defined by variables stored in NVRAM (where to load kernel, bootloader, size, etc.).

UEFI can detected an OS bootloader (is stored in EFI system partition) automatically by using a standardize file paths: `<EFI_SYSTEM_PARTITION>\EFI\BOOT\BOOT<MACHINE_TYPE_SHORT_NAME>.EFI`. For example, on x64L `\efi\boot\bootx64.efi` and `\efi\boot\bootaa64.efi` on ARM64 architecture.

#### 1.2.2. Secure boot

The UEFI specification defines a protocol known as **Secure Boot**, which can secure the boot process by preventing the loading of UEFI drivers or OS boot loaders that are not **singed** with an acceptable **digital signature**.

When Secured Boot is enabled, it is initially placed in **setup** mode, which allows a public key from known as the **Platform key** (PK) to be written to the firmware. Once the key is written, Secure Boot enters **User mode**, where only UEFI drivers and OS boot-loaders signed with the **Platform Key** can be loaded by the firmware. Additional **Key Exchange Keys** (KEK) can be added to a database stored in memory to allow other certificates to be used, but they must still have a connection to the private portion of the platform key.

Secure Boot can also be placed in **Custom** mode, where additional public keys can be added to the system that do not match the private key.

### 1.3. EFI partition

An EFI system partition (ESP), is a **data storage device partition** that is used computers adhering to the UEFI specification. Accessed by the UEFI firmware when a computer is powered up, it stores UEFI applications and the files these applications need to run, **including OS bootloader**. Supported partition table schemes include MBR and GPT, as well as **El Torito** volumes on optical discs. For use on ESPs, UEFI defines a specific version of the FAT file system, which is maintained as part of the UEFI specification and independently from the original FAT specification, including the FAT32, FAT16, and FAT12 file systems. The ESP also provides space for a boot sector as part of the backward BIOS compatibility.

### 1.4. Services

EFI defines two types of services: **boot services** and **runtime services**. Boot services are available only while the firmware owns the platform (i.e. before the `ExitBootServices()` call), and they include text and graphical consoles on various devices, and bus, block and file services. Runtime services are still accessible while the OS is running; they include services such as date, time and NVRAM access.

- Graphics Output Protocol (GOP) services: The Graphics Output Protocol (GOP) provides runtime services; The OS is permitted to directly write to the frame-buffer provided by GOP during runtime mode.
- UEFI Memory Map Service.
- SMM services.
- ACPI services.
- SMBIOS services.
- DeviceTree Services (For RISC processors).
- Variable services:
  - UEFI variables provide a way to store data, in particular non-volatile data.
  - Some UEFI variables are shared between platform firmware and OSes.
  - Variable namespaces are identified by GUIDs, and variables are key/value pair.
  - For example, UEFI variables can be used to keep crash messages in NVRAM, after a crash for the Operating system to retrieve after a reboot.
- Time services:
  - UEFI provides time services. Time services include support for time zone and daylight saving fields, which allow the hardware real-time clock to be set to local time or RTC.

## 2. Application Development

Beyond loading an OS, UEFI can run **UEFI applications**, which reside as files on the EFI system partition. They can be executed from the UEFI Shell, by the firmware's boot manager, ot by other UEFI applications.

A type of UEFI application is an OS boot loader such as GRUB, rEFInd, Gummiboot, and Windows Boot Manager, which loads some OS files into memory and executes them. Also, an OS bootloader can provide a user interface to allow the selection of another UEFI application to run. Utilities like the UEFI Shell are also UEFI applications.

### 2.1. Download UEFI images

If you choose VirtualBox for virtualization, UEFI is already included, no need to download the image manually. Just enable it.

otherwise for emulation and virtual machines, we need an **OVMF.fd** firmware image. Install on Debian/Ubuntu:

```bash
apt-get install ovmf
```

#### 2.2. Emulation
