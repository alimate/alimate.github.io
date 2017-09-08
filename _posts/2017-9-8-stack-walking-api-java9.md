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

Just to recap, current solutions are inefficient, incomplete, inconvenient and forbidden! [JEP 259][jep259] is an attempt to address these issues.

## Stack Walking API

---
JEP 259 or the *Stack Walking API* provides a standard and efficient API for stack walking (surprise!) that allows easy filtering of, and lazy access to, the information in stack traces.

The `StackWalker` class is the main entry point to access the Stack Walking API. Hence, In order to work with it, we should first obtain an instance of `StackWalker` using one of four different flavors of `getInstance` factory method. Those different overloaded versions of `getInstance` differ in how much information you want to obtain and how deep you're willing to go in the stack!

### So who's calling then?

---
Let's write some code! Suppose we're going to find the caller of the current executing method:
{% highlight java %}
public class Foo {
    public static void whoIsIt() {
        StackWalker walker = StackWalker.getInstance();
        System.out.println(walker.getCallerClass());
    }
}
{% endhighlight %}
If we run the preceeding code, we'd get the following exception:

    Exception in thread "main" java.lang.UnsupportedOperationException: 
    This stack walker does not have RETAIN_CLASS_REFERENCE access

The exception is very informative. When we call the `StackWalker.getInstance()` factory method, we get a `StackWalker` with default configuration which **would not retain class information**. In order to fix this, we can configure the `StackWalker` with `StackWalker.Option.RETAIN_CLASS_REFERENCE` option which instructs the `StackWalker` to retain `Class` information in each `StackFrame`:

{% highlight java %}
public class Foo {
    public static void whoIsIt() {
        StackWalker walker = StackWalker.getInstance(RETAIN_CLASS_REFERENCE);
        System.out.println(walker.getCallerClass());
    }
}
{% endhighlight %}
Running this modified version would print the following on console (Assuming we're calling `whoIsIt` from a method in class `me.alidg.Bar`):

    class me.alidg.Bar

### Let's go for a walk

---

`StackWalker` provides a `walk` method which enables us to traverse the stack frames without performing an eager capture of whole stack. The walk API takes a function that map the given `Stream<StackFrame>` to anything we want. For example, in order to find the caller of current executing method:
{% highlight java %}
Optional<? extends Class<?>> caller = walker.walk(frames -> 
                frames.skip(1).findFirst().map(StackFrame::getDeclaringClass)
        );
{% endhighlight %}
The `walk` method opens a sequential stream of `StackFrame`s for the current thread and then applies the function with the StackFrame stream.

[jep259]:http://openjdk.java.net/jeps/259