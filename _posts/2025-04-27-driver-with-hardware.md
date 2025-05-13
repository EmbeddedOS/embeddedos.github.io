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

Every device has its own specification, such as register addresses, interrupt line. Some devices cannot expose their info themselves. Now we have 2 choices to let kernel know about them. The first solution, hardcoding the device's information inside drivers, 

To understand the historic reasons, let's get an example, assume that you have 2 similar boards that are only different in some GPIO registers

#### 2.1.1. DeviceTree

#### 2.1.2. ACPI

### 2.2. Bus enumeration

Peripherals are connected to the processor via a Bus. Some buses have supported device enumeration (discovery). The Bus drivers detect new devices automatically.

## 3. Hardware interaction

### 3.1. Memory-mapped I/O

### 3.2. Port-mapped I/O

### 3.3. Interrupt

### 3.4. Polling

### 3.5. DMA

### 3.6. 
