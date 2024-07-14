---
title: 'Object Oriented Programming in C part I: Object'
description: >-
  OOP is a programming paradigm that revolves around the concept of "Object". Even though C doesn't support OOP features, we can still achieve the OOP benefits by using it as a coding pattern.
  
author: Cong
date: 2024-07-14 7:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
pin: true
---

## Why OOP?

OOP aims to structure code in a way that mimics real-world objects. And every object has three things:

- **Lifecycle**: This refers to the period during which an object exists in memory, starting from its creation (birth) until it is destroyed (death).
- **Properties**: These are characteristics or attributes that describe the object's state. Properties define what an object is or what it possesses.
- **Behaviors**: These are actions or operations that an object can perform. Behaviors define what an object can do or how it interacts with other objects and the outside world.

Let take a look an `Person` class in CPP:

```cpp
class Person
{
public:
  /* Person properties. */
  std::string name;
  std::string language;

  int length;
  int weight;

  /* Person behaviors. */
  int talk();
  int work();

  /* Person object lifecycle events: birth and death. */
  Person()
  { // Do initialize...

  }

  ~Person()
  { // Do cleanup ...

  }
}
```

Managing object lifecycle events helps us in the following ways:

- **Construct Objects**: We can observe when an object is being created, initialize its resources, and provide multiple ways to initialize objects.
- **Destroy Objects**: We can observe when an object is being destroyed and perform cleanup of its resources.

In summary, OOP organizes code around the concept of objects, which encapsulate data (properties) and behavior (methods), and each object has a defined lifecycle from creation to destruction. This approach helps developers model complex systems more effectively by mirroring real-world entities and their interactions.

## Object Pattern in C

Unlike CPP or another OOP supported programming languages, C doesn't support managing object lifecycle events. So 

```c
typedef struct
{
  /* Person properties. */
  const char name[MAX_LENGTH];
  const char language[MAX_LENGTH];

  int length;
  int weight;
  int current_location;

} person_t;

/* Person object lifecycle events: init and de-init. */
int person_init(person_t *self, const char *name, ...)
{
  memset(self, 0, sizeof(person_t));
  /* Init object ... */
  return 0;
}

int person_deinit(person_t *self)
{
  /* De-init object ... */
  return 0;
}
```

> Try to use `self` naming to represent the object context.
{: .prompt-tip }

Object contexts are passed like parameters to object's method, and in the implementations, we access the object data to perform object behaviors. Functions actually only act on objects, additional parameters are data that functions used to change the object behavior. This ensures that functions are always **reentrant**.

```c
/* Person behaviors. */
int person_talk(animal_t *self)
{
  if (strcmp(self->language, "English") == 0)
  {
    printf("%s: Hi there!\n", self->name);
  }

  return 0;
}

int person_move(animal_t *self, int distance)
{
  self->current_location += distance;
  return 0;
}
```

> Making functions reentrant helps us easily manage data flow, ensuring that the same input parameters always yield the same output, while also avoiding concurrency problems.
{: .prompt-tip }
