---
title: "The Linux device model"
description: >-
  Understand the linux device model.

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [kernel, device driver]
tags: [Linux, kernel, device driver model, kobject]
published: false
---

## Objective

- Understand the linux device model.

## 1. The reason for Linux Device model

- Kernel versions that before 2.5 had no single data structure to obtain information is put together.
- Some complicated features such as power management need to know the structure of the system.

One of the goals for 2.5 was the creation of a **unified device model**. What this model provides are:

- Power management & system shutdown -- These require an understanding of system's structure. For example, some devices need to be shutdown in order, or need to notice to others before shutdown, for example, USB host need to notice USB devices before shuting down. *The device model enables a traversal of the system's hardware in the right order*.
- Communication with user space -- sysfs virtual filesystem is tightly tied into the device model and exposes the struct represented by it.
- Hot-pluggable devices -- Device model manages the hotplug mechanism inside kernel and communicate with user space about plugging and unplugging of devices.
- Device classes -- What kinds of devices are available. This model includes a mechanism for assigning devices to classes, which describe those devices at a higher, functional level and allow them to be discovered from user-space.
- Object lifecycle -- these functions above make creation and manipulation objects within kernel more complicated. The device model provides a set of mechanisms for dealing with object: lifecycle, communicate bw objects, inheritance, and representation in user space.

## 2. Kobject

The `kobject` was first introduced in kernel 2.5, it was initially meant as a simple way of unifying kernel code which manages reference counted objects. But now it's the glue that holds much of the device model and its sysfs interface together.

```c
struct kobject {
  const char                *name;
  struct list_head          entry;
  struct kobject            *parent;
  struct kset               *kset;
  const struct kobj_type    *ktype;
  struct kernfs_node        *sd; /* sysfs directory entry */
  struct kref               kref;

  unsigned int state_initialized:1;
  unsigned int state_in_sysfs:1;
  unsigned int state_add_uevent_sent:1;
  unsigned int state_remove_uevent_sent:1;
  unsigned int uevent_suppress:1;
};
```

A `Kobject` is usually embedded within other structure which contains main stuffs. For example, a `struct cdev` that present a character device:

```c
struct cdev {
    struct kobject kobj;
    struct module *owner;
    struct file_operations *ops;
    struct list_head list;
};
```

```c
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype);
```
