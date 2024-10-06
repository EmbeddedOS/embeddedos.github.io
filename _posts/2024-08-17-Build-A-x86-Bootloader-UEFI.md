---
title: 'Build your own bootloader for x86-64 OS: UEFI'
description: >-
  Let's discover what is UEFI and how to use it to load our custom x86-64 OS loader and boot system.
  
author: Cong
date: 2024-08-17 7:00:00 +0800
categories: [Bootloader-x86, UEFI]
tags: [Bootloader, UEFI, x86, QEMU, Assembly, GNU-EFI]
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

### 2.2. Download and compile

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

### 2.3. Develop custom bootloader

> GNU-EFI uses the **host** compiler, you might need additional gcc options to get ABI work. The `uefi_call_wrapper()` is a wrapper function that makes sure every ABI always work. So there is no matter what ABI your gcc is using, `uefi_call_wrapper()` always correctly translate that into UEFI ABI. Using `uefi_call_wrapper()` whenever possible.
{: .prompt-info }

To make it more convenient, we develop file basic operation functions `open()`, `close()`, `read()`. The `uefi_get_volume()` function return the file system protocol that is used to do basic file operations.

```c
#include "file.h"

/* Public functions ----------------------------------------------------------*/
EFI_FILE_HANDLE uefi_get_volume(EFI_HANDLE image)
{
  EFI_LOADED_IMAGE *loaded_image = NULL;
  EFI_GUID lipGuid = EFI_LOADED_IMAGE_PROTOCOL_GUID;
  EFI_FILE_IO_INTERFACE *IOVolume = NULL;
  EFI_GUID fsGuid = EFI_SIMPLE_FILE_SYSTEM_PROTOCOL_GUID;
  EFI_FILE_HANDLE Volume;

  /* 1. Get the loaded image protocol interface for our "image". */
  uefi_call_wrapper(BS->HandleProtocol, 3, image, &lipGuid,
                    (void **)&loaded_image);

  /* 2. Get the volume handle. */
  uefi_call_wrapper(BS->HandleProtocol, 3, loaded_image->DeviceHandle, &fsGuid,
                    (VOID *)&IOVolume);
  uefi_call_wrapper(IOVolume->OpenVolume, 2, IOVolume, &Volume);

  return Volume;
}

UINT64 uefi_get_file_size(EFI_FILE_HANDLE file_handle)
{
  UINT64 ret = 0;
  EFI_FILE_INFO *FileInfo = NULL;

  FileInfo = LibFileInfo(file_handle);

  ret = FileInfo->FileSize;
  FreePool(FileInfo);
  return ret;
}

EFI_STATUS uefi_open_file(EFI_FILE_HANDLE volume,
                          const CHAR16 *filename,
                          EFI_FILE_HANDLE *file_handle)
{
  EFI_STATUS status = EFI_SUCCESS;
  status = uefi_call_wrapper(
      volume->Open, 5,
      volume,
      file_handle,
      filename,
      EFI_FILE_MODE_READ,
      EFI_FILE_READ_ONLY | EFI_FILE_HIDDEN | EFI_FILE_SYSTEM);

  if (EFI_ERROR(status))
  {
    Print(L"Failed to open file: %d\n", status);
  }

  return status;
}

EFI_STATUS uefi_close_file(EFI_FILE_HANDLE file_handle)
{
  EFI_STATUS status = EFI_SUCCESS;
  status = uefi_call_wrapper(file_handle->Close, 1, file_handle);

  if (EFI_ERROR(status))
  {
    Print(L"Failed to close file: %d\n", status);
  }

  return status;
}

EFI_STATUS uefi_read_file(EFI_FILE_HANDLE file_handle,
                          UINT8 *buffer, UINT64 size)
{
  EFI_STATUS status = EFI_SUCCESS;
  UINT64 read_size = size;
  status = uefi_call_wrapper(file_handle->Read, 3,
                             file_handle, &read_size,
                             buffer);
  if (EFI_ERROR(status))
  {
    Print(L"Failed to close file: %d\n", status);
  }
  else if (read_size != size)
  {
    Print(L"Can't get %d bytes, actual reading size: %d\n", size, read_size);
  }

  return status;
}
```

#### 2.3.1. Load binary kernel

To load a kernel in binary format, we simply allocate the memory for kernel, and jump to the binary start as a kernel entry point:

