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

## An introduction to kprobes

Kprobes (Kernel probes) is a mechanism that allows you to insert probes (breakpoints) at almost any instructions in the kernel and executes custom handler routines when those probes are hit.

How does it work? when a kprobe is registered, Kprobes make a copy of the probed instruction and replace the first byte(s) with breakpoint instructions. When the CPU hits the breakpoint instruction, a trap occurs, CPU's registers are saved and the control passes to Kprobes.

 Kprobes executes the *pre_handler* associated with the kprobe, passing the handler the addresses of the `kprobe struct` and the saved registers.

 Next, Kprobes single-steps its copy of the probed instruction.

 After the instruction is single-stepped, Kprobes executes the *post_handler*, if any, that is associated with the kprobe. Execution then continues with the instruction following the probepoint.

## Kprobes changing execution path

