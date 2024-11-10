---
title: 'QEMU Object Module: QEMU framework with OOP style.'
description: >-
  Let's discover the QEMU Object Module and how it provides a framework for registering user creatable types using an OOP approach.
  
author: Cong
date: 2024-11-14 00:00:00 +0800
categories: [QEMU, QOM]
tags: [QEMU, QOM, OOP]
image:
  path: assets/img/qemu_logo.jpg
  alt: QEMU.
---

## 1. QEMU Object Model

QEMU Object Model (QOM) is a framework that QEMU provides to register user creatable type.

## 3. The QOM Tree

## 2. The QOM Class

To create a new QOM class, QEMU provided APIs and a struct `TypeInfo` to describe your class.

```c
struct TypeInfo
{
    const char *name;
    const char *parent;

    size_t instance_size;
    size_t instance_align;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;
    size_t class_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void *class_data;

    InterfaceInfo *interfaces;
};
```

## 3. The QOM Object
