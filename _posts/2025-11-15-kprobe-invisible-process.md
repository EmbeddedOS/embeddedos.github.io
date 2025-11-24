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

## Kernel probes and return probes

Kprobes enables you to dynamically break into any kernel routine, collect information non-disruptively. You can trap at almost any kernel code address, specifying a handler routine to be invoked when the break point is hit.

There are two types of probes:

1. kprobes - can be inserted on virtually any instruction in the kernel.
2. kretprobes - return probe fires when a specified function returns.

## How does it work?