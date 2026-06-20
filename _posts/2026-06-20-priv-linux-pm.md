---
title: "Linux Power Management"
description: >-
  Power management in Linux from hardware to user space

author: Cong
date: 2025-11-15 00:01:00 +0800
categories: [kernel, pm]
tags: [linux, kernel, pm]
image:
  path: assets/img/invisible_process.png
  alt: USB gadget driver
published: false
---

## Power management in a computing system

Three major components:

- Hardware: responsbile for providing configuration interfaces ("knobs") so the OS can properly manage the power consumption.
- Kernel: via device drivers and subsystems, it is responsible for exposing power management abstractions to user space applications.
- User: Via abstractions provided by the kernel, they are responsible for implementing a policy for power management.

## The role of hardware

The hardware is the foundation upon which sw based pm strategies are built, so an energy-efficient hw design is a very important strategy to reduce power consumption.

- Select inherently energy-efficient hw components.
- Select hw components that provide configuration interfaces for PM.
- Design circuits that minimize electrical losses and reduce power consumption.

## What is power?

P = I * V

## Power management techniques

Every power management technique can be put into 2 major categories: *idle* and *active* pm.

- An idle pm technique will help to save the power when the system is not doing any work (shutting down some hardware components) -> trade off between power reduction and latency.
- An active power management technique will help to save power when the system is doing some work (e.g: scaling down the CPU frequently). -> trade off bw power reduction and performance.

##