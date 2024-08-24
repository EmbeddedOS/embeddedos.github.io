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

To ensure backward compatibility, UEFI firmware implementations on PC-class machines could support booting in legacy BIOS mode from MBR partitioned disks through the **Compatibility Support Module (CSM)** that provides legacy BIOS compatibility.

BIOS-style booting from MBR-partitioned disks is commonly called BIOS-MBR, regardless of it being performed on UEFI or legacy BIOS-based systems. Furthermore, booting legacy BIOS-based systems from GPT disks is also possible, and such a boot scheme is commonly called BIOS-GPT.

The Compatibility Support Module allows OSes and some legacy option ROMs that do not support UEFI to still be used.

##### 1.3.6.3. Networking booting

The UEFI specification includes support for booting over network via the **Pre-boot eXecution Environment** (PXE). PXE booting network protocols include internet protocol (IPv4 and Ipv6), User Data-gram Protocol (UDP), Dynamic Host Configuration Protocol (DHCP), TFTP, and iSCSI.

OS images can be remotely stored on Storage Area Networks (SANs).

Version 2.5 of the UEFI specification adds support for accessing boot images over the HTTP.

##### 1.3.6.4. Secure Boot

The UEFI specification defines a protocol known as **Secure Boot**, which can secure the boot process by preventing the loading of UEFI drivers or OS boot loaders that are not **singed** with an acceptable **digital signature**.

When Secured Boot is enabled, it is initially placed in **setup** mode, which allows a public key from known as the **Platform key** (PK) to be written to the firmware. Once the key is written, Secure Boot enters **User mode**, where only UEFI drivers and OS boot-loaders signed with the **Platform Key** can be loaded by the firmware. Additional **Key Exchange Keys** (KEK) can be added to a database stored in memory to allow other certificates to be used, but they must still have a connection to the private portion of the platform key.

Secure Boot can also be placed in **Custom** mode, where additional public keys can be added to the system that do not match the private key.

#### 1.3.7. UEFI Shell

UEFI provides a **shell environment**, which can be used to execute other UEFI applications, including UEFI bootloader. Apart from that, commands available in the UEFI shell can be used for obtaining various other information about the system or the firmware, including getting the memory map (`memmap`), modifying boot manager variables (`bcfg`), running partitioning programs (`diskpart`), loading UEFI drivers, and editing text files (`edit`).

Methods used for launching UEFI shell depend on the manufacture and model of the system motherboard. e.g. x86: `<EFI_SYSTEM_PARTITION>/SHELLX64.EFI`, some other system have an already embedded UEFI shell which can be launched by appropriate press combinations.

#### 1.3.8. UEFI capsule

UEFI capsule defines a Firmware-to-OS firmware update interface, marketed as modern and secure.

#### 1.3.9. Hardware

Like BIOS, UEFI initializes and tests system hardware components (e.g. Memory Training, PCIe link training, USB training), and then loads the bootloader from a mass storage device or through a network connection. In x86 systems, the UEFI firmware is usually stored in the NOR flash chip of the motherboard.

### 1.4. Classes

UEFI machines can have one of the following classes, which were used to help ease the transition to UEFI:

- Class 0: Legacy BIOS.
- Class 1: UEFI with a CSM interface and no external UEFI interface.
- Class 2: UEFI with a CSM and external interfaces, eg: UEFI boot.
- Class 3: UEFI without a CSM interface and with an external UEFI interface.
- Class 3+: UEFI class 3 that has Secure Boot enabled.

### 1.5. Boot stages

- Step 1: **SEC - Security Phase**:
  - This is the first stage of the UEFI boot but may have platform specific binary code that precedes it (e.g: Intel ME, CPU microcode).
  - It consists of minimal code written in assembly language for the specific architecture.
  - It initializes a temporary memory (often CPU cache as RAM, or SoC on-chip SRAM) and serves as the system's software root of trust with the option of verifying PEI before hand-off.
- Step 2: **PEI - Pre-EFI Initialization**:
  - The second stage of UEFI boot consists of a dependency-aware dispatcher that loads and runs PEI modules to handle early hardware initialization task such as main memory initialization (initialize memory controller and DRAM) and firmware recovery operations.
  - Additionally, it is responsible for discovery of the current boot mode and handling many ACPI S3 operations.
- Step 3: **DXE - Driver Execution Environment**:
  - This stage consist of C modules and a dependency-aware dispatcher. With main memory now available, CPU, chipset, main-board and other I/O devices are initialized in DXE and BDS.
- Step 4: **BDS - Boot Device Select**:
  - BDS is a part of the DXE. In this stage, boot devices are initialized, UEFI drivers or Option ROMs of PCI devices are executed according to system configuration, and boot options are processed.
- Step 5: **TSL - Transient System Load**:
  - This is the stage between boot device selection and hand-off to the OS. At this point one may enter UEFI shell, or execute an UEFI application such as the OS boot loader.
- Step 6: **RT - Runtime**:
  - The UEFI hands off to the OS after `ExitBootServices()` is executed. A UEFI compatible OS is now responsible for exiting boot services triggering the firmware to unload all no longer needed code and data, leaving only runtime services code/data, e.g. SMM and ACPI. A typical modern OS will prefer to use its own programs (such as kernel drivers) to control hardware devices.

### 1.6. Applications development

**EDK2 Application Development Kit** (EADK) makes it possible to use standard C library functions in UEFI applications. EADK can be freely downloaded from the Intel's TianoCore UDK / EDK2 SourceForge project. Github: [EADK](https://github.com/tianocore/edk2)

A minimalistic `hello world` C program written using EADK looks similar to its usual C counterpart:

```c
#include <Uefi.h>
#include <Library/UefiLib.h>
#include <Library/ShellEntryLib.h>

EFI_STATUS EFIAPI ShellAppMain(IN UINTN Argc, IN CHAR16 **Argv)
{
    Print(L"hello, world\n");
    return EFI_SUCCESS;
}
```

## 2. OSDev

UEFI is a specification for x86, x86-64, ARM and Itanium platforms that defines a **software interface** between the OS and the platform firmware/BIOS.

### 2.1. UEFI Basics

#### 2.1.1. Download UEFI images

If you choose VirtualBox for virtualization, UEFI is already included, no need to download the image manually. Just enable it.

otherwise for emulation and virtual machines, we need an **OVMF.fd** firmware image. Install on Debian/Ubuntu:

```bash
apt-get install ovmf
```

#### 2.1.2. UEFI vs legacy BIOS

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

#### 2.1.3. Emulation