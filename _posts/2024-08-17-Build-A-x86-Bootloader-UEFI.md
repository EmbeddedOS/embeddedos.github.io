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
- A class 3 is the same class 2 but without CSM supporting.

#### 1.2.1. UEFI Booting

UEFI specification define a **boot manager** that checks boot configurations, settings when a computer is powered on, and then decide to executes Bootloader, Kernel, Applications, etc.

The boot configurations is defined by variables stored in NVRAM (where to load kernel, bootloader, size, etc.).

UEFI can detected an OS bootloader (is stored in EFI system partition) automatically by using a standardize file paths: `<EFI_SYSTEM_PARTITION>\EFI\BOOT\BOOT<MACHINE_TYPE_SHORT_NAME>.EFI`. For example, on x64 arch `\efi\boot\bootx64.efi` and `\efi\boot\bootaa64.efi` on ARM64 arch.

#### 1.2.2. Secure boot

UEFI Secure Boot is a "Protocol" that is defined by UEFI. UEFI Secure Boot based on **Digital Signature** scheme. ONLY drivers, bootloader, kernel with signed key can be loaded.

UEFI that support Secure Boot is always in one of three states:

- Setup mode: UEFI applications can change or delete Platform Keys.
- User Mode and Secure Boot off: Applications can switch to Setup mode to configure keys.
- User Mode and Secure Boot on: Applications must be signed to be launched.

### 1.3. EFI System Partition

An EFI system partition (ESP), is a **data storage device partition** that is used computers adhering to the UEFI specification. Accessed by the UEFI firmware when a computer is powered up, it stores UEFI applications and the files these applications need to run, **including OS bootloader**.

The ESP also provides space for a boot sector as part of the backward BIOS compatibility.

The ESP is located at the beginning of the disk and its partition record at the beginning of the GPT. It can be formatted to any FAT file system FAT12, FAT16, FAT32.

Important Files on ESP:

- `FS0:\STARTUP.NSH` - An EFI Shell script, similar to MS-DOS `autoexec.bat`.
- `FS0:\BOOTMGR.EFI` - The EFI boot manager.
- `FS0:\BOOT\BOOTX86.EFI` - The default x86-32 bootloader.
- `FS0:\BOOT\BOOTX64.EFI` - The default x86-64 bootloader.

### 1.4. Services

EFI defines two types of services: **boot services** and **runtime services**. Boot services are available only while the firmware owns the platform (i.e. before the `ExitBootServices()` call), and they include text and graphical consoles on various devices, and bus, block and file services. Runtime services are still accessible while the OS is running; they include services such as date, time, variables (key-pair) and NVRAM access.

For example, with variable service, UEFI variables can be used to keep crash messages in NVRAM, after a crash for the Operating system to retrieve after a reboot.

## 2. Application Development

Beyond loading an OS, UEFI can run **UEFI applications**, which reside as files on the EFI system partition. They can be executed from the UEFI Shell, by the firmware's boot manager, or by other UEFI applications.

UEFI application examples: UEFI Shell, OS bootloader like GRUB, rEFInd, Gummiboot, and Windows Boot Manager, etc.

### 2.1. Developing Applications with GNU-EFI

GNU-EFI is a very lightweight developing environment to create UEFI applications. It is a set of libraries and headers for compiling UEFI applications with a system's native GCC.

#### 2.2. Download and compile

```bash
git clone https://git.code.sf.net/p/gnu-efi/code gnu-efi
cd gnu-efi
make
```

The GNU-EFI includes three main components:

- **Libraries**: are generated when you `make`:
  - `crt0-efi-x86_64.o`: A CRT0 (C runtime initialization code) that will call your `efi_main` function.
  - `libgnuefi.a`: A library contains a single function (`_relocate`) that is used by the CRT0.
  - `libefi.a`: A library contains convenience functions like CRC computation, text printing, etc.
- **Headers**: Convenience headers that provide structures, typedef, and constants improve readability when access UEFI resources.
- **Linker Script**: linker script to link your application with ELF binaries.

#### 2.3. Develop custom bootloader