```c
EFI_STATUS load_binary_kernel(UINT8 *buffer,
                              UINT64 size,
                              void **entry_point)
{
    UINT8 * kernel_buffer = NULL;
    kernel_buffer = AllocatePool(size);
    uefi_call_wrapper(CopyMem, 3, kernel_buffer, buffer, size);

    *entry_point = (VOID *)kernel_buffer;
    return EFI_SUCCESS;
}
```

#### 2.3.2. Load ELF kernel

An ELF kernel start with an ELF header, we parse the ELF structure to get the executable information and then load it into memory. Below here is the ELF header in C format:

```c
typedef struct
{
    struct e_ident_t
    {
        UINT8 ei_magic0;      /* Magic number 0: '0x7F'.                    */
        UINT8 ei_magic1;      /* Magic number 1: 'E'.                       */
        UINT8 ei_magic2;      /* Magic number 2: 'L'.                       */
        UINT8 ei_magic3;      /* Magic number 3: 'F'.                       */
        UINT8 ei_class;       /* Signify 32- or 64-bit format.              */
        UINT8 ei_data;        /* Signify little or big endianness.          */
        UINT8 ei_version;     /* Signify Original and current version.      */
        UINT8 ei_os_abi;      /* Identifies target Operating System.        */
        UINT8 ei_abi_version; /* Specify the ABI version.                   */
        UINT8 ei_pad[7];      /* Reversed padding bytes.                    */
    } e_ident;                /* Identifies field.                          */

    UINT16 e_type;      /* Identifies object type.                            */
    UINT16 e_machine;   /* Specifies target instruction set arch.             */
    UINT32 e_version;   /* Signify original version.                          */
    UINT64 e_entry;     /* Memory address of the entry point.                 */
    UINT64 e_phoff;     /* Points to the start of the program header table.   */
    UINT64 e_shoff;     /* Points to the start of the section header table.   */
    UINT32 e_flags;     /* Flags, depends on architecture.                    */
    UINT16 e_ehsize;    /* Size of this header.                               */
    UINT16 e_phentsize; /* Size of a program header table entry.              */
    UINT16 e_phnum;     /* Number of entries in the program header table.     */
    UINT16 e_shentsize; /* Size of a section header table entry.              */
    UINT16 e_shnum;     /* The number of entries in the section header table. */
    UINT16 e_shstrndx;  /* Index of the section header table entry.           */
} __attribute__((packed)) elf64_header_t;

typedef struct
{
    UINT32 p_type;    /* Identifies the type of the segment.              */
    UINT32 p_flags64; /* Segment-dependent flags.                         */
    UINT64 p_offset;  /* Offset of the segment in memory.                 */
    UINT64 p_vaddr;   /* Virtual address of the segment in memory.        */
    UINT64 p_paddr;   /* On systems where physical is relevant.           */
    UINT64 p_filesz;  /* Size in bytes of the segment in the file image.  */
    UINT64 p_memsz;   /* Size in bytes of the segment in memory.          */
    UINT64 p_align;   /* Signify no alignment.                            */
} __attribute__((packed)) elf64_program_header_t;
```

The ELF file start with 4 magic characters `0x7F` follow by `E`, `L` and `F`, we use them to determine the ELF format. Based on ELF header metadata, next thing we need to do is calculate and allocate memory for loadable sections. After loading these sections to memory, we get the kernel entry point based on the offset `e_entry` header field.

