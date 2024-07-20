---
title: 'Object Oriented Programming in C part III: Encapsulation'
description: >-
 Let's understand the OOP encapsulation concept and how we achieve it in C.
  
author: Cong
date: 2024-07-20 2:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
---

## 1. Encapsulation in C

In OOP, encapsulation refers to the bundling of data with the methods that operate on that data. Or the restricting of direct access to some of an object's components.

### 1.2. Bundling data

- With C, that could be done by define your `struct` type, you can refer to [OOP: Object Blog](/posts/oop-with-C-part-I/) to see how we create objects in C using an OOP style.

### 1.3. Hiding and restricting access to object's components

In the [OOP: Abstraction in C Blog](/posts/oop-with-C-part-II/), we were discussing how to hide entire objects and only expose APIs that manipulate them. In the case you only want to restrict access to some properties, **make them like opaque properties and write wrapping APIs to access them**. Let take a look to this example.

The header file provides public properties `name` and `language`, along with methods `person_talk()`, `person_move()`, `person_get_length()` and `person_set_length()` to user.

```c
struct person_private_i_t;
typedef struct person_private_i_t* person_private_t;

typedef struct {
    /* Public properties. */
    const char name[MAX_LENGTH];
    const char language[MAX_LENGTH];

    /* Private properties. */
    person_private_t p_data;
} person_t;

/* Constructor. */
int person_init(person_t *self, const char *name);

/* Destructor. */
int person_deinit(person_t *self);

/* And public methods ... */
int person_talk(person_t *self, const char *speech);
int person_move(person_t *self, int distance);

int person_get_length(person_t *self);
int person_set_length(person_t *self, int length);

```

Here is our implementation for private properties `length` and `current_location`, along with private methods `person_init_private_data` and `person_deinit_private_data`:

```c
struct person_private_i_t
{
    /* Person private properties. */
    int length;
    int current_location;
}

/* Private methods. */
static int person_init_private_data(person_t *self)
{
    self->p_data = (person_private_t)malloc(sizeof(person_private_i_t));
    return 0;
}

static int person_deinit_private_data(person_t *self)
{
    free(self->p_data);
    self->p_data = NULL;
    return 0;
}

/* Public methods. */
int person_init(person_t *self, const char *name)
{
    memset(self, 0, sizeof(person_t));
    person_init_private_data(self);
    /* ... */
    return 0;
}

int person_get_length(person_t *self)
{
    return self->p_data->length;
}

int person_set_length(person_t *self, int length)
{
    self->p_data->length = length;
    return 0;
}

int person_deinit(person_t *self)
{
    /* De-init object ... */
    person_deinit_private_data(self);
    return 0;
}
```

> This often necessitates dynamic memory allocation using `malloc()` (applications don't know about the object size) every time we create object, which can lead to **memory fragmentation** if used excessively. One solution is to provide a special API to retrieve the object size, enabling users to manage memory themselves. This approach allows for alternative allocation methods like `alloca()` for stack memory allocation, addressing concerns about memory fragmentation caused by frequent `malloc()` calls.
{: .prompt-warning }

The application code:

```c
int main()
{
    /* 1. Create a object. */
    person_t John = {0};
    person_init(&John, "John");

    /* 2. Do object behaviors. */
    person_set_length(&John, 180);
    printf("Person length: %d\n", person_get_length(&John));
    printf("Person name: %s\n", John.name);

    /* 3. Clean-up object. */
    person_deinit(&person);
}
```

## 2. Summary

Encapsulation offers several benefits:

- **Modularity**: Encapsulation allows you to group related variables (properties) and functions (methods) together to become structures (objects),  making the code more modular and easier to manage.
- **Information Hiding**: By hiding the internal state of an object and exposing only the necessary interfaces (public methods), encapsulation protects the integrity of data and prevents unauthorized access.
