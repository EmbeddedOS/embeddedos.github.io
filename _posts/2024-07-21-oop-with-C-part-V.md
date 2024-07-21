---
title: 'Object Oriented Programming in C part V: Polymorphism'
description: >-
    Polymorphism in OOP enables different objects to be treated as instances of a shared base type. This allows objects to be manipulated in a uniform manner without regard to their specific object type.

author: Cong
date: 2024-07-21 5:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
pin: true
---

## 1. Polymorphism

Polymorphism is one of the most Important principles of object-oriented programming. `Poly` means many, and `morphism` means forms.  Essentially, through an interface, we can offer a variety of services.

### 1.1. Examples of polymorphism in the real world

Let take some examples of polymorphism in the real-world:

- Animals are great real life example of polymorphism, if we ask different animals to speak, they respond in their own way. Like if we ask Dog to speak it will `bark`, similarly, cow will `moo`, cat will `meow`. So the same action of speaking is performed in different ways by different animals exhibiting polymorphism.
- Another excellent real time example of polymorphism is your smartphone.The smartphone can act as phone, camera, music player and what not, taking different forms and hence polymorphism.

### 1.2. Examples of polymorphism in Linux Programming

Linux Architecture is an excellent example of polymorphism:

- **The Universal I/O Model**: All system calls for performing I/O refer to open files using a `file descriptor`, a (usually small) nonnegative integer. `File descriptors` are used to refer to all types of open files, including pipes, FIFOs, sockets, terminals, devices, and regular files. That means a `file descriptor` is simply a nonnegative integer that can have many forms, represent various entities, such as files, devices, sockets, and numerous other resources in Linux.

- **Linux Kernel Device Driver**: Device Driver Model in Linux using the same interfaces (file operations) for devices. For example, with the same `read()` system call,You can read data from a temperature sensor or simply read from your keyboard.

> In C++, there is another concept of polymorphism known as **Compile-Time Polymorphism (Static Binding)**, which involves function overloading—using the same function name but with different parameters. We won't analyze it in this topic.
{: .prompt-info }

## 2. Abstract Interface

An abstract interface in OOP defines a contract for classes that inherit from it, specifying methods that must be implemented by subclasses but not providing an implementation itself. This allows for polymorphic behavior without specifying how each subclass should behave. Key aspects include.

## 3. Many forms of object

## 4. Summary
