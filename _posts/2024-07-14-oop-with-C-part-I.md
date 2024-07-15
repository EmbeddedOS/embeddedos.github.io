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

## 1. Why OOP?

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

## 2. Common pitfalls when coding in C without OOP concepts

### 2.1. Problems with object life-cycle

Unlike CPP or another OOP supported programming languages, C doesn't support managing object lifecycle events. So, initializing and cleaning up data structures may require more user code. It's not a big deal for simple objects, but for objects that have complex construction, it can become cumbersome.

Sometimes, creating complex objects leads to forgetting to clean up, causing resource leaks. Let's take a look at the example below: the function `some_fn()` creates a device object, initializes its properties, performs some tasks, and attempts to clean up the object before returning. However, the function forgets to `free()` a memory region and `close()` the file.

```c
struct device
{
  uint8_t *memory_region_1;
  uint8_t *memory_region_2;
  uint8_t *memory_region_3;
  int device_fd;
  const char device_path[MAX_LENGTH];
  /* More properties. */
}

void some_fn()
{
  /* ... */
  struct device dev = {.device_path = DEVICE_PATH};
  dev.memory_region_1 = malloc(MEMORY_LENGTH);
  dev.memory_region_2 = malloc(MEMORY_LENGTH);
  dev.device_fd = open(dev.device_path);

  /* ... */
  if (device_write(&dev, buf) < 0)
  {
    free(dev.memory_region_1);
    return;
  }
  
  /* ... */

  free(dev.memory_region_1);
  free(dev.memory_region_2);
}
```

### 2.2. Make global data structures

Creating global data structures and accessing them from everywhere makes:

- Controlling data flow more challenging when now the data are accessed from anywhere, and any time.
- Having to use synchronization mechanisms makes the code slower.
- Functions now become non-reentrant, and harder to test.
- The code becomes harder to read and maintain.

Here is an example where a developer runs `some_fn_1()` and `some_fn_2()` functions simultaneously. `some_fn_1()` opens a file and performs another task. Later, it attempts to read more data without realizing that the `fd` file descriptor has already been modified by `some_fn_2()`.

```c
static struct device dev = {0};
void some_fn_1()
{
  lock();
  dev.fd = open(DEVICE_PATH);
  res = read(dev.fd, buf, len);
  unlock();

  /* ... */

  lock();
  res = read(dev.fd, buf, len);
  unlock();
}


void some_fn_2()
{
  lock();
  dev.fd = open(ANOTHER_DEVICE_PATH);
  res = read(dev.fd, buf, len);
  unlock();
}
```

## 3. Object Pattern in C

### 3.1. Object properties

Let get an example we have a `person` object. We create a struct that encapsulate `person` properties.

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
```

### 3.2. Init and De-init methods

First thing we need to do for every objects is creating init and de-init methods. Encapsulating initialization and de-initialization becomes methods help use easy to manage objects. Even though developers forget to clean up, it's still easier to debug 😛

```c
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
> Resources that are initialized in `init()`, Should be cleaned-up in the `de-init()`.
{: .prompt-tip }

### 3.3. Object behaviors

Object contexts are passed like parameters to object's method, and in the implementations, we access the object data to perform object behaviors. Functions actually only act on objects, additional parameters are data that functions used to change the object behavior. This ensures that functions are always **reentrant**.

```c
/* Person behaviors. */
int person_talk(person_t *self, const char *speech)
{
  if (strcmp(self->language, "English") == 0)
  {
    printf("%s: %s\n", self->name, speech);
  }

  return 0;
}

int person_move(person_t *self, int distance)
{
  self->current_location += distance;
  return 0;
}
```

> Making functions reentrant helps us easily manage data flow, ensuring that the same input parameters always yield the same output, while also avoiding concurrency problems.
{: .prompt-tip }

### 3.4. Application code

Now take a look how we use this object in the application code.

```c

int main()
{
  /* 1. Create a object. */
  person_t person;
  person_init(&person, "John", ...);

  /* 2. Do object behaviors. */
  person_talk(&person, "Hi there!");
  person_move(&person, 3);

  /* 3. Clean-up object. */
  person_deinit(&person);
}
```

## 4. Summary

**OOP is not a feature that this language has and others don't**. It should be an idea that we should consider while developing applications. Here are some main ideas:

- Try to structure the code to become objects: have their own properties, behaviors, and lifecycle events.
- Avoid global data, static, extern variables that make the code is harder to read, test, control data flow.
- Clarify object life-cycle events: constructor and destructor, binding resources with object's lifetime.
- Make re-entrant methods, methods only act on objects and parameters.