```c
EFI_STATUS load_elf_kernel(UINT8 *buffer,
                           UINT64 size,
                           void **entry_point)
{
    EFI_STATUS res = EFI_SUCCESS;
    elf64_header_t *header = (elf64_header_t *)buffer;
    elf64_program_header_t *program_header = NULL;
    UINT64 max_alignment = PAGE_SIZE;
    UINT64 mem_start = UINT64_MAX;
    UINT64 mem_end = 0;
    uint32_t needed_memory_size = 0;
    VOID *program_memory_buffer = NULL;

    if (header->e_ident.ei_magic0 != 0x7F ||
        header->e_ident.ei_magic1 != 'E' ||
        header->e_ident.ei_magic2 != 'L' ||
        header->e_ident.ei_magic3 != 'F')
    {
        Print(L"kernel is not in EFL format.\n");
        return EFI_LOAD_ERROR;
    }

    if (header->e_type != ELF64_E_TYPE_ET_DYN)
    { // Only access shared object.
        Print(L"ELF type is not supported: %d\n", header->e_type);
        return EFI_LOAD_ERROR;
    }

    Print(L"Loading ELF kernel...\n");
    print_elf_info(buffer);

    /* 1. Calculate memory bounds for all program sections. */
    program_header = (elf64_program_header_t *)(buffer + header->e_phoff);
    for (INT32 i = 0; i < header->e_phnum; i++, program_header++)
    {
        if (program_header->p_type == ELF64_P_PT_LOAD)
        { // Handle loadable segment only.
            if (program_header->p_align > max_alignment)
            {
                max_alignment = program_header->p_align;
            }

            UINT64 segment_mem_begin = program_header->p_vaddr;
            UINT64 segment_mem_end =
                program_header->p_vaddr +
                program_header->p_memsz + max_alignment - 1;

            segment_mem_begin &= ~(max_alignment - 1);
            segment_mem_end &= ~(max_alignment - 1);

            if (segment_mem_begin < mem_start)
            {
                mem_start = segment_mem_begin;
            }

            if (segment_mem_end > mem_end)
            {
                mem_end = segment_mem_end;
            }
        }
    }

    needed_memory_size = mem_end - mem_start;

    Print(L"Needed kernel's memory: 0x%x\n", needed_memory_size);

    /* 2. Allocate buffer for program headers. */
    res = uefi_call_wrapper(BS->AllocatePool, 3,
                            EfiLoaderData,
                            needed_memory_size,
                            &program_memory_buffer);
    if (EFI_ERROR(res))
    {
        Print(L"Cannot allocate memory for program sections!\n");
        return res;
    }

    /* 3. Load loadable section into memory. */
    program_header = (elf64_program_header_t *)(buffer + header->e_phoff);
    for (INT32 i = 0; i < header->e_phnum; i++, program_header++)
    {
        if (program_header->p_type == ELF64_P_PT_LOAD)
        { // Handle loadable segment only.

            UINT64 relative_offset = program_header->p_vaddr - mem_start;
            UINT8 *dst = (UINT8 *)program_memory_buffer + relative_offset;
            UINT8 *src = (UINT8 *)buffer + program_header->p_offset;
            UINT32 len = program_header->p_filesz;

            uefi_call_wrapper(CopyMem, 3, dst, src, len);

            Print(L"Loaded %p to %p len: %x, offset: %ld\n",
                  src, dst, len, relative_offset);
        }
    }

    Print(L"Program memory: %p, entry offset: %x, start: 0x%x\n",
          program_memory_buffer, header->e_entry, mem_start);

    /* 4. Update entry point. */
    *entry_point = (VOID *)((UINT8 *)program_memory_buffer +
                            (header->e_entry - mem_start));

    return EFI_SUCCESS;
}
```

#### 2.3.3. Build custom kernel

In this blog, we focus on developing the OS loader, so for the kernel will be minimal. The kernel parameters keep basic information and services: memory map and UEFI runtime services ans graphic output protocol:

```c
typedef struct
{
    UINTN mm_size;
    EFI_MEMORY_DESCRIPTOR *mm_descriptor;
    UINTN map_key;
    UINTN descriptor_size;
    UINT32 descriptor_version;
} memory_map_t;

/**
 * @brief   - Kernel boot parameters structure. This structure will be shared
 *            with uefi os loader code. The loader will pass this param to
 *            kernel when it's passing the control to kernel.
 */
typedef struct
{
    memory_map_t mm;
    EFI_RUNTIME_SERVICES *runtime_services;
    EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE graphic_out_protocol;
    UINTN custom_protocol_data;
} boot_params_t;
```

In the kernel source file, we change the graphic color to make sure our kernel is running:

```c
#include "kernel.h"

void main(boot_params_t *params)
{
    UINT32 *frame_buffer = NULL;
    UINT32 x_res = 0;
    UINT32 y_res = 0;

    /* 1. We get Graphic Output Protocol to change graphic mode in kernel code.
     */
    frame_buffer = (UINT32 *)params->graphic_out_protocol.FrameBufferBase;
    x_res = params->graphic_out_protocol.Info->PixelsPerScanLine;
    y_res = params->graphic_out_protocol.Info->VerticalResolution;

    for (UINT32 y = 0; y < y_res; y++)
    {
        for (UINT32 x = 0; x < x_res; x++)
        {
            frame_buffer[x + y * x_res] = 0xFFCC2222;
        }
    }

        for (UINT32 y = 0; y < y_res/50; y++)
    {
        for (UINT32 x = 0; x < x_res/50; x++)
        {
            frame_buffer[x + y * x_res] = 0xFFCC2222;
        }
    }

    params->custom_protocol_data;
}
```

