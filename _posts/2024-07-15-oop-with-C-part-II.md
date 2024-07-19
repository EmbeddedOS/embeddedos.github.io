---
title: 'Object Oriented Programming in C part II: Abstraction'
description: >-
 Abstraction in software design hides complex implementations, large systems, and providing simple interfaces to users, making it a key concept. Let's see how we achieve this in C.

author: Cong
date: 2024-07-15 7:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
pin: true
---

## 1. Introduction

Abstraction refers to the the concept of hiding the complex implementation details of a system, and exposing only the essential features of the object.

### 1.1. Abstraction in real world example

Let get an example of abstraction in Internet System. Assume that you want to access to the `facebook.com` website. You simply typing it in your browser. But behind the scene are combination of massive systems.

1. The browser query to the DNS server in your location to get the IP address of the `facebook.com` website.
2. The browser perform sending requests to the `facebook.com` domain's IP address.
3. As you know, an IP address represents a host machine on the network. However, to handle requests from millions of users per second, Facebook needs to deploy a large-scale system infrastructure comprising many servers. To provide a simple interface to users in a region, Facebook deploys load-balancer systems. These systems dispatch requests to multiple servers behind the scenes, all accessible via a single address.

In this example, abstraction is how your browser abstracts **How to get resources from a website** and how Facebook abstracts **How to access Facebook servers**.

### 1.2. Abstraction in C example

A typical example of abstraction in C is system calls, which are interfaces provided by the kernel for user space to access its services. Take this example: using the `open()` and `write()` system calls, you tell the kernel that you want to write content to a file, and the kernel handles the writing process.

```c
int fd = open(FILE_PATH, O_WRONLY);
int res = write(fd, "Your data", 9);
```

## 2. Abstraction with Opaque Object

### 2.1. The Opaque Type

The opaque Object Pattern, based on the C opaque type feature, revolves around a data type whose concrete data structure is not defined in its interface. With this pattern, you cannot create objects directly, but you can hold a pointer to them (known as an `opaque pointer`). Users can interact with APIs that accept these opaque pointers as parameters, enabling functionality while keeping the underlying implementation hidden.

```c
struct opaque_object_t;
int opaque_object_do_something(struct opaque_object_t *obj, int params);
```

### 2.2. Abstraction with Opaque Object

Back to the previous `person object` example in the [OOP: Object blog](2024-07-14-oop-with-C-part-I.md). But in this time, we don't want to expose all the `person struct`, only expose necessary APIs. Here is the way we do.

In the header file (the interface that we provide):

```c
typedef struct person
```
