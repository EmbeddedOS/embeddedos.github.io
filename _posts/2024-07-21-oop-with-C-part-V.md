---
title: 'Object Oriented Programming in C part V: Polymorphism'
description: >-
    Polymorphism in OOP enables different objects to be treated as instances of a shared base type. This allows objects to be manipulated in a uniform manner without regard to their specific object type.

author: Cong
date: 2024-07-21 5:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C, CONTAINER_OF]
pin: true
image:
  path: assets/img/oop.jpg
  alt: Object Oriented Programming.
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

**An abstract interface** in OOP defines a contract for classes that inherit from it, specifying methods that must be implemented by subclasses but not providing an implementation itself. This allows for polymorphic behavior without specifying how each subclass should behave.

In the example of **Linux Kernel Device Drivers**, the abstract interface is represented by the `file operations struct`. This requires device driver developers to define their own operation callbacks following a standard. Developers have flexibility in how they interact with devices but must adhere to the same interface for user-space interactions. That's how Linux provides simple interfaces for user-space.

## 3. Implementation in C

Let's consider an example of a `person` object that provides a virtual method `work()`. We also have more derived types of it: `worker` and `singer`.

### 3.1. Base `person` object

The interface that we provide:

```c
struct person_i_t;
typedef struct person_i_t *person_t;

/* Public methods ... */
const char *person_get_name(person_t self);
int person_talk(person_t self, const char *speech);

/* Virtual methods. */
int person_work(person_t self);
```

In the source file, we implement base methods. For the the virtual method, we call the call back of derived objects:

```c
typedef int (*v_work)(person_t self);
struct person_i_t
{
    v_work work_cb;

    /* Base properties. */
    char name[MAX_LENGTH];
    char language[MAX_LENGTH];
    int age;
};
typedef struct person_i_t person_i_t;

static int person_work_base(person_t self)
{
    printf("Do default person work behavior.\n");
}

const char *person_get_name(person_t self);
{
    return self->name;
}

int person_work(person_t self)
{
    if (self->work_cb)
    {
        return self->work_cb(self);
    }
    else
    { // If the user doesn't override this, we call default method.
        return person_work_base(self);
    }
}
```

### 3.2. derived `worker` object

Here is the interface that we provide, where we use the type `person_t` to represent the `self` object for generic purposes.

```c
typedef person_t worker_t;

worker_t worker_init(const char *name);
int worker_deinit(worker_t *self);

/* More worker specific methods ... */
int worker_get_current_construction(worker_t self);
int worker_get_current_tools(worker_t self);
```

#### 3.2.1. `CONTAINER_OF` macro

But the problem in our implementation now is how to retrieve the original `worker_i_t` object from `person_t` object? The answer is using the `CONTAINER_OF` macro.

> `CONTAINER_OF` macro is used to find the container structure address of the given member. The `CONTAINER_OF` used in Linux Kernel development, so it's not available in the C standard library. Manually, we have to add the macro to our project.
{: .prompt-info }

```c
#define CONTAINER_OF(ptr, type, member) ({         \
    const typeof( ((type *)0)->member ) *__mptr = (ptr); \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

The `worker` object source file implementation:

```c
typedef struct {
    /* Worker specific  properties. */
    const char tools[MAX_SIZE];
    const char current_construction[MAX_SIZE];
    person_i_t person;
} worker_i_t;

/* Implementation of worker specific work() API. */
static int worker_work(person_t self)
{
    worker_i_t *worker_self = CONTAINER_OF(self, worker_i_t, person);
    printf("Using %s to build %s.", worker_self->tools,
            worker_self->current_construction);
    return 0;
}

worker_t worker_init(const char *name)
{
    worker_i_t *self = malloc(sizeof(worker_i_t));
    memset(self, 0, sizeof(worker_i_t));
    strcpy(self->person.name, name);

    self->person.work_cb = worker_work;

    return &self->person;
}

int worker_get_current_construction(worker_t self)
{
    worker_i_t *worker_self = CONTAINER_OF(self, worker_i_t, person);
    return worker_self->current_construction;
}

/* ... */
```

### 3.3. derived `singer` object

We can implement `singer` object in similar way with `worker` object.

The interface:

```c
typedef person_t singer_t;

singer_t singer_init(const char *name);
int singer_deinit(singer_t *self);

/* More worker specific methods ... */
int singer_get_current_concert(singer_t self);
int singer_dance(singer_t self);
```

Specific `singer` object implementation:

```c
typedef struct {
    /* Singer specific properties. */
    const char current_concert[MAX_SIZE];
    person_i_t person;
} singer_i_t;

/* Implementation of singer specific work() API. */
static int singer_work(person_t self)
{
    singer_i_t *singer_self = CONTAINER_OF(self, singer_i_t, person);
    printf("Singing in concert %s.", singer_self->current_concert);
    return 0;
}

singer_t singer_init(const char *name)
{
    singer_i_t *self = malloc(sizeof(singer_i_t));
    memset(self, 0, sizeof(singer_i_t));
    strcpy(self->person.name, name);

    self->person.work_cb = singer_work;

    return &self->person;
}

/* ... */
```

### 3.4. Application code

In the application code, we can use the same virtual API `person_work()` for two types derived from the base object `person`: `worker` and `singer`. Also, we can do specific behaviors of derived objects.

```c
int main()
{
    worker_t John = worker_init("John");
    single_t Anna = singer_init("Anna");

    /* Do specific work with the same API. */
    person_work(John);
    person_work(Anna);

    /* Do specific behaviors. */
    printf("%s is working at %s.\n", person_get_name(John), worker_get_current_construction(John));
    singer_dance(&Anna);

    /* Destroy objects. */
    worker_deinit(&John);
    singer_deinit(&Anna);
}
```

## 4. Summary

- **Polymorphism** enhances the modularity, flexibility, and maintainability of object-oriented systems by allowing objects of different types to be manipulated through a unified interface, promoting code reuse and extensibility.
- **CONTAINER_OF** is a very useful macro that allows us to retrieve the address of a containing struct from the address of one of its members. In this topic, we learned how to obtain the address of a derived object from a base object address. We will delve deeper into the additional benefits of this macro in a separate topic.
