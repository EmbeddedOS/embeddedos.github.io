---
title: "GNU utilities: Linker and Assembler."
description: >-
  Understand GNU utilities linker and assembler basic concepts.

author: Cong
date: 2025-02-25 00:01:00 +0800
categories: [Compiling]
tags: [linker, arm, assembler, assembly, gnu]
image:
  path: assets/img/kernel_image_format.png
  alt: Linux Kernel Image format.
published: false
---

It's so important to understand the GNU utilities and toolchains in Embedded System. While the assembler allows us to write the assembly code and compile to machine code, the linker gives us the ability to link objects to become the final executable image. This blog's targets will be:

- Understand the GNU Assembler basic concepts and the GNU Assembly syntax.
- Understand the GNU Linker basic concepts and how the linker merges all entities becoming an executable image.

## 1. GNU Assembler

There are so many high level languages C/C++, Java, Python that allows programmers to work at a high level of abstraction, so they don't need to understand exactly what instructions are used by a particular CPU. Compiled languages such as C and C++, a compiler translates high-level language into a assembly language for the particular CPU. The assembler then converts the assembly code into the binary codes, that the CPU can reads as instructions.

There are some situations, writing a assembly code directly might be better than using a high-level language.

- First steps of booting system.
- Handle exceptions & interrupts.
- low-level synchronization and locking code in multi thread environments.
- There's no compiler or the code should be optimized more than by the compiler.
- Low level access to processor features.

Every CPU architecture have its own assembly instruction set to work with it. That also requires its own Assembler to produce these instruction into machine code.

> The GNU Assembler executable is named `as`. Some architectures name their assembler with `-as` as a suffix. For instance: `aarch64-none-linux-gnu-as`.
{: .prompt-info }

### 1.1. Assembler directives

## 2. GNU Linker

The linker combines a number of object and archive files, relocates their data and ties up symbol references. This usually is the last step in compiling a program. Similar as GNU Assembler, GNU Linker is the most popular than other linkers. It covers a broad range of situations, and be as compatible as possible with other linkers.

> The GNU Linker executable is named `ld`. Some architectures name their linker with `-ld` as a suffix. For instance: `aaarch64-none-linux-gnu-ld`.
{: .prompt-info }

### 2.1. Command line options

Some common options:

- `-T file` -- Use`file` as a linker script.
- `-e entry` -- Use `entry` label address as the entry point of your program.
- `-static` -- Do not link against shared libraries.
- `-Map=file` -- Print a link map to `file`.
- `-nostdlib` -- Only search directories explicitly specified on the command line. Lib directories specified in linker scripts are ignored.
- `--oformat=format` -- Specify your output binary `format` instead of default one.
- `-shared` -- Create a shared library.
- `--sysroot=directory` -- Use `directory` as the location of the sysroot instead of default one.

> `sysroot` folder -- The compilation toolchains (include compiler, assembler, linker, and other utilities) need some default folder to lookup dependencies: libraries, headers, etc. This is specified by default when you install compiler. But sometimes, you want to change this folder to build custom program that is intended to run in other rootfs.
{: .prompt-info }
> List all binary output formats supported by your linker use: `objdump -i`. If you are using particular linker, using particular `objdump` also. For example: `aaarch64-none-linux-gnu-objdump -i`.
{: .prompt-info }

### 2.2. Linker script

This script is written in the linker command language. The main purpose of this script is to describe how the sections in the input files should be mapped into the output file, and to control the memory layout of the output file.

- <https://ee209-2019-spring.github.io/references/gnu-assembler.pdf>
- <https://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf>
