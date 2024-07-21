---
title: 'Object Oriented Programming in C part IV: Inheritance'
description: >-
    In OOP, inheritance is a foundational concept that enables classes to inherit attributes and methods from others. Let's explore how this is achieved in C.
author: Cong
date: 2024-07-21 2:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
---

## 1. Basic about inheritance

Inheritance, a fundamental concept **enabling classes to inherit attributes and methods from others**, fosters hierarchical relationships in object-oriented programming. While C lacks the `class` concept, it allows for implementing inheritance through clever techniques, laying the groundwork for the powerful concept of **polymorphism**.

## 2. Apply inheritance in C

In the previous blog [OOP: Object Blog](/posts/oop-with-C-part-I/) we talked about how to create an object with OPP style. Let's explore how we can create a base object type and implement inheritance in a derived type using C. Look at to this example, we have a base object type `person` that contains base attributes like `name`, `age`, along with base methods like `talk()` and `work()`:

```c
typedef struct {
    char name[MAX_LENGTH];
    char language[MAX_LENGTH];
    int age;
} person_t;

int person_init(person_t *self, const char *name);
int person_deinit(person_t *self);

/* Pubic methods. */
int person_talk(person_t *self, const char *speech);
int person_work(person_t *self);
```

Now assume that we want to define an `employee` object type, also an `employee` is a `person` 😛, and we don't want to re-code all person's methods, attributes. So here we consist the `person_t` as a property of `employee_t`:

```c
typedef struct
{
    char company[];
    int salary;
    person_t person;
} employee_t;

int employee_init(employee_t *self, const char *name);
int employee_deinit(employee_t *self);
```

To create and destroy base object:

```c
int employee_init(employee_t *self, const char *name)
{
    memset(self, 0, sizeof(employee_t));
    person_init(&self->person, name);

    /* Init another employee's components ... */
    return 0;
}

int employee_deinit(employee_t *self)
{
    person_deinit(&self->person);
    return 0;
}
```

For using base type attributes and methods in application code, we have two choices:

1. Access the person attributes, methods directly (**Public Inheritance**).

    ```c
    int main()
    {
        employee_t John = {0};
        employee_init(&John, "John");

        printf("Employee name: %s\n", John.person.name);
        printf("Employee age: %s\n", John.person.age);

        person_talk(&John.person, "Hi, I'm John!");

        employee_init(&John);
    }
    ```

2. Wrapping them by employee methods (**Private Inheritance**).

    ```c
    const char *employee_get_name(employee_t *self)
    {
        return (const char *)self->person.name;
    }

    int employee_talk(employee_t *self, const char *speech)
    {
        return person_talk(&self->person, speech);
    }
    ```

    In the application code:

    ```c
    int main()
    {
        employee_t John = {0};
        employee_init(&John, "John");

        printf("Employee name: %s\n", employee_get_name(&John));
        employee_talk(&John, "Hi, I'm John!");

        employee_init(&John);
    }
    ```

## 3. Summary

Inheritance simplifies code reuse, establishes relationships between objects, and, most importantly, forms the foundation of polymorphism, enhancing code flexibility and generality for user applications.
