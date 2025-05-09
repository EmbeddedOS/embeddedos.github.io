---
title: "The Linux device driver part I: hardware interaction."
description: >-
  Understand possible ways kernel interact with hardware.

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [kernel, device driver]
tags: [Linux, kernel, device driver model, hardware interaction]
published: false
---

## 1. Objective

Understand every possible way the kernel communicates with hardware.

## 2. Hardware identification

As generic as possible, no hard code, reuse kernel but we still need some way to tell kernel about un-discov

### 2.1. Hardware description

Don't support enumeration.

### 2.2. dynamically

Peripherals are connected to the processor via a Bus. Some buses have supported device enumration (discovery). The Bus drivers detect new devices automatically.

## 3. Hardware interaction

### 3.1. Memory-mapped I/O

### 3.2. Port-mapped I/O

### 3.3. Interrupt

### 3.4. Polling

### 3.5. DMA

### 3.6. 
