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

If you are thinking of things in object-oriented terms, kobjects can be seen as a top level, abstract class from which other classes are derived.

Now let's say you have a pointer of base class, all the abstract APIs use this pointer as a generic interface, how can you get the pointer to the derived class? this isn't C++ where you can cast pointers upward or downward. Also you should avoid using some tricks like assuming the kobject is at the beginning of the structure. There is a solution for that, using the `container_of()` macro. So, for example, get `cdev` pointer:

```c
struct cdev *device = container_of(kp, struct cdev, kobj);
```

### 2.1. Initialization

The kobject constructor:

```c
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype);
```

Every object MUST have a type, the `ktype` is required for a kobject to be created properly, as every kobject must have an associated `kobj_type`. The `ktype` controls what happens to the kobject when it is created and destroyed.

```c
struct kobj_type {
  void (*release)(struct kobject *kobj);
  const struct sysfs_ops *sysfs_ops;
  const struct attribute_group **default_groups;
  const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
  const void *(*namespace)(const struct kobject *kobj);
  void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

To register the kobject with sysfs, the function `kobject_add()` must be called:

```c
__printf(3, 4) __must_check int kobject_add(struct kobject *kobj,
                        struct kobject *parent,
                        const char *fmt, ...);
```

This function also sets up the parent of the object, and the name for the kobject. There is a helper for do both init and add:

```c
int kobject_init_and_add(struct kobject *kobj, const struct kobj_type *ktype,
                         struct kobject *parent, const char *fmt, ...);
```

### 2.2. Kobject user event

Let's say after a kobject has been registered with the kobject core, you need to announce to the world that it has been created you can do this by calling `kobject_uevent()` with `KOBJ_ADD` action. The `kobject_uevent()` function notifies userspace that kobject performed some actions.

```c
enum kobject_action {
    KOBJ_ADD,
    KOBJ_REMOVE,
    KOBJ_CHANGE,
    KOBJ_MOVE,
    KOBJ_ONLINE,
    KOBJ_OFFLINE,
    KOBJ_BIND,
    KOBJ_UNBIND,
};

int kobject_uevent(struct kobject *kobj, enum kobject_action action);
```

### 2.3. Reference count

Similar like shared-pointer, there's a way to track the object existing in the kernel. There're 2 low-level functions for manipulating the reference counter:

```c
struct kobject kobject_get(struct kobject kobj);
void kobject_put(struct kobject *kobj);
```

### 2.4. Object attributes

To create simple attributes and associated with the kobject:

```c
int sysfs_create_file(struct kobject *kobj, const struct attribute *attr);
```

Or with group:

```c
int sysfs_create_group(struct kobject *kobj, const struct attribute_group *grp);
```

### 2.3. Destruction

The code which created the object generally does not know when the reference counter reaches zero. the last call to `kobject_put()` will do destroying object, and the destructor is nothing but the `release` method of `kobj_type`.

```c
void my_object_release(struct kobject *kobj)
{
    struct my_object *mine = container_of(kobj, struct my_object, kobj);

    /* Perform any additional cleanup on this object, then... */
    kfree(mine);
}
```

The `KOBJ_REMOVE` action will be notified automatically when the kobject is being released.

You should never use `kfree()` to free a kobject directly once it is registered via `kobject_add()`.

## 3. Kobject hierarchies, Kset, and subsystems

The kobject is often used to link together objects into a hierarchical structures. There 2 ways for this linking: `parent` pointer and ksets.

### 3.1. ksets

A Kset is merely a collection of kobjects that want to be associated with each other. These object should have the same ktype. The ksets provide:

- A bag contains a group of objects. For example *all block devices* or *all PCI device drivers*.
- A subdirectory in sysfs.
- Support the hotplugging of kobjects and influence how uevent events are reported to the user.

Kset manage its children in a linked list.

To create and release a kset:

```c
struct kset *kset_create_and_add(const char *name,
                                   const struct kset_uevent_ops *uevent_ops,
                                   struct kobject *parent_kobj);
void kset_unregister(struct kset *k);
```

### 3.2. Subsystem

## 4. Hotplug event generation

A hotplug event is is a notification to user space from the kernel that something has changed in the system's configuration. They are generated whenever a kobject is created or destroyed. Such events are generated, for example, when a digital camera is plugged in with a USB cable, When a user switches console modes, etc.

## 5. Buses, Devices, and Drivers

Much of the material covered here will never be needed by many driver authors. You might never add a new bus type or implement a driver at bus level.

### 5.1. Buses

A bus is a channel between the processor and one or more devices. For the purposes of the device model, all devices are connected via a bus, even if it is an internal, virtual, "platform" bus. Buses can plug into each other -- a USB controller is usually a PCI device, for example. The device model represents the actual connections busses and the devices they control.

A bus is represented by the `bus_type` structure:

```c
struct bus_type {
  char *name;
  struct subsystem subsys;
  struct kset drivers;
  struct kset devices;
  int (*match)(struct device dev, struct device_driver drv);
  struct device (add)(struct device parent, char bus_id);
  int (*hotplug) (struct device dev, char *envp,
  int num_envp, char *buffer, int buffer_size);
  /* Some fields omitted */
};
```

#### 5.1.1. Iterating over devices and drivers

If you are writing bus-level code you may find yourself having to perform some operation on all devices or drivers that have been registered with your bus.

To operate on every device known to the bus, use:

```c
int bus_for_each_dev(const struct bus_type *bus, struct device *start,
                     void *data, device_iter_t fn);
```

Similar for iterating drivers:

```c
int bus_for_each_drv(const struct bus_type *bus, struct device_driver *start,
         void *data, int (*fn)(struct device_driver *, void *));
```

You also can find devices, drivers on the bus:

```c
struct device *bus_find_device(const struct bus_type *bus, struct device *start,
       const void *data, device_match_t match);
static inline struct device *
bus_find_device_by_of_node(const struct bus_type *bus, const struct device_node *np)
{
  return bus_find_device(bus, NULL, np, device_match_of_node);
}

/* And more. */
```
