---
layout: post
title: "Who's calling me?"
permalink: /blog/2017/8/25/stack-walking-api-java9
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2017-8-25-stack-walking-api-java9.md"
excerpt: "There is new API in Java 9 which provides efficient mechanism for accessing to current thread's stack frames with abilities to filter and lazy access..."
---
# Status Quo

---
Imagine you're going to find the caller of the current executing method. To make it even more interesting, imagine you're going to get current thread's stack frames one by one and filter them to extract or expose some meaningful information. How would you do these?

Before Java 9, there were three different approaches that we could use to solve mentioned problems:

- `Throwable::getStackTrace` and `Thread::getStackTrace` which return an array of `StackTraceElement`s. Each `StackTraceElement` represents a stack frame. Both approaches would force the VM to eagerly capture a snapshot of the **entire stack**, which is very inefficient if you are only interested in the top few frames on the stack. Also, `StackTraceElement` doesn't retain class information and we're just have access to the class name. Obviously, `Class<?>` tokens are far more desirable in some use cases.

- `SecurityManager::getClassContext` is a `protected` method, which allows a `SecurityManager` subclass to access the class context. Sure it's working and somewhat efficient but if we only had a more pleasant API at our disposal!

- Last but not certainly least, there is my beloved, deprecated and JDK internal `sun.reflect.Reflection::getCallerClass` which is a very efficient way to find the caller of current executing method. Regardless of the fact that it's now deprecated, it's also a internal API which we shouldn't use in the first place.

Just to recap, current solutions are inefficient, incomplete, inconvenient and forbidden! [JEP 259][jep259] is an attempt to answer these problems.

## Stack Walking API

---
TODO

[jep259]:http://openjdk.java.net/jeps/259