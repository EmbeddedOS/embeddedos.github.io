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

The root object class is `TYPE_OBJECT` which provides basic methods. Every type have to inherit from another class, at least from the root object class. That QOM become a composition tree.

Here is an example of how to define a new device type in QEMU, the object inherit from the type `TYPE_DEVICE`:

```c
#define TYPE_MY_DEVICE "my-device"

typedef DeviceClass MyDeviceClass;
typedef struct
{
    DeviceState parent_class;

    int reg0, reg1, reg2;
} MyDevice;

static const TypeInfo my_device_info = {
    .name = TYPE_MY_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(MyDevice),
};

static void my_device_register_types(void)
{
    type_register_static(&my_device_info);
}

type_init(my_device_register_types)
```

### QOM tree

The root object class is `TYPE_OBJECT`, and then basic component types and finally end up with machine types.

```text
TYPE_OBJECT
├──TYPE_EVENT_LOOP_BASE
│   ├──TYPE_MAIN_LOOP
│   └──TYPE_IOTHREAD
├──TYPE_DEVICE
│   ├──TYPE_TYPE_AW_A10
│   ├──TYPE_FLOPPY_DRIVE
│   ├──TYPE_SERIAL
│   ├──TYPE_CPU
│   │   ├──TYPE_ALPHA_CPU
│   │   ├──TYPE_ARM_CPU
│   │   │   ├──TYPE_AARCH64_CPU
│   │   │   ├──TYPE_ARM_CPU '-cortex-a7'
│   │   │   ├──TYPE_ARM_CPU '-cortex-m4'
│   │   │   ├──TYPE_ARM_CPU '-cortex-r5'
│   │   │   └──...
│   │   ├──TYPE_X86_CPU
│   │   ├──TYPE_RISCV_CPU
│   │   └──...
│   ├──TYPE_REGISTER
│   ├──TYPE_I2C_SLAVE
│   │   ├──TYPE_SMBUS_DEVICE
│   │   ├──TYPE_AT24C_EE
│   │   └──...
│   ├──TYPE_USB_DEVICE
│   │   ├──TYPE_USB_HID
│   │   ├──TYPE_USB_STORAGE
│   │   └──...
│   ├──TYPE_VIRTIO_DEVICE
│   │   ├──TYPE_VIRTIO_NET
│   │   ├──TYPE_VIRTIO_SERIAL
│   │   ├──TYPE_VIRTIO_IOMMU
│   │   ├──TYPE_VIRTIO_MEM
│   │   └──...
│   ├──TYPE_SYS_BUS_DEVICE
│   │   ├──TYPE_ARMV7M
│   │   ├──TYPE_OMAP_I2C
│   │   ├──TYPE_STM32F405_SOC
│   │   └──...
│   └──...
├──TYPE_MEMORY_BACKEND
│   ├──TYPE_MEMORY_BACKEND_FILE
│   ├──TYPE_MEMORY_BACKEND_RAM
│   └──...
├──TYPE_CHARDEV
│   ├──TYPE_CHARDEV_SOCKET
│   │   └──TYPE_CHARDEV_DBUS
│   ├──TYPE_CHARDEV_FD
│   ├──TYPE_CHARDEV_FILE
│   ├──TYPE_CHARDEV_NULL
│   ├──TYPE_CHARDEV_PIPE
│   ├──TYPE_CHARDEV_SERIAL
│   ├──TYPE_CHARDEV_CONSOLE
│   └──...
├──TYPE_BUS
│   ├──TYPE_I2C_BUS
│   ├──TYPE_SYSTEM_BUS
│   └──TYPE_FLOPPY_BUS
├──TYPE_IRQ
├──TYPE_MACHINE
│   ├──TYPE_X86_MACHINE
│   ├──TYPE_ARDUINO_MACHINE
│   ├──TYPE_VIRT_MACHINE
│   └──...
├──TYPE_MEMORY_REGION
└──...
```

### 2.3. Class Instance

Every type has an `ObjectClass` associate with it. The struct that hold shared information between object instances. For example, an `ObjectClass` can hold virtual table, methods, cast-caching that are generic for all object instances of the type.

In C++, or higher level OOP languages, we don't see the class instance concepts, because the compiler make it automatically, under the hood. But in C, QEMU have to do it manually.

#### 2.3.1. Class initialization

Before any objects are initialized, the class for the object must be initialized. There is only one class object for all instance objects.

After all of the parent classes have been initialized, the `TypeInfo::class_init` callback is called.

For example, in this case, we define our type `OUR_TYPE_DEVICE` that inherit from `TYPE_DEVICE` class. We define `TypeInfo::class_init` callback to override the default `reset()` and `realize()` methods of `TYPE_DEVICE`.

```c
void our_device_class_init(ObjectClass *klass, void *class_data)
{
    /* 1. Get parent class object. */
    DeviceClass *dc = DEVICE_CLASS(klass);

    /* 2. Override reset() and realize() methods. */
    dc->reset = our_device_reset;
    dc->realize = our_device_realize;
}

static const TypeInfo my_device_info = {
    .name = OUR_TYPE_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(OurDevice),
    .class_init = our_device_class_init,
};
```

## 3. The QOM Object

### 3.1. Create a new object

To create new object instance we can use `object_new()` API:

```c
Object *object_new(const char *typename);

Object *object_new_with_class(ObjectClass *klass);
```

The API returns an object base that we can cast to our type.

### 3.2. Inheritance casting

QOM use `Object` struct as base for all class. The first member of any types always are their parents. For example, our type inherit from `TYPE_DEVICE`, we have to put struct `DeviceState` as a first member.

```c
typedef DeviceClass MyDeviceClass;
typedef struct
{
    DeviceState parent_class;

    int reg0, reg1, reg2;
} MyDevice;

static const TypeInfo my_device_info = {
    .name = TYPE_MY_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(MyDevice),
};
```

The `DeviceState` that represents `TYPE_DEVICE` type, do the the same way. It inherits `TYPE_OBJECT` and have to define `Object parent_obj;` as a first member.

```c
struct DeviceState {
    /* private: */
    Object parent_obj;
    /* public: */

    /**
     * @id: global device id
     */
    char *id;
    /**
     * @canonical_path: canonical path of realized device in the QOM tree
     */
    char *canonical_path;
    /**
     * @realized: has device been realized?
     */
    bool realized;

    // ...
};

```

The reason why we have to put the parent object as a first member, since C guarantees that the first member of a structure always begins at byte 0 of that structure, as long as any sub-object places its parent as the first member, we can cast directly to a `Object`.

### 3.3. Properties

### 3.4. Methods

### 3.5. Destroy an object

## 4. Load the module with `type_init()` macro

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
