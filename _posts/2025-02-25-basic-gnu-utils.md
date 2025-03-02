---
title: "GNU core utilities: Linker and Assembler."
description: >-
  Understand GNU utilities linker and assembler basic concepts. Also a quick view about ARM instruction set

author: Cong
date: 2025-02-25 00:01:00 +0800
categories: [Compiling]
tags: [linker, arm, assembler, assembly, gnu]
image:
  path: assets/img/gnu_coreutils.png
  alt: GNU core utils.
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

Every CPU architecture have its own assembly instruction set to work with it. That also requires its own Assembler to produce these instruction into machine code. Although they look quite similar, there is no single syntax for assembly language.

The GNU Assembler is highly portable re-configurable assembler, it uses a *simple, general syntax that works for a wide variety of architectures*.

> The GNU Assembler executable is named `as`. Some architectures name their assembler with `-as` as a suffix. For instance: `aarch64-none-linux-gnu-as`.
{: .prompt-info }

### 1.1. Assembly program structure

First, let's take a look to a simple "Hello World" program:

```asm
        .data
str:    .asciz "Hello World\n" @ Define a null-terminated string

        .text
        .globl main
        /∗ This is the beginning of the main() function.
           It will print "Hello World" and then return.
         ∗/
main:   stmfd sp!,{lr}      @ push return address onto stack
        ldr r0, =str        @ load pointer to format string
        bl printf           @ printf("Hello World\n");
        mov r0, #0          @ move return code into r0
        ldmfd sp!,{lr}      @ pop return address from stack
        mov pc, lr          @ return from main
```

An assembly program include 4 basic elements:

1. Labels -- Is used to refer to the address of data, function, or block of code. There're 2 labels in the program: `main` and `str`. The `str` holds the start address of string "Hello World\n" and section `.asciz`. The `main` label holds the address of program' entry point. You can imagine the label working like a pointer in C.
2. Assembler directive -- Is used to define symbol, allocate storage, and control the behavior of the assembler.
3. Assembly instruction -- Program statements that will be executed on the CPU.
4. Comments -- There are two basic comment styles: multi-line and single. Where multi-line syntax is similar like C style: `/* comments. */`. The single comment syntax might depend on the architecture. For example, ARM start a single comment with `@` character.

### 1.2. Assembly directives

The GNU assembler supports a relatively large set of directives, that are used in a generic way for all supported architecture. Let's discover some of most commonly used directives.

> Every directive start with `.` character.
{: .prompt-info }

#### 1.2.1. Selecting sections

Every instructions, data, comments are stored in *sections*. There are some standard sections the programmer **CAN** choose to put code or data in. Sections also can be divided into numbered subsections.

- `.<section>` -- to select a section.
- `.section <name>`  -- create a new section.
- `.<section> subsection` -- Create a new subsection.

```asm
.section my_section         @ Create my_section.
/* Put code and/or data. */

.text my_text_subsection    @ Create my_text_subsection under text section.
/* Put code and/or data. */
```

Once a section has been selected, all of the instructions and/or data will go into that section until another section is selected.

#### 1.2.2. Allocating space

The assembler supports bytes, integer types, floating point types, and strings. There are several directives to allocate a fixed amount of space in memory. The syntax to allocate like this `.<type> expressions`

```asm
.data                           @ select `data` section.
i:      .word   0               @ make a label `i` that point to a memory space that enough for int.
str:    .asciz  "Hello\n"       @ String.
arr:    .word   0,1,2,3         @ Array of integer.
```

This program might look like this in C:

```c
static int i = 0;
static char str[] = "Hello\n";
static int arr[] = {0, 1, 2, 3};
```

#### 1.2.3. Filling and aligning

On almost CPU architecture, data can be moved to and from 1 or 2 or 4 bytes at a time. Moving a word (4 byte) will take significantly more time if the address is not aligned on a 4-byte boundary. That's similar for moving half-word. By that and some other use cases, GNU Assembler provide some directive to align memory on any boundary desired.

`.align abs-expr, abs-expr, abs-expr`

Here is an simple example, that `.align` directive tell the assembler that omits next 2 bytes after `ch`, to make sure access `arr` will be at 4 byte boundary:

```asm
ch: .byte 'A','B'
.align 2
arr: .word 0, 1, 2, 3, 4
```

`.skip size, fill`
`.space size, fill`

These directive allocate a large area of memory and fill it with the same value.

#### 1.2.4. Symbol operations

#### 1.2.5. Conditional

#### 1.2.6. Macros

The directives `.macro` and `.endm` allow the programmer to define macros that the assembler expands to generate assembly code. The syntax to define a macro look like this:

`.macro macro_name macro_args ...` -- Define macro called `macro_name`, if the macro requires arguments, their name
are specified after the name `macro_args ...`. The arguments can also have a default value `macro_args=default_value`. When a macro is called, the argument values can be specified either by position, or by keyword. Let's take a look to this example:

```asm
.macro SHIFT a,b
.if \b < 0
mov \a, \a, asr #-\b
.else
mov \a, \a, lsl #\b
.endm
```

To refer to the argument, we start with `\`, for example `\a`, `\b`. There is a special variable `\@` that is used by the assembler to maintain a count of how many macros it has executed.
To end a macro definition, we can use `.endm` macro, or use `.exitm` to exit early.

Calling macro:

```asm
SHIFT r1, 3
SHIFT r4, -6
```

The code will be generated:

```asm
mov r1, r1, asr #3
mov r4, r4, lsl #6
```

#### 1.2.7. Includes

### 1.3. ARM instruction set

#### 1.3.1. Load/Store

#### 1.3.2. Branch

- `bl` branch to other instruction, the `LR` will load the address of next instruction after `bl`, using this indicate you will return in the future.
- `b` branch without return, you just want to jump to other instruction.

## 2. GNU Linker

The linker combines a number of object and archive files, relocates their data and ties up symbol references. This usually is the last step in compiling a program. Similar as GNU Assembler, GNU Linker is the most popular than other linkers. It covers a broad range of situations, and be as compatible as possible with other linkers.

> The GNU Linker executable is named `ld`. Some architectures name their linker with `-ld` as a suffix. For instance: `aaarch64-none-linux-gnu-ld`.
{: .prompt-info }

### 1.4. Inline Assembly

```asm

```

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

## 3. Other utilities

### 3.1. objcopy

- <https://ee209-2019-spring.github.io/references/gnu-assembler.pdf>
- <https://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf>
