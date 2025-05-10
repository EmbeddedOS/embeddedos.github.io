---
title: "The Linux device driver part I: hardware interaction."
description: >-
  How kernel identifies and interacts with hardware.

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [kernel, device driver]
tags: [Linux, kernel, device driver model, hardware interaction]
published: false
---

## 1. Objective

- Understand the ways kernel identifies devices: device descriptions and Bus enumerations.
- Understand the ways kernel interacts with hardware: memory map, interrupt, DMA, etc.

## 2. Hardware identifications

To identify devices, kernel must know about the devices that're connected to it. Some devices are detected automatically by their bus drivers, e.g., USB, PCU. Kernel uses bus enumeration to detect and identify them. And for those that are completely silent like UART, GPIO, etc. Kernel uses device descriptions to determine them. Let's explore both ways.

### 2.1. Device descriptions

Every device has its own specification, such as registers, memory addresses, interrupt line. Some devices cannot expose their info themselves to the kernel. So to let kernel notice about those devices, we might have 2 choices. First one is defining the device's information inside drivers, kernel itself. It's easy when develop drivers with hardcoding device information like registers, etc. But this way causes more issues, the kernel become specific, each hardware version, require specific kernel firmware. Every time a device trait changes, the kernel needs to be rebuilt. That brings us to the second choice: Separate the device description outside the kernel. Kernel now becomes more generic, independent with the hardware, and only needs to implement the device description parsers to archive the device information.

Some architectures have their own ways to describe their hardwares. ARM, PowerPC, or embedded devices prefer using DeviceTree, meanwhile, computer systems, like x86, prefer using ACPI table.

#### 2.1.1. DeviceTree

#### 2.1.2. ACPI

### 2.2. Bus enumeration

Peripherals are connected to the processor via a Bus. Some buses have supported device enumeration (discovery). The Bus drivers detect new devices automatically.

## 3. Hardware interaction

Assume that the kernel identifed its devices, now, how can the kernel communicate with them?

### 3.1. Memory-mapped I/O

### 3.2. Port-mapped I/O

### 3.3. Interrupt

### 3.4. Polling

### 3.5. DMA

### 3.6. 
