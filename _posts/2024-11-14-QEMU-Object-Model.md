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

Some of important members:

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

The reason why we have to put the parent object as a first member, since C guarantees that the first member of a structure always begins at byte 0 of that structure, as long as any sub-object places its parent as the first member, we can cast directly to a `Object` as well as to parent and sub types.

### 3.3. Properties

In QOM, we have two type of class member: C struct member of your type and Property member. The different is that, you can use Property as a external interface. We don't need to expose internal implement like directly access to struct member, Accessor accesses the properties via setter, getter APIs.

A property is represented by struct `ObjectProperty`.

```c
struct ObjectProperty
{
    char *name;
    char *type;
    char *description;
    ObjectPropertyAccessor *get;
    ObjectPropertyAccessor *set;
    ObjectPropertyResolve *resolve;
    ObjectPropertyRelease *release;
    ObjectPropertyInit *init;
    void *opaque;
    QObject *defval;
};
```

Some of important members:

- `name` The property name.
- `type` property type. Can be fundamental types (int, uint32, string, etc), user structure, or specified command line types (`on|off|split`).
- `get` Getter.
- `set` Setter.
- `init` property's initializer.
- `release` property's destructor.

Both `Object` and `ObjectClass` hold a GHashTable `properties` fields. The `ObjectClass` hold common class's properties whereas the `Object` hold specified object's properties.

#### 3.3.1. Add a property to an object

To add a property to a object, QEMU provide some APIs and some helper functions to add specific types. The basic API `object_property_add()`

```c
ObjectProperty *object_property_add(Object *obj, const char *name,
                                    const char *type,
                                    ObjectPropertyAccessor *get,
                                    ObjectPropertyAccessor *set,
                                    ObjectPropertyRelease *release,
                                    void *opaque);

```

The `opaque` is an opaque pointer to pass to the callbacks. An error like property already exist might abort the program. That is the reason QEMU provide other `object_property_try_add()` that can return an error code instead.

QEMU also provide some helpers to add well-known types, for example `string` and `bool`:

```C
ObjectProperty *object_property_add_str(Object *obj, const char *name,
                             char *(*get)(Object *, Error **),
                             void (*set)(Object *, const char *, Error **));

ObjectProperty *object_property_add_bool(Object *obj, const char *name,
                              bool (*get)(Object *, Error **),
                              void (*set)(Object *, bool, Error **));
```

#### 3.3.2. Get the property's value

To get a property's value we use the API `object_property_get()` and provide a Visitor that hold the callback to get value.

The getter callback also will be called in case of getting success.

```c
bool object_property_get(Object *obj, const char *name, Visitor *v,
                         Error **errp);
```

`Visitor` struct is a generic interface to access the properties with any types. It holds function pointer for every types:

```c
struct Visitor
{
    /* Must be set */
    bool (*type_bool)(Visitor *v, const char *name, bool *obj, Error **errp);

    /* Must be set */
    bool (*type_str)(Visitor *v, const char *name, char **obj, Error **errp);

    /* Must be set to visit numbers */
    bool (*type_number)(Visitor *v, const char *name, double *obj,
                        Error **errp);

    /* Must be set to visit arbitrary QTypes */
    bool (*type_any)(Visitor *v, const char *name, QObject **obj,
                     Error **errp);

    /* Must be set to visit explicit null values.  */
    bool (*type_null)(Visitor *v, const char *name, QNull **obj,
                      Error **errp);
    // ...
}
```

QEMU also provides helper function to get specific types easier for example: `object_property_get_str()`, `object_property_get_bool()`.

#### 3.3.3. Set the property's value

Similar like get API, we have basic set API:

```c
bool object_property_set(Object *obj, const char *name, Visitor *v,
                         Error **errp);
```

QEMU also provides helper function to set specific types easier for example: `object_property_set_str()`, `object_property_set_bool()`.

#### 3.3.4. Remove the property from the object

```C
void object_property_del(Object *obj, const char *name);
```

#### 3.3.5. Add, remove properties to the object class

Some properties that you want to share for every objects of a class, you can add them directly to the `ObjectClass` struct, QEMU provides same APIs like adding and removing properties object, but for class object. For example:

```c
ObjectProperty *object_class_property_add(ObjectClass *klass, const char *name,
                                          const char *type,
                                          ObjectPropertyAccessor *get,
                                          ObjectPropertyAccessor *set,
                                          ObjectPropertyRelease *release,
                                          void *opaque);
```

There is no APIs for getting, setting properties on class object, because getting and setting is each object's behaviors.

### 3.4. Methods

A method a function pointer that is hold as a struct member. The different thing in QOM is that, you can override the default parent method by assign another function pointer when the `ClassObject` init() running. For example:

```c
typedef struct MyState MyState;

typedef void (*MyDoSomething)(MyState *obj);

typedef struct MyClass {
    ObjectClass parent_class;

    MyDoSomething do_something;
} MyClass;

static void my_do_something(MyState *obj)
{
    // do something
}

static void my_class_init(ObjectClass *oc, void *data)
{
    MyClass *mc = MY_CLASS(oc);

    mc->do_something = my_do_something;
}

static const TypeInfo my_type_info = {
    .name = TYPE_MY,
    .parent = TYPE_OBJECT,
    .instance_size = sizeof(MyState),
    .class_size = sizeof(MyClass),
    .class_init = my_class_init,
};

typedef struct DerivedClass {
    MyClass parent_class;
} DerivedClass;

static void derived_do_something(MyState *obj)
{
    DerivedClass *dc = DERIVED_GET_CLASS(obj);
    // do something here
}

static void derived_class_init(ObjectClass *oc, void *data)
{
    MyClass *mc = MY_CLASS(oc);
    mc->do_something = derived_do_something;
}

static const TypeInfo derived_type_info = {
    .name = TYPE_DERIVED,
    .parent = TYPE_MY,
    .class_size = sizeof(DerivedClass),
    .class_init = derived_class_init,
};
```

In the above example, we define class `DerivedClass` inherit `MyClass`, in the `derived_class_init()` we override method `do_something()` by our `derived_do_something()`. That means from now, with every objects of the type, every call to `do_something()` method will always call the `derived_do_something()` instead.

### 3.5. Destroy an object

Object reference count is that way QEMU manege object deletion. When you create a new one, the default count is 1. The API `object_ref()` make a reference to object and increase the count by 1, the API `object_unref()` delete the reference and decrease the count by 1. When the count become zero, the object will be destroyed and the destructor will be call.

You shouldn't hold the object pointer copy by hand, the better thing is hold the reference instead.

### 3.6. Device Life-cycle

In QEMU everything is a object even devices. On the `realize()` successful completion, the object is added to QOM tree and make visible to the guest.

The reverse `unrealize()` should be defined to undo device initialization in `realize()`.

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
