---
title: "Generic kernel init code and init call level"
description: >-
  Understand the linux kernel main start up flow after the architecture initialization.

author: Cong
date: 2025-02-16 00:01:00 +0800
categories: [Kernel, generic init]
tags: [Kernel, Booting, initcall, generic init code]
image:
  path: assets/img/linux_generic_init.png
  alt: linux_generic_init.
---

## 1. The Architecture init code

The `arch/<arch>` folder in the kernel source tree contains architecture-specific code. There are so many architectures and each has its own way to boot up system. Generally, if you look into each arch folder, you will see the different ways to setup system. Let's get some examples:

- The `arch/x86` contains setting up code for x86 architecture. A lot of code is written by using x86 Assembly instruction sets that only can run on x86 CPU families. The code do x86-specific things like setup protected mode, long mode, enable A20, video mode and soft of things like that. For more details about what kernel do on x86, I actually have other blog about x86 Bootloader, take a look [x86 Bootloader](/posts/Build-A-x86-Bootloader-BIOS/).
- The `arch/arm64` contains specific code for AArch64, I have separate blog that discuss about it [Kernel booting with AArch64](/posts/Linux-Kernel-booting-with-Aarch64/).

But after all, the kernel is generic, a lot of stuff are generic for all architecture. For example, even if your SoC has its own timer peripheral, you don’t need to write separate schedulers for each type of SoC. Instead, you can create a generic timer interface that the scheduler code can use without being aware of the underlying architecture. Another example: sys_call_table.

 we cannot write everything duplicately for each architecture. so we have to define a generic way to setup system before finishing the architecture init code. Such as jump to the `start_kernel()` symbol.

## 2. The start_kernel() function

The `start_kernel()` function that is located in `init/main.c`, is the entry point to the kernel generic code. This function complete the initialization of the Linux kernel, nearly every component is initialized by this function.

### 2.1. The parameters

Unlike normal functions where parameters is passed via C function's parameters style, the kernel parameters need to passed from the arch boot code. So we need to define a generic way for all architecture. Kernel do that by calling a generic symbol `setup_arch()`
