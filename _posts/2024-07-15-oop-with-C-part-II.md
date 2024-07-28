---
title: 'Object Oriented Programming in C part II: Abstraction'
description: >-
 Abstraction in software design hides complex implementations, large systems, and providing simple interfaces to users, making it a key concept. Let's see how we achieve this in C.

author: Cong
date: 2024-07-15 7:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
pin: true
image:
  path: assets/img/oop.jpg
  alt: Object Oriented Programming.
---

## 1. Introduction

Abstraction refers to the the concept of hiding the complex implementation details of a system, and exposing only the essential features of the object.

### 1.1. Example of abstraction in the real world

Let get an example of abstraction in Internet System. Assume that you want to access to the `facebook.com` website. You simply typing it in your browser. But behind the scene are combination of massive systems.

1. The browser query to the DNS server in your location to get the IP address of the `facebook.com` website.
2. The browser perform sending requests to the `facebook.com` domain's IP address.
3. As you know, an IP address represents a host machine on the network. However, to handle requests from millions of users per second, Facebook needs to deploy a large-scale system infrastructure comprising many servers. To provide a simple interface to users in a region, Facebook deploys load-balancer systems. These systems dispatch requests to multiple servers behind the scenes, all accessible via a single address.

In this example, abstraction is how your browser abstracts **How to get resources from a website** and how Facebook abstracts **How to access Facebook servers**.

### 1.2. Example of abstraction in C

A typical example of abstraction in C is system calls, which are interfaces provided by the kernel for user space to access its services. Take this example: using the `open()` and `write()` system calls, you tell the kernel that you want to write content to a file, and the kernel handles the writing process.

```c
int fd = open(FILE_PATH, O_WRONLY);
int res = write(fd, "Your data", 9);
```

## 2. Abstraction with objects

### 2.1. The Opaque Type

The opaque Object Pattern, based on the C opaque type feature, revolves around a data type whose concrete data structure is not defined in its interface. With this pattern, you cannot create objects directly, but you can hold a pointer to them (known as an `opaque pointer`). Users can interact with APIs that accept these opaque pointers as parameters, enabling functionality while keeping the underlying implementation hidden.

```c
struct opaque_object_t;
int opaque_object_do_something(struct opaque_object_t *obj, int params);
```

### 2.2. Abstraction with Opaque Object

Back to the previous `person object` example in the [OOP: Object Blog](/posts/oop-with-C-part-I/). This time, we aim to expose only necessary APIs without revealing the entire `person struct`. In the header file (the interface provided), we include a constructor `person_init` for creating a person object, a destructor `person_deinit` for cleanup, and additional methods for manipulating the object:

```c
struct person_i_t;
typedef struct person_i_t* person_t;

/* Constructor. */
person_t person_init(const char *name);

/* Destructor. */
int person_deinit(person_t *self);

/* And more methods ... */
int person_talk(person_t self, const char *speech);
int person_move(person_t self, int distance);
```

And in the source file, we define the internal person struct:

```c
struct person_i_t
{
  /* Person properties. */
  const char name[MAX_LENGTH];
  const char language[MAX_LENGTH];

  int length;
  int weight;
  int current_location;
};

int person_deinit(person_t *self)
{
    free(*self);
    *self = NULL;
    return 0;
}

person_t person_init(const char *name)
{
    person_t self = (person_t)malloc(sizeof(person_i_t));
    /* Init the object ... */
    strcpy(self->name, name);

    return self;
}

int person_talk(person_t self, const char *speech)
{
  if (strcmp(self->language, "English") == 0)
  {
    printf("%s: %s\n", self->name, speech);
  }

  return 0;
}

int person_move(person_t self, int distance)
{
  self->current_location += distance;
  return 0;
}
```

> The reason we use a double pointer for the destructor function is to be able to assign the user pointer to `NULL`, thereby preventing it from holding an invalid address.
{: .prompt-tip }

This approach allows us to conceal all implementation details of the person object. In the application code, there is no need to concern ourselves with the internal implementation of the `person struct`. We can interact with the object solely through its defined interface and methods provided in the header file. This encapsulation simplifies usage and enhances maintainability by abstracting away unnecessary implementation specifics.

```c
int main()
{
  /* 1. Create a object. */
  person_t John = person_init("John");

  /* 2. Do object behaviors. */
  person_talk(John, "Hi there!");
  person_move(John, 3);

  /* 3. Clean-up object. */
  person_deinit(&John);
}
```

> This often necessitates dynamic memory allocation using `malloc()` (applications don't know about the object size) every time we create object, which can lead to **memory fragmentation** if used excessively. One solution is to provide a special API to retrieve the object size, enabling users to manage memory themselves. This approach allows for alternative allocation methods like `alloca()` for stack memory allocation, addressing concerns about memory fragmentation caused by frequent `malloc()` calls.
{: .prompt-warning }

## 3. Summary

Abstraction is a fundamental concept in software development, crucial not only in Object-Oriented Programming (OOP) but also in real-world applications. Understanding this concept can shift our coding mindset towards always striving to simplify interfaces for the users.
Let's summarize some benefits of deploying the OOP abstraction concept in C code:

- **Interface Simplification**: Provides clear and simplified interfaces to interact with objects, hiding implementation details.
- **Code Reusability**: Promotes reuse of code through inheritance and polymorphism, facilitating easier maintenance and updates.
- **Enhanced Modifiability**: Facilitates easier modification and extension of code without affecting other parts of the system.
