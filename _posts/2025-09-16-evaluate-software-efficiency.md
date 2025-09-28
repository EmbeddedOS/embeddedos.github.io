---
title: "Software Observability: Answer The Age-Old Question — How Efficient Are Your Programs?"
description: >-
  An introduction to software efficiency and evaluating methods.

author: Cong
date: 2025-09-16 00:01:00 +0800
categories: [low-latency, observability]
tags: [Low Latency, OOP, observability]
image:
  path: assets/img/rootfs.jpeg
  alt: evaluate-sw-overview.
published: false
---

## 1. Let's talk efficiency

In software development, using qualitative words is not always a good choice. Sayings such as *it's awesome, I promise it will run very fast*, *it's very efficient and stable, high performance*, etc. That sounds good, but in the end, what does that actually mean? or how we prove that? Your customers, recruiters, or managers can't count on those things to decide to buy your products, hiring, or promote you. Remember the XYZ format in resume writing, what we got: Accomplished [X] as measured by [Y] by doing [Z] Format. They need the real things! the thing that they can measure and compare to others and that is the target of this article.

By measuring your programs, you do know is this actually effective? does your program run fast, and the most important your effort can be count! for example, now you can report to your boss, *hey men, I do optimize functions and now our business run faster 20%!* Big change, right? OK, let's go to the main topic.

There are two categories of efficiency that can be measured in software development: *performance* and *resource*.

### 1.1. Performance efficiency

The performance 

- Latency
- Throughput
- Scalability

At system level:

- Kernel interaction: syscall rates, page faults, interrupts.
- context switching cost.

### 1.2. Resource efficiency

- CPU.
- Memory.
- I/O.
- Network, bandwidth usage.
- Energy.


## 2. Evaluating methods 

- logging -> basic element, export the info you measuring in an effective way, maintainability.
- measuring -> efficiency.
  - Micro benchmarking.
- observing, tracing -> observability.
- testing -> stability.
