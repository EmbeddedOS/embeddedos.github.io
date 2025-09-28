---
title: "Low latency programming: lock-free data structures."
description: >-
  Designing lock-free concurrent data structures.

author: Cong
date: 2025-09-16 00:01:00 +0800
categories: [low-latency, lock-free]
tags: [low-latency, oop, lock-free, cpp]
image:
  path: assets/img/lockfree.png
  alt: lock-free.
published: true
---

If a data structure is to be accessed from multiple threads, either it must be completely immutable and no synchronization is necessary, or the program must be designed to ensure that changes are correctly synchronized between threads. Before we discover the lock-free design, let's take a look at lock-based concurrent data structures.

## 1. Lock based concurrent data structures

Basically, designing a data structure for concurrency means that multiple threads can access the data structure concurrently. To ensure concurrent operations are safe and correctly, one option is to use a lock to protect the data, and we can call that as a thread-safe data structure. The simplest one typically use mutexes and locks to protect data.

Design of lock based concurrent data structures is all about ensuring that the right mutex is locked when accessing the data and the lock is held for the minimum amount of time. Let's get an example of a thread-safe lock based queue, that use mutex to protect access to the queue, and the condition variable to notify waiting threads.

```cpp
#pragma once

#include <queue>
#include <condition_variable>
#include <mutex>

template <typename T>
class threadsafe_queue
{
    std::queue<T> _queue;
    std::mutex _mutex;
    std::condition_variable _cond;

public:
    T pop()
    {
        std::unique_lock<std::mutex> lock(this->_mutex);

        this->_cond.wait(lock, [this]() -> bool
                         { return !this->_queue.empty(); });

        T item = this->_queue.front();
        this->_queue.pop();
        return item;
    }

    bool try_pop(T &item)
    {
        std::unique_lock<std::mutex> lock(this->_mutex);
        if (this->_queue.empty())
        {
            return false;
        }

        item = std::move(this->_queue.front());
        this->_queue.pop();
        return true;
    }

    void push(T item)
    {
        std::unique_lock<std::mutex> lock(this->_mutex);
        this->_queue.push(std::move(item));
        this->_cond.notify_one();
    }
};
```

By using mutexes, we ensure multiple threads can safely access a data structure without encountering race conditions. But every coin has two sides, using mutex incorrectly can lead to a deadlock, and the algorithms to implement those things are called as *blocking*. Others threads attempting to acquire the same lock will be blocked, and that might lead to performance bottlenecks. In this article, we will learn other designing of concurrent data structure, where we can access the structure safely but no locks are required and can avoid those problems too.

## 2. Lock free concurrent data structures

In the previous sections we learned lock-based data structure can be blocking. But 

## 3. atomic operations

- Compare and Swap (CAS).
- Fetch and Add.

## 4. Pros and cons of lock-free data structure

## 5. Lock free containers

## 6. Wait free style

