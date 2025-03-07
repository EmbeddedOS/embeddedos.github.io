---
title: "Writing Aarch64 bare metal bootloader."
description: >-
  Writing bare metal boot code for Aarch64.

author: Cong
date: 2025-03-05 00:01:00 +0800
categories: [bootloader]
tags: [aarch64, exception level, bare metal, qemu, gdb]
published: true
---

> Full source code now is available on this repo: [AArch64 Boot Code](https://github.com/EmbeddedOS/aarch64_boot_code).
{: .prompt-info }

## 1. Objective

- Understand Aarch64 booting flow, exception levels.
- Using QEMU as a base platform.
- Firmware can boot from any EL level and able to change Exception Level from EL3 to EL0.
- Firmware is able to be loaded to ROM and perform self-relocating or loaded to RAM and running directly.
- Using system call `svc` from EL0 user space to request EL1 Kernel service.
- Debugging system with GDB.

## 2. Hardware platform - QEMU

The platform I choose is QEMU, that make us easy to run, debug and able to access by everyone. The machine (SoC) would be `virt` and the cpu would be `cortex-a57`.

The `virt` machine [Memory Layout](https://github.com/qemu/qemu/blob/master/hw/arm/virt.c#L160) might look like this:

```text
 _______________ 0x00000000
|     Flash     |
|_______________|0x08000000
|      ...      |
|_______________|0x09000000
|     USART0    |
|_______________|0x09001000
|      ...      |
|_______________|0x40000000
|      RAM      |
|_______________|RAM_LIMIT
```

- 0..128Mb is space for a flash device, that is used for boot rom code such as UEFI.
- 128MB..256MB is used for device I/O.
- 256MB..1GB is reserved for possible future PCI support.
- 1GB and up is RAM.

 Because we are assuming that our application will run as a bare-metal bootloader, the firmware format should be a raw binary image, and should be loaded at a well-known address, that CPU will start at when power up. The QEMU help us to do that by specifying the `-bios <firmware>` option.

After a reset, the CPU start execute at ROM code `0x00000000`, if the `-bios <firmware>` option is used, QEMU assumes that we are running ROM code, and will load the firmware into Flash memory. The firmware, in this case, have to relocate itself into RAM, and do tasks (like the way [UBoot](https://github.com/ARM-software/u-boot/blob/master/arch/arm/lib/relocate_64.S#L22) do).

Another choice is using `-kernel <firmware>` option. Now the QEMU assumes that we are running a kernel, and the bootloader was running before, so it acts like bootloader, loads the firmware directly into RAM and start executing. We can specify our application entry point address in the firmware header, but it should be RAM beginning point, `0x40000000`.

So now we have two choices:

- Write ROM code and relocate our application into RAM.
- Write kernel code and let the QEMU loads our application into RAM.

> Using `-kernel <firmware>` option also has other advantage is that, QEMU acts like a bootloader and able to parse variety kind of kernel image format, for example ELF, PE, or even Linux Kernel format.
{: .prompt-info }

We will build 2 versions of firmware, `boot.elf` that is in ELF format, and can be run as a kernel. And the second `boot.bin` is ROM boot code version, that has the ability to self-relocate the application into RAM and start executing.

```bash
# Start from ROM (Flash memory).
qemu-system-aarch64 -M virt,virtualization=on,secure=on -cpu cortex-a57 -bios boot.bin -d int -D exceptions.log

# Loaded and start from RAM by QEMU.
qemu-system-aarch64 -M virt,virtualization=on,secure=on -cpu cortex-a57 -kernel boot.elf -d int -D exceptions.log
```

### 2.1. Debugging

By using QEMU, we have some advantages for debugging, the bare metal boot code may be harder, but not impossible. let's discover some ways.

#### 2.1.1. Using GDB

QEMU support using GDB as a remote target, where QEMU will start listen on a port (by default 127.0.0.1:1234) and GDB remote to start debugging. `-s` option make QEMU listen on that port and `-S` option make qemu not start the guest until you tell it to start from GDB.

When building the firmware, we should build along with the ELF version, that's very useful for debugging. We can check symbols and address with  `aarch64-none-linux-gnu-readelf` and also load debug symbol with GDB.

You can run either Aarch64 GDB `aarch64-none-linux-gnu-gdb` that is provided by ARM or generic version `gdb-multiarch`. You can specific `-ex` to exec previous commands, so you don't have to do this every time. For example, I load the symbol table from `boot.elf` and remote to QEMU server

```bash
aarch64-none-linux-gnu-gdb boot.elf -ex "target remote:1234"
```

NOTE that the symbol address in the binary do not match runtime address. The symbol addresses that you see in ELF file, might not what GDB see. For example, the ELF files define program start at the address `0x40000000`, and symbols are based on this address, but when the program running, we DIDN'T load the firmware to address `0x40000000`, instead of that we load them to ROM first (`0x00000000`), and then the code will perform self-relocating to `0x40000000`. So when the program is running at ROM, self-relocating code part will not match to the symbol address that GDB see. So you can not set break point with a label, for example `b _relocate`, GDB will map to it to `0x40000000` from ELF file, but in fact, this code is only be run in case we load the firmware to ROM, so symbol `_relocate` actually start from `0x00000000`. In that case, we have another solution, is that setting break point at a specific address.

For example, let's say we want to set break point at ROM code `copy_loop` label. We read ELF file:

```bash
$ ../toolchain/bin/aarch64-none-linux-gnu-readelf boot.elf -s | grep copy_loop
    15: 0000000040000010     0 NOTYPE  LOCAL  DEFAULT    1 copy_loop
```

This message show that the label `copy_loop` is located at address `0x0000000040000010`. But in fact, we know that this piece of code will run at `0x00000000` instead of `0x40000000`. So we have to set break point at address `0x10`:

```bash
aarch64-none-linux-gnu-gdb boot.elf -ex "target remote:1234"

(gdb) b *0x10
Breakpoint 1 at 0x10
(gdb) c
Continuing.

Breakpoint 1, 0x0000000000000010 in ?? ()
(gdb)
```

When we perform `c -- continue` the program will start at our point: `0x0000000000000010`.

We will discuss more in the ROM boot - Self Relocating section.

#### 2.1.2. Using QEMU log items

QEMU support logging specific items, using `-d <item>` option with the item, we want to log:

```text
Log items (comma separated):
out_asm         show generated host assembly code for each compiled TB
in_asm          show target assembly code for each compiled TB
op              show micro ops for each compiled TB
op_opt          show micro ops after optimization
op_ind          show micro ops before indirect lowering
int             show interrupts/exceptions in short format
exec            show trace before each executed TB (lots of logs)
cpu             show CPU registers before entering a TB (lots of logs)
fpu             include FPU registers in the 'cpu' logging
mmu             log MMU-related activities
pcall           x86 only: show protected mode far calls/returns/exceptions
cpu_reset       show CPU state before CPU resets
unimp           log unimplemented functionality
guest_errors    log when the guest OS does something invalid (eg accessing a
non-existent register)
page            dump pages at beginning of user mode emulation
nochain         do not chain compiled TBs so that "exec" and "cpu" show
complete traces
plugin          output from TCG plugins
strace          log every user-mode syscall, its input, and its result
tid             open a separate log file per thread; filename must contain '%d'
vpu             include VPU registers in the 'cpu' logging
trace:PATTERN   enable trace events

Use "-d trace:help" to get a list of trace events.
```

You can specify multiple `-d` options to select multiple kinds of log. Some useful option is `-d int` that show interrupts and exceptions in short format. We might use it a lot when changing Exception levels. Here is the log we might get:

```text
Exception return from AArch64 EL1 to AArch64 EL0 PC 0x190
Taking exception 2 [SVC] on CPU 0
...from EL0 to EL1
...with ESR 0x15/0x56000001
...with ELR 0x1bc
...to EL1 PC 0x40001400 PSTATE 0x3c5
```

The log is showing that We just perform jumping from EL1 to EL0 at the PC 0x190 entry point. And then take an exception 2 `svc` to jump back from EL0 to EL1. The `ESR` register is Exception syndrome register, that is showing the syndrome information for an exception, because we are in EL1, `ESR` here refer to `ESR_EL1`. Similarly, The `ELR` refer to `ELR_EL1` and that is the Exception Link register that hold the address when returning from current exception level. The `0x40001400`

Other options like `-D` help you to redirect these item logs into other output, for example, files `-D log.txt`.

#### 2.1.3. Logging with virt UART

Unlike the real hardware, maybe need more steps to setup UART, by using QEMU, we can write directly characters into UART Tx by writing to the Tx register. And the log by default will be redirected into stdout. As we know the UART0 peripheral start at memory address 0x09000000, here is the simple macro that I wrote in Aarch64 Assembly to write character directly to Tx register:

```assembly
.macro qemu_print reg, len
    sub sp, sp, #24
    str x10, [sp, #0]
    str x11, [sp, #8]
    str x12, [sp, #16]

    mov x10, #0x09000000
    mov x11, #\len
    ldr x12, =\reg
1:
    ldrb w2, [x12] /* Get the first byte and clear all the rest. */
    strb w2, [x10]
    cmp x11, #1
    beq 2f
    sub x11, x11, #1
    add x12, x12, #1
    b 1b
2:
    ldr x12, [sp, #16]
    ldr x11, [sp, #8]
    ldr x10, [sp, #0]
    add sp, sp, #24
.endm
```

This macro send byte to byte in a loop from the address `addr` with length `len` to the Tx register `0x09000000`. The `ldrb` get the first byte in the address that is hold by `x12`.
And here is how we use the `qemu_print`:

```assembly
qemu_print hello_message, hello_message_len

hello_message: .asciz "Aarch64 bare metal code!\n"
hello_message_len = . - hello_message
```

it's easier to write the print function with C:

```c
volatile unsigned int *const UART0DR = (unsigned int *)0x09000000;
void print_uart0(const char *s)
{
    while (*s != '\0')
    {                             
        *UART0DR = (unsigned int)(*s); 
        s++;                         
    }
}

void main()
{
    print_uart0("Hello world!\n");
}
```

But we're only using C in EL0 for user code.

### 2.2. Exception level with QEMU

By default, The QEMU assume that you want to emulate kernel code, so the EL default will be 1. To change the EL level to higher, we can specify options `virtualization=on`, the QEMU assume that we'll start running at EL2 Hypervisor layer, and with `secure=on`, QEMU assume that we want to run at EL3 Secure Monitor layer.

In the real hardware, after a reset, the processors will enter EL3 by default. So we will enable both features to emulate real system. The command to start QEMU might look like this:

```bash
qemu-system-aarch64 -M virt,virtualization=on,secure=on -cpu cortex-a57 -bios boot.bin -d int -D exceptions.log
```

## Checking current exception level

```assembly
_start:
    /* We don't know current EL yet, so we load a generic stack pointer. */
    ldr x30, =stack_top
    mov sp, x30

    qemu_print hello_message, hello_message_len

    /* Check current EL.*/
    mrs x1, CurrentEL
    cmp x1, 0
    beq in_el0

    cmp x1, 0b0100
    beq in_el1

    cmp x1, 0b1000
    beq in_el2

    cmp x1, 0b1100
    beq in_el3

    b .

in_el3:
    /* EL3 code. */

in_el2:
    /* EL2 code. */

in_el1:
    /* EL1 code. */

in_el0:
    /* EL0 code. */

```

## 4. Build system
