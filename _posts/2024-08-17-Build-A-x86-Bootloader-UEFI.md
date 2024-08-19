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

### 1.2. Compatibility

#### 1.2.1. Processor compatibility

Standard PC BIOS is limited to a 16 bit processor mode and 1 MB of addressable memory space, resulting from the design based on the IBM 5150 that used a 16-bit Intel 8080 Processor.

In comparison, the processor mode in a UEFI environment can be either 32-bit (IA-32, AAarch32) or 64 bit (x86-64, Itanium, and AArch64). 64 bit UEFI firmware supported **long mode**, which allows application in the pre-boot environment to use 64-bit addressing to get direct access to all of the machine's memory.

UEFI requires the firmware and the OS loader (or kernel) to be **size-matched**; That is, 64-bit UEFI firmware can only load 64-bit OS, and the same applies to 32-bit. After the kernel take control it can change the processor mode if desires.

#### 1.2.2. Disk Device Compatibility

In addition to the standard PC disk partition scheme that uses a Master Boot Record (MBR), UEFI also works with the **GUID Partition Table** partitioning scheme, which is free from many of the limitations of MBR. The MBR limits on number and size of disk partitions are released. GPT allows for a maximum disk and partition size of 8 ZiB (8 * 2 ^ 70).

#### 1.2.3. Linux

Option `CONFIG_EFI_PARTITION` enable supporting for GPT in Linux, this allows Linux to recognize and use GPT disks after the system firmware passes control over the system to Linux.

For reverse compatibility, Linux can use GPT disks in BIOS-based systems for both data storage and booting, as both GRUB 2 and Linux are **GPT-aware**.

### 1.3. Features

#### 1.3.1. Services

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

#### 1.3.2. Applications

Beyond loading an OS, UEFI can run **UEFI applications**, which reside as files on the EFI system partition. They can be executed from the UEFI Shell, by the firmware's boot manager, ot by other UEFI applications.

UEFI applications can be developed and installed independently of the Original Equipment Manufactures (OEMs).

A type of UEFI application is an OS boot loader such as GRUB, rEFInd, Gummiboot, and Windows Boot Manager, which loads some OS files into memory and executes them. Also, an OS bootloader can provide a user interface to allow the selection of another UEFI application to run. Utilities like the UEFI Shell are also UEFI applications.

#### 1.3.3. Protocols

EFI defines protocols as a set of software interfaces used for communication between two binary modules. All EFI drivers must provide services to others via protocols. The EFI protocols are similar to the BIOS interrupt calls.

#### 1.3.4. Device Drivers

In additional to standard ISA Instruction Set Architecture-specific device drivers, EFI provides for **a ISA-independent device driver** stored in Non-Volatile Memory as **EFI byte code** or **EBC**. System firmware has an interpreter for EBC images.

Some architecture-specific (non-EFI Byte Code) EFI drivers for some device types can have interfaces for use by the OS. This allows the OS to rely on EFI for drivers to perform basic graphic and network functions before, and if, Operating-System-Specific drivers are loaded.

In other cases, the EFI driver can be filesystem drivers that allow for booting from other types of disk volumes.

#### 1.3.5. Graphics features

The EFI 1.0 specification defined a UGA (Universal Graphic Adapter) protocol as a way to support graphics features.
The UEFI 2.1 defined a **Human Interface Infrastructure** to manage user input, localized strings, fonts, and forms.

Most early UEFI firmware implementations were console-based. Today many UEFI firmware implementations are GUI-based.

#### 1.3.5. EFI system partition

An EFI system partition (ESP), is a **data storage device partition** that is used computers adhering to the UEFI specification. Accessed by the UEFI firmware when a computer is powered up, it stores UEFI applications and the files these applications need to run, **including OS bootloader**. Supported partition table schemes include MBR and GPT, as well as **El Torito** volumes on optical discs. For use on ESPs, UEFI defines a specific version of the FAT file system, which is maintained as part of the UEFI specification and independently from the original FAT specification, including the FAT32, FAT16, and FAT12 file systems. The ESP also provides space for a boot sector as part of the backward BIOS compatibility.

#### 1.3.6. Booting

##### 1.3.6.1. UEFI booting

Unlike the legacy PC BIOS, UEFI does not rely on boot sectors, defining instead a boot manager as part of the UEFI specification. When a computer is powered on, the boot manager checks the boot configuration and, based on its settings, then executes the specified OS boot-loader or OS kernel. The boot configuration is defined by variables stored in NVRAM, including variables that indicate the file system paths to OS loaders or OS kernels.

OS boot loader can be automatically detected by UEFI, which enables easy booting from removable devices such as USB flash drives. This automated detection relies on standardized file paths to the OS boot-loader, with the path varying depending on the computer architecture. The format of the file path is defined as: `<EFI_SYSTEM_PARTITION>\EFI\BOOT\BOOT<MACHINE_TYPE_SHORT_NAME>.EFI`; for example, the file path to the OS loader on an x86-64 system is `\efi\boot\bootx64.efi` and `\efi\boot\bootaa64.efi` on ARM64 architecture.

Booting UEFI systems from GPT-partitioned disks is commonly called UEFI-GPT booting. Despite the fact that the UEFI specification requires MBR partition tables to be fully supported, some UEFI firmware implementations immediately switch to the BIOS based CSM booting depending on the type of boot disk's partition table, effectively preventing UEFI booting to be performed from EFI System Partition on MBR-partitioned disks. Such a boot scheme is commonly called UEFI-MBR.

It is also common for a boot manager to have a textual user interface so the user can select the desired OS (or setup utility) from a list of available boot options.

##### 1.3.6.2. CSM booting

##### 1.3.6.3. Networking booting

##### 1.3.6.4. Secure Boot

### 1.4. UEFI Shell
