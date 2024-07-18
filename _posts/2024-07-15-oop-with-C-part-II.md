---
title: 'Object Oriented Programming in C part II: Abstraction'
description: >-
 Abstraction in software design hides complex implementations, large systems, and providing simple interfaces to users, making it a key concept. Let's see how we achieve this in C.

author: Cong
date: 2024-07-15 7:00:00 +0800
categories: [Design Pattern, OOP-C]
tags: [Design Pattern, OOP, C]
pin: true
---

## 1. Introduction

Abstraction refers to the the concept of hiding the complex implementation details of a system, and exposing only the essential features of the object.

### 1.1. Abstraction in real world example

Let get an example of abstraction in Internet System. Assume that you want to access to the `facebook.com` website. You simply typing it in your browser. But behind the scene are combination of massive systems.

1. The browser query to the DNS server in your location to get the IP address of the `facebook.com` website.
2. The browser perform sending requests to the `facebook.com` domain's IP address.
3. As you know, an IP address represents a host machine on the network. However, to handle requests from millions of users per second, Facebook needs to deploy a large-scale system infrastructure comprising many servers. To provide a simple interface to users in a region, Facebook deploys load-balancer systems. These systems dispatch requests to multiple servers behind the scenes, all accessible via a single address.

In this example, abstraction is how your browser abstracts **How to get resources from a website** and how Facebook abstracts **How to access Facebook servers**.

### 1.2. Abstraction in C example

## 2. Abstraction with Opaque Object pattern
