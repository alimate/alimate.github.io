---
layout: post
title: "Time Travel with JVM"
permalink: /blog/2020/2/23/time-travel-jvm
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-2-23-time-travel-jvm.md"
excerpt: "Let's allocate a Java object without calling its constructor!"
image: "https://www.crystalinks.com/wormhole716.jpg"
---
Let's repeat an ancient mistake:
{% highlight java %}
public final class Date {
    private final long timestamp;

    public Date() {
        timestamp = System.currentTimeMillis();
    }

    public long getTimestamp() {
        return timestamp;
    }
}
{% endhighlight %}
Our `Date` abstraction is supposed to mimic the infamous `java.util.Date` class. Unless we've cracked the time travel problem or our clock is way off, the `getTimestamp()` method *should never return 0*, right?

## "First Step"
---
Let's declare an instance of this `Date` class like a normal human being:
{% highlight java %}
Date now = new Date();
System.out.println(now.getTimestamp());
{% endhighlight %}
In my machine, in this very moment of writing, this prints:
{% highlight java %}
1582489721066
{% endhighlight %}
Sounds plausible!

## "Afraid of Time"
---
This time around, the `sun.misc.Unsafe` holds the *Answer to the Ultimate Question of Life, the Universe, and Everything:*
{% highlight java %}
private static Field getUnsafe() throws NoSuchFieldException {
    Field field = Unsafe.class.getDeclaredField("theUnsafe");
    field.setAccessible(true);

    return field;
}
{% endhighlight %}
Let's create an instance of `Date` via `Unsafe.allocateInstance(Class<?> clazz)` method:
{% highlight java %}
Date epoch = (Date) ((Unsafe) field.get(null)).allocateInstance(Date.class);
System.out.println(epoch.getTimestamp());
{% endhighlight %}
Quite surprisingly, this will print `0`, *because the `allocateInstance` method bypasses the Java constructor!*

By now one might question the integrity of the whole Java platform regarding `final` fields!

## "What Happens Now?"
---
Here's *what happens* when we instantiate a new object instance via `new` keyword in Java:
 1. At first, JVM tries to [find a place](https://alidg.me/blog/2019/6/21/tlab-jvm) for that new object in its process space.
 2. Then it executes the *System Initialization* process. In this phase, the newly created object would be initialized with its default values. For example, the `timestamp` value would be `0` at this stage.
 3. Finally, it calls the instance initializer and constructors. In this case, the constructor changes the `timestamp` value to the current *epoch* time.

For instance, the following code:
{% highlight java %}
Date now = new Date();
{% endhighlight %}
Transltes to the following bytecode:
{% highlight shell %}
0: new           #2  // class me/alidg/Date
3: dup
4: invokespecial #3 // Method me/alidg/Date."<init>":()V
{% endhighlight %}
The first part, i.e. `0: new #2  // class me/alidg/Date`, represents the system initialization phase. Additionally, the `invokespecial` is responsible for calling the constructor.

When using `Unsafe`, on the contrary, **the constructor call would be skipped**. Therefore, the `Date` instance would be initialized with its default values, *enabling us to travel back in time!*

## "<del>No</del> Time for Caution"
---
If we try to compile any code that exploits the `Unsafe` class, we might get the following warning from `javac`:
{% highlight shell %}
warning: Unsafe is internal proprietary API and may be removed in a future release
{% endhighlight %}
So stay away from `Unsafe`.