#### 2.3.4. Setup environment and load the kernel

The main OS loader do 3 main tasks:

- Load the kernel into memory.
- Setup environment & boot parameters for the kernel.
- Exit boot services and pass control to kernel.

```c
/* Public defines ------------------------------------------------------------*/
#define KERNEL_IMAGE_PATH L"kernel.elf"
#define CUSTOM_PROTOCOL_DATA 123

/* Public functions prototypes -----------------------------------------------*/
EFI_GRAPHICS_OUTPUT_PROTOCOL *
uefi_get_graphic_output_protocol();

EFI_STATUS
uefi_install_custom_protocol(EFI_HANDLE ImageHandle);

CUSTOM_PROTOCOL *
uefi_get_custom_protocol();

EFI_STATUS
uefi_get_mm(memory_map_t *mm);

/**
 * @brief   - Get a character from keyboards.
 */
EFI_INPUT_KEY uefi_get_key(void);

/* Public functions ----------------------------------------------------------*/
EFI_STATUS
EFIAPI
efi_main(EFI_HANDLE ImageHandle, EFI_SYSTEM_TABLE *SystemTable)
{
  EFI_STATUS res = EFI_SUCCESS;
  EFI_FILE_HANDLE fs_volume;
  EFI_FILE_HANDLE file_handle;
  UINT64 file_size = 0;
  UINT8 *buffer = NULL;
  boot_params_t kernel_params = {0};
  EFI_GRAPHICS_OUTPUT_PROTOCOL *gop = NULL;
  CUSTOM_PROTOCOL *custom_protocol = NULL;
  void *entry_point = NULL;

  InitializeLib(ImageHandle, SystemTable);

  res = uefi_install_custom_protocol(ImageHandle);
  if (EFI_ERROR(res))
  {
    Print(L"Failed to install custom protocol: %d\n", res);
    goto exit;
  }

  /* 1. Configure screen colours. */
  uefi_call_wrapper(SystemTable->ConOut->SetAttribute, 2,
                    SystemTable->ConOut,
                    EFI_TEXT_ATTR(EFI_BLUE, EFI_LIGHTGRAY));

  uefi_call_wrapper(SystemTable->ConOut->ClearScreen, 1, SystemTable->ConOut);

  /* 2. Load kernel. */
  res = uefi_load_kernel(ImageHandle, SystemTable,
                         KERNEL_IMAGE_PATH, &entry_point);

  /* 3. Make kernel parameters. */
  gop = uefi_get_graphic_output_protocol();
  if (gop == NULL)
  {
    goto uefi_get_graphic_output_protocol_failure;
  }

  custom_protocol = uefi_get_custom_protocol();
  if (custom_protocol)
  {
    kernel_params.custom_protocol_data = custom_protocol->data;
  }

  kernel_params.runtime_services = SystemTable->RuntimeServices;
  kernel_params.graphic_out_protocol = *gop->Mode;
  /* 4. Jump to kernel. */
  Print(L"Press any key to enter to kernel...\n");
  uefi_get_key();
  uefi_call_wrapper(SystemTable->ConOut->ClearScreen, 1, SystemTable->ConOut);

  res = uefi_get_mm(&kernel_params.mm);
  if (EFI_ERROR(res))
  {
    Print(L"Failed to get memory map: %d\n", res);
    goto uefi_get_mm_failure;
  }

  /* NOTE: Don't do anything between get memory and exit boot services step. */

  res = uefi_call_wrapper(BS->ExitBootServices, 2,
                          ImageHandle,
                          kernel_params.mm.map_key);
  if (EFI_ERROR(res))
  {
    Print(L"Failed to exit boot services: %d\n", res);
    goto ExitBootServices_failure;
  }

  ((kernel_entry)entry_point)(&kernel_params);

ExitBootServices_failure:
uefi_get_mm_failure:
uefi_get_graphic_output_protocol_failure:
  /* TODO: cleanup memory pool. */

exit:
  Print(L"Failed to load kernel, press any key to exit...\n");
  uefi_get_key();

  return res;
}

EFI_GRAPHICS_OUTPUT_PROTOCOL *
uefi_get_graphic_output_protocol()
{
  EFI_STATUS status = EFI_SUCCESS;
  EFI_GUID guid = EFI_GRAPHICS_OUTPUT_PROTOCOL_GUID;
  EFI_GRAPHICS_OUTPUT_PROTOCOL *gop = NULL;

  status = uefi_call_wrapper(BS->LocateProtocol, 3, &guid, NULL, (void **)&gop);
  if (EFI_ERROR(status))
  {
    Print(L"Failed to locate graphics output protocol: %d\n", status);
    return NULL;
  }

  return gop;
}

EFI_INPUT_KEY uefi_get_key(void)
{
  EFI_EVENT events[1];
  EFI_INPUT_KEY key;
  UINTN index = 0;

  key.ScanCode = 0;
  key.UnicodeChar = u'\0';
  events[0] = ST->ConIn->WaitForKey;

  uefi_call_wrapper(BS->WaitForEvent, 3, 1, events, &index);

  if (index == 0)
  {
    uefi_call_wrapper(ST->ConIn->ReadKeyStroke, 2, ST->ConIn, &key);
  }

  return key;
}

EFI_STATUS
uefi_get_mm(memory_map_t *mm)
{
  EFI_STATUS res = EFI_SUCCESS;

  /* 1. Get memory size by pass size 0. */
  res = uefi_call_wrapper(BS->GetMemoryMap, 5,
                          &mm->mm_size,
                          mm->mm_descriptor,
                          &mm->map_key,
                          &mm->descriptor_size,
                          &mm->descriptor_version);
  if (EFI_ERROR(res) && res != EFI_BUFFER_TOO_SMALL)
  {
    return res;
  }

  /* 2. Update new memory size to get. */
  mm->mm_size += mm->descriptor_size * 2;
  res = uefi_call_wrapper(BS->AllocatePool, 3,
                          EfiLoaderData,
                          mm->mm_size,
                          &mm->mm_descriptor);
  if (EFI_ERROR(res))
  {
    return res;
  }

  /* 3. Get memory map. */
  res = uefi_call_wrapper(BS->GetMemoryMap, 5,
                          &mm->mm_size,
                          mm->mm_descriptor,
                          &mm->map_key,
                          &mm->descriptor_version,
                          &mm->descriptor_size);
  if (EFI_ERROR(res))
  {
    uefi_call_wrapper(BS->FreePool, 1, mm->mm_descriptor);
  }

  return res;
}

EFI_STATUS
uefi_install_custom_protocol(EFI_HANDLE ImageHandle)
{
  EFI_STATUS res = EFI_SUCCESS;
  CUSTOM_PROTOCOL *custom_protocol = NULL;
  EFI_GUID guid = EFI_CUSTOM_PROTOCOL_GUID;

  res = uefi_call_wrapper(BS->AllocatePool, 3,
                          EfiBootServicesData,
                          sizeof(CUSTOM_PROTOCOL),
                          (VOID **)&custom_protocol);
  if (EFI_ERROR(res))
  {
    return res;
  }

  custom_protocol->data = CUSTOM_PROTOCOL_DATA;

  res = uefi_call_wrapper(BS->InstallProtocolInterface, 4,
                          &ImageHandle,
                          &guid,
                          EFI_NATIVE_INTERFACE,
                          custom_protocol);
  if (EFI_ERROR(res))
  {
    uefi_call_wrapper(BS->FreePool, 1, custom_protocol);
  }

  return res;
}

CUSTOM_PROTOCOL *
uefi_get_custom_protocol()
{

  EFI_STATUS status = EFI_SUCCESS;
  EFI_GUID guid = EFI_CUSTOM_PROTOCOL_GUID;
  CUSTOM_PROTOCOL *custom_protocol = NULL;

  status = uefi_call_wrapper(BS->LocateProtocol, 3,
                             &guid, NULL, (void **)&custom_protocol);
  if (EFI_ERROR(status))
  {
    Print(L"Failed to locate custom protocol: %d\n", status);
    return NULL;
  }

  return custom_protocol;
}
```

