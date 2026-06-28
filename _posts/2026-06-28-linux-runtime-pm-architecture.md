---
title: "Linux System & Runtime Power Management"
description: >-
  Let's dive deep into the power management framework inside the linux kernel

author: Cong
date: 2025-06-28 00:01:00 +0800
categories: [kernel, pm]
tags: [linux, kernel, runtime, pm]
image:
  path: assets/img/invisible_process.png
  alt: Linux Power Management
published: false
---

Power Management in Linux is a big subsystem that includes so many subdomains: supported hardware, CPUIdle, CPUFreq governors, DVFS, thermal, and so on. In this blog, we are going to discuss the System Wide and the Runtime Power Management frameworks, how device drivers should handle 

## 1. The key concept

### 1.1. `struct dev_pm_ops`

The structure is a key part of device-driver and power management. That's embedded into exist `struct device_driver`, `struct bus_type`, ...

```c
struct dev_pm_ops {
    int (*prepare)(struct device *dev);
    void (*complete)(struct device *dev);
    int (*suspend)(struct device *dev);
    int (*resume)(struct device *dev);
    int (*freeze)(struct device *dev);
    // ...
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```



What happen when you do a system suspend

```mermaid
sequenceDiagram
    participant PL as Platform Specific
    participant DEV as Per deivce
    Note over PL: struct platform_suspend_ops
    Note over DEV: struct dev_pm_ops
    PL->>PL: begin()
    PL->>DEV:
    DEV->>DEV: prepare()
    DEV->>DEV: suspend()
    DEV->>PL:
    PL->>PL: prepare()
    PL->>DEV:
    DEV->>DEV: suspend_late()
    DEV->>DEV: suspend_noirq()
    DEV->>PL:
    PL->>PL: enter()
    Note over PL: Suspended ...
    PL->>PL: wake()
    PL->>DEV:
    DEV->>DEV: resume_noirq()
    DEV->>DEV: resume_early()
    DEV->>PL:
    PL->>PL: finish()
    PL->>DEV:
    DEV->>DEV: resume()
    DEV->>DEV: complete()
    DEV->>PL:
    PL->>PL: end()

```

## 1. System power management

## 2. Runtime power management

## 3. Core PM

## 3. Userspace expotion