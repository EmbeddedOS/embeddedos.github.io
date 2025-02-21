---
title: "Executable files"
description: >-
  Understand executable formats and how they are executed in various platforms.

author: Cong
date: 2025-02-21 00:01:00 +0800
categories: [Compiling]
tags: [executable, linux, bare metal]
image:
  path: assets/img/service_call_routing.png
  alt: aarch64 exception levels.
published: false
---

Targets:

- Understand executable file formats like binary, ELF, and PE.
- How bare metal images are loaded in embedded system.
- Understand Linux kernel image formats and how they are loaded.

## 1. Executable file formats

### 1.1. Binary

Executable binary files is nothing but a sequence of bytes. These files contain no additional information to execute. To execute them, the executable file loader MUST know **where to locate the binary** and **where to start executing**. That means the loader and the compiler have to deal with each other to exchange these information in some way.

The embedded systems are mostly using this way, for example, we flash the binary file onto well-known address, let's say, FLASH memory starting point `0x0800 0000`, where the CPU will automatically start executing after a reset. In that case, the execution entry point will be at the same address. The flasher have no idea about these addresses, we must to specific them ourselves.

### 1.2. ELF

Executable & Linkable format (aka. ELF) is a standard file format for executable files. Unlike raw binary files, The ELF image contains metadata that describes itself. Each ELF file includes four parts:

1. ELF header, basic executable file information, entry point, and info to get program header table and section header table.
2. Program header table, describe memory segments.
3. Section header table, describe sections.
4. Program Sections.

![ELF File Layout](assets/img/elf_file_layout.png)

First thing first, the ELF header is located at the beginning of the ELF file. Start with magic numbers and followed by basic information like type, machine, version, etc. The ELF header also contains `e_entry` this is the memory address of the entry point from where the process starts executing. Other important fields are, the `e_phoff` that points to the program header table and the `e_shoff` that points to the section header table. By using them, the loader can locate these tables location in the ELF file.

Program header table contains memory segments that are used for runtime execution of the file, it tells the loader how to create a process image. Whereas, the section header table is used for locating sections on the file image and explaining them.

There are some special sections do special jobs. For example, `SHT_DYNAMIC` section holds the information for dynamic linking, for example, shared libraries. `SHT_SYMTAB` holds a symbol table, that might be useful for debugging.

#### 1.2.2. Loading an ELF

1. The loader reads the ELF header, validates, and locates the program header table.
2. Creates a process memory, loads loadable segments (`p_type` equal `PT_LOAD`) into memory based on the entry on the program table. The entry contains all the information to load the segment into memory, for example, the memory permission (read/write/executable) and the virtual memory address.
3. Load dynamic linking programs, shared libraries if required (in `SHT_DYNAMIC`section).
4. Pass the CPU control at the entry point `e_entry` to start the program.

![ELF Loader load segments](assets/img/elf_file_layout.png)

### 1.3. PE

## 2. Linux kernel image formats

- <https://stackoverflow.com/questions/74325349/exact-difference-between-image-and-vmlinux>
- <https://users.informatik.haw-hamburg.de/~krabat/FH-Labor/gnupro/5_GNUPro_Utilities/c_Using_LD/ldLinker_scripts.html>
- <https://github1s.com/krinkinmu/aarch64/blob/master/bootstrap/Makefile>
- <https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html>

## 3. Loading executable files

### 3.1. Bare metal programs

### 3.2. User programs in OS environment

### 3.3. Bootloader