### 2.4. Create a bootable disk image

To launch a our UEFI application, we need to create a disk image and present it to QEMU.
In reality, a disk image usually include an EFI system partition (contains bootloader, kernel, etc) and another partitions (for rootfs, data, etc. And kernel will take care of handling and mounting those partitions). In our system, the bootloader will only load a dummy kernel and do not thing, so we just need to create a simple disk image with EFI partition only.

Create a raw image and an EFI partition on it.

```bash
# 1. Create raw image.
dd if=/dev/zero of=uefi.img bs=512 count=93750

# 2. Create UEFI partition on the image.
gdisk uefi.img <<EOF
o
Y
n



ef00
w
Y
EOF

# 3. Format into the FAT32 file system. 
sudo losetup --offset 1048576 --sizelimit 46934528 /dev/loop99 uefi.img
sudo mkdosfs -F 32 /dev/loop99
sudo losetup -d /dev/loop99
```

### 2.5. Make all system

```make
CC=gcc
LD=ld
INC=-Ignu-efi/inc -I. -Iloaders
CFLAGS=-fpic -ffreestanding -fno-stack-protector -fno-stack-check -fshort-wchar -mno-red-zone -maccumulate-outgoing-args
LDFLAGS=-shared -Bsymbolic
LDLIB_DIRS=-Lgnu-efi/x86_64/lib -Lgnu-efi/x86_64/gnuefi
LDLIBS=-lgnuefi -lefi
LD_LINKER_FILE=gnu-efi/gnuefi/elf_x86_64_efi.lds
LD_STARTUP_FILE=gnu-efi/x86_64/gnuefi/crt0-efi-x86_64.o
OBJCOPY_FLAGS=-j .text -j .sdata -j .data -j .rodata -j .dynamic -j .dynsym  -j .rel -j .rela -j .rel.* -j .rela.* -j .reloc --target efi-app-x86_64 --subsystem=10
BOOTLOADER_IMG=main.efi
KERNEL_IMG=kernel.elf
OBJS= main.o file.o loaders/elf.o loaders/binary.o loaders/loader.o
OS_IMAGE=uefi.img

lib:
	sudo make -C gnu-efi install

app:$(BOOTLOADER_IMG)
	echo "Built BOOTLOADER_OBJ"

kernel.bin:
	gcc -ffreestanding $(INC) -c kernel.c -o kernel.o -fPIE
	ld -o kernel.bin kernel.o -nostdlib --oformat=binary -e main

kernel.elf:
	gcc -ffreestanding $(INC) kernel.c -o kernel.elf -nostdlib -e main

image: $(OS_IMAGE) $(BOOTLOADER_IMG) $(KERNEL_IMG)
	sudo losetup --offset 1048576 --sizelimit 46934528 /dev/loop99 uefi.img
	sudo mount /dev/loop99 /mnt
	sudo mkdir -p /mnt/EFI/BOOT/
	sudo cp $(BOOTLOADER_IMG) /mnt/EFI/BOOT/BOOTX64.EFI
	sudo cp $(KERNEL_IMG) /mnt/
	sudo umount /mnt
	sudo losetup -d /dev/loop99

all: lib image $(KERNEL_IMG)

clean:
	rm -rf *.o *.so *img *.efi *.elf *.bin loaders/*.o

%.o: %.c
	$(CC) $(CFLAGS) $(INC) -c $< -o $@

%.so: $(OBJS)
	$(LD) $(LDFLAGS) $(LDLIB_DIRS) -T$(LD_LINKER_FILE) $(LD_STARTUP_FILE) $(OBJS) -o $@ $(LDLIBS)

%.efi: %.so
	objcopy $(OBJCOPY_FLAGS) $< $@

%.img:
	sudo ./create-img.sh
```

`make image` to build all system.

### 2.6. Emulate

If you choose VirtualBox for virtualization, UEFI is already included, no need to download the image manually. Just enable it.

otherwise for emulation and virtual machines, we need an **OVMF.fd** firmware image (OVMF is a port of Intel's tianocore firmware to the qemu virtual machine). Install on Debian/Ubuntu:

```bash
# Install the firmware.
apt-get install ovmf
```

Start system with qemu:

```bash
sudo qemu-system-x86_64 -cpu qemu64 -bios /usr/share/qemu/OVMF.fd -drive format=raw,unit=0,file=uefi.img -m 256M -vga std -net none
```

System load the kernel and print ELF information:

![Kernel Information](assets/img/UEFI_print_kernel_info.png)

Kernel is loaded and replace screen with the red color:

![Kernel Running](assets/img/UEFI_pass_control_to_kernel.png)

### 3.References

Full source code on [github](https://github.com/EmbeddedOS/uefi-bootloader)
