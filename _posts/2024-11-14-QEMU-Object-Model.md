---
title: "QEMU Object Module: a QEMU's Framework with OOP style"
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

### 2.1. `TypeInfo` struct

To create a new QOM class, QEMU provided APIs and a `TypeInfo` struct to describe your class. The `TypeInfo` describe information about the type including what its parent, constructor and destructor hooks.

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

Some of important fields:

- `name` The type name.
- `parent` The parent type name.
- `instance_init` Object constructor, initialize its own members.
- `instance_finalize` Object destructor, destroy and free its own members.
- `class_init` Class initializer. That is called after parent types were initialized, this can be used to override parents's virtual method.
- `class_data` The hook data that is passed to the `class_init`.

The APIs to register your type:

```c
Type type_register(const TypeInfo *info);

Type type_register_static(const TypeInfo *info);

void type_register_static_array(const TypeInfo *infos, int nr_infos);
```

### 2.2. `type_init()` macro

Similar like Linux Kernel Module, you need a way to tell where to start your module. The QEMU provide the `type_init()` macro, that automatically call you function when module is loaded. Behind the scene, we have GCC extension attribute `__attribute__((constructor))`, that make your function is called automatically before `main()`

```c
#define type_init(function) module_init(function, MODULE_INIT_QOM)

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
```

The `register_module_init()` function allocates your type init function and append to the queue's tail.

```c
void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

## 3. The QOM Object
