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

> The GNU Assembler executable is named `as`. Some architectures name their assembler with `-as` as a suffix. For instance: `arm-none-eabi-as`.
{: .prompt-info }

### 1.1. Assembler directives

## 2. GNU Linker

- <https://ee209-2019-spring.github.io/references/gnu-assembler.pdf>
- <https://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf>
