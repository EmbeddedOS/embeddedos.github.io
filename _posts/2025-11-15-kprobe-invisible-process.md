---
title: "Hiding processes from the OS with Kprobes"
description: >-
  Using Kprobes to make processes completely invisible, unkillable from the OS.

author: Cong
date: 2025-11-15 00:01:00 +0800
categories: [kernel, probe]
tags: [linux, kernel, invisible, unkillable, process, proc, kprobe, kretprobe]
image:
  path: assets/img/invisible_process.png
  alt: invisible process
published: true
---

Kprobes is a powerful feature in the kernel. In this blog, we will practice it by hooking into kernel points, change the kernel execution path to hide our process completely. Before we start, since the register set is architecture specific and different kernel versions, configs have different system maps, symbol addresses, etc. This is important to note about our target system:

- kernel version: 
- target arch: arm64
- board: qemu-aarch64-system virt.
- kernel config: default + 

## An introduction to kprobes

Kprobes (Kernel probes) is a mechanism that allows you to insert probes (breakpoints) at almost any instructions in the kernel and executes custom handler routines when those probes are hit.

How does it work? when a kprobe is registered, Kprobes make a copy of the probed instruction and replace the first byte(s) with breakpoint instructions. When the CPU hits the breakpoint instruction, a trap occurs, CPU's registers are saved and the control passes to Kprobes. It executes the *pre_handler* associated with the kprobe, passing the handler the addresses of the `kprobe struct` and the saved registers. Next, Kprobes single-steps its copy of the probed instruction. After the instruction is single-stepped, Kprobes executes the *post_handler*, if any. Execution then continues with the instruction following the probepoint.

Image here.

Since Kprobes can probe into kernel running code, it can change the register set, even instruction pointer `pc` register. One note if you do that, you should return non zero value from the *pre_handler*, so Kprobes will bypass the single step and just return to the given address.

## How a process information is exposed in user space?

Process information is primarily exposed to user space via a virtual file system called procfs and mounted at `/proc` (by default). Procfs provides a dynamic interface to kernel data structure to get system information or change kernel parameters (you can change kernel parameters at runtime (or we called tuning) by using `sysctl` via this interface but this is not what we're gonna discuss in this blog).

The process information details are under the dynamic folder `/proc/[pid]`:

- cmdline: The command-line arguments used to start the process.
- status: Detailed status information about the process.
- exe: A symbolic link to the executable file of the process.
- cwd: A symbolic link to the current working directory of the process.

User space tools such as `ps`, `top`, `free` check for this virtual fs and get system + processes information.

We call procfs as virtual since it's not actually existing on the hard disk. Every time you use system call to access them kernel redirect it to different part with normal regular directories of files.

Draw an image here to compare how you `cat` a `/proc/[pid]/status` file compare with when you `cat` a regular file.

## Hook into kernel proc vfs symbol

## Make the process unkillable from user space

## Testing 