> GNU-EFI uses the **host** compiler, you might need additional gcc options to get ABI work. The `uefi_call_wrapper()` is a wrapper function that makes sure every ABI always work. So there is no matter what ABI your gcc is using, `uefi_call_wrapper()` always correctly translate that into UEFI ABI. Using `uefi_call_wrapper()` whenever possible.
{: .prompt-info }

```c
// https://wiki.osdev.org/Loading_files_under_UEFI
EFI_STATUS
EFIAPI
efi_main(EFI_HANDLE ImageHandle, EFI_SYSTEM_TABLE *SystemTable)
{
  InitializeLib(ImageHandle, SystemTable);

  while (1)
  {
    Print(L"Hello, world!\n");
  }

  return EFI_SUCCESS;
}
```

#### 2.4. Creating an EFI executable

```bash
# Compile the application.
gcc -Ignu-efi/inc -fpic -ffreestanding -fno-stack-protector -fno-stack-check -fshort-wchar -mno-red-zone -maccumulate-outgoing-args -c main.c -o main.o

# Link with UEFI libraries.
ld -shared -Bsymbolic -Lgnu-efi/x86_64/lib -Lgnu-efi/x86_64/gnuefi -Tgnu-efi/gnuefi/elf_x86_64_efi.lds gnu-efi/x86_64/gnuefi/crt0-efi-x86_64.o main.o -o main.so -lgnuefi -lefi

# Converting Shared Object to EFI executable.
objcopy -j .text -j .sdata -j .data -j .rodata -j .dynamic -j .dynsym  -j .rel -j .rela -j .rel.* -j .rela.* -j .reloc --target efi-app-x86_64 --subsystem=10 main.so main.efi
```

### 2.2. Create disk images

To launch a our UEFI application, we need to create a disk image and present it to QEMU.
In reality, a disk image usually include an EFI system partition (contains bootloader, kernel, etc) and another partitions (for rootfs, data, etc. And kernel will take care of handling and mounting those partitions). In our system, the bootloader will only load a dummy kernel and do not thing, so we just need to create a simple disk image with EFI partition only.

Create a raw image and an EFI partition on it.

```bash
# Create a raw image.
$ dd if=/dev/zero of=uefi.img bs=512 count=93750

# Create a EFI partition on the image.
$ gdisk uefi.img

GPT fdisk (gdisk) version 1.0.5

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries in memory.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-93716, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-93716, default = 93716) or {+-}size{KMGTP}: 
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): ef00
Changed type of partition to 'EFI system partition'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to uefi.img.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot or after you
run partprobe(8) or kpartx(8)
The operation has completed successfully.
```

Attach the image into a loopback device and mount it into our system:

```bash
# Attach the image to a loopback device.
sudo losetup --offset 1048576 --sizelimit 46934528 /dev/loop99 uefi.img

# Format into FAT32 file system format.
sudo mkdosfs -F 32 /dev/loop99

# Mount the loopback device into `/mnt` directory.
sudo mount /dev/loop99 /mnt
```

Now we can read-write to the image via the mount point, we'll copy our UEFI applications and kernel dummy image in a directly way. If you want UEFI firmware execute your application automatically (as an OS bootloader) you can rename it to `EFI\BOOT\BOOTX64.EFI`.

```bash
sudo mkdir -p /mnt/EFI/BOOT/
sudo cp main.efi /mnt/EFI/BOOT/BOOTX64.EFI
sudo cp kernel.bin /mnt/EFI

# Optional, if you custom your startup script.
sudo cp startup.nsh /mnt/EFI/STARTUP.NSH
```

Unmount and detach the loopback device:

```bash
sudo umount /mnt
sudo losetup -d /dev/loop99
```

### 2.3. Emulate

If you choose VirtualBox for virtualization, UEFI is already included, no need to download the image manually. Just enable it.

otherwise for emulation and virtual machines, we need an **OVMF.fd** firmware image (OVMF is a port of Intel's tianocore firmware to the qemu virtual machine). Install on Debian/Ubuntu:

```bash
# Install the firmware.
apt-get install ovmf

# Boot system with the firmware like BIOS firmware.
qemu-system-x86_64 -cpu qemu64 -bios /usr/share/qemu/OVMF.fd
```

```bash
qemu-system-x86_64 -cpu qemu64 -bios /usr/share/qemu/OVMF.fd -drive file=uefi.img,if=ide
```
