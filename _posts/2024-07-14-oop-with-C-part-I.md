---
title: 'Object Oriented Programming in C part I: Object'
description: >-
  OOP is a programming paradigm that revolves around the concept of "Object". Even though C doesn't support OOP features, we can still achieve the OOP benefits by using it as a coding pattern.
  
author: Cong
date: 2024-07-14 7:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP in C]
pin: true
---

## Why OOP?

OOP aims to structure code in a way that mimics real-world objects. And every object has three things:

- **Lifecycle**: This refers to the period during which an object exists in memory, starting from its creation (birth) until it is destroyed (death).
- **Properties**: These are characteristics or attributes that describe the object's state. Properties define what an object is or what it possesses.
- **Behaviors**: These are actions or operations that an object can perform. Behaviors define what an object can do or how it interacts with other objects and the outside world.

Let take a look an `Animal` class in CPP:

```cpp
class Animal
{
public:
  /* Animal properties. */
  int weight;
  int length;

  /* Animal behaviors. */
  void make_sound();
  void move();

  /* Animal lifecycle events: birth and death. */
  Animal()
  { // Do initialize...

  }

  ~Animal()
  { // Do cleanup ...

  }
}
```

Managing object lifecycle events helps us in the following ways:

- **Construct Objects**: We can observe when an object is being created, initialize its resources, and provide multiple ways to initialize objects.
- **Destroy Objects**: We can observe when an object is being destroyed and perform cleanup of its resources.

In summary, OOP organizes code around the concept of objects, which encapsulate data (properties) and behavior (methods), and each object has a defined lifecycle from creation to destruction. This approach helps developers model complex systems more effectively by mirroring real-world entities and their interactions.

## Make an object in C
