---
layout: post
title: "Java Records: An Introduction"
permalink: /blog/2020/1/31/java14-records
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-1-31-java14-records.md"
excerpt: "A sneak peek at Records which are going to previewed in Java 14"
---
Java 14 is introducing a new preview feature called *Records*. Records provide a nice compact syntax to declare classes that are supposed to be *dumb data holders*. That may not sound much impressive but by taking a look at how we define such classes now, you might change your mind:
{% highlight java %}
public class Range {
    
    private final int min;
    private final int max;

    public Range(int min, int max) {
        this.min = min;
        this.max = max;
    }

    public int getMin() {
        return min;
    }

    public int getMax() {
        return max;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Range range = (Range) o;
        return min == range.min && max == range.max;
    }

    @Override
    public int hashCode() {
        return Objects.hash(min, max);
    }

    @Override
    public String toString() {
        return "Range{" +
          "min=" + min +
          ", max=" + max +
          '}';
    }
}
{% endhighlight %}
One might argue that writing these sort of boilerplates are a breeze with the help of our modern IDEs. However, we still need to read them, right? 

## Introducing Records
---
Anyway, we can rewrite the same code with Records like:
{% highlight java %}
public record Range(int min, int max) {}
{% endhighlight %}
Java compiler generates the following methods for this simple one-liner:
 - Simple methods to access *Record Components*, e.g. `min()` and `max()`.
 - A simple `equals` implementation that considers all record components while comparing.
 - An `equals` compatible implementation for `hashCode`.
 - A simple `toString` that simply prints all components.
 - and of course, a constructor!

Let's create a new record object:
{% highlight shell %}
jshell> var r = new Range(30, 42);
r ==> Range[min=30, max=42]
{% endhighlight %}
 Then we can have fun with it:
{% highlight shell %}
jshell> r.min()
$5 ==> 30

jshell> r.toString()
$6 ==> "Range[min=30, max=42]"

jshell> r
r ==> Range[min=30, max=42]
{% endhighlight %}
Records are **immutable** constructs, so we can't change them after construction! That is, Javac doesn't generate *setter* methods for records.

## Preview Feature
---
As we mentioned earlier, Records are part of the preview features in Java 14. So we should use the `--enable-preview` flag while compiling Records:
{% highlight shell %}
> javac --enable-preview -source 14 Person.java
{% endhighlight %}
Same is true when we're going use Records in JShell:
{% highlight shell %}
> jshell --enable-preview
{% endhighlight %}

## Customizing Records
---
Except for a few limitations, Records are normal Java classes. Therefore, we can add members or logics just like what we did with normal classes. For example, in order to enforce a pre-condition while constructing a Record instance:
{% highlight java %}
public record Range(int min, int max) {
    public Range { // no parameter list here!
        if (min > max) 
            throw new IllegalArgumentException("Min should be less than or equal to max");
    }
}
{% endhighlight %}
When we omit the parameter list in a custom constructor, then we're practically enhancing the primary constructor. Suppose we violate this pre-condition:
{% highlight shell %}
jshell> var r = new Range(30, 12);
|  Exception java.lang.IllegalArgumentException: Min should be less than or equal to max
|        at Range.<init> (#8:3)
|        at do_it$Aux (#9:1)
|        at (#9:1)
{% endhighlight %}
It's also possible to declare instance or static methods on Records:
{% highlight java %}
public record Range(int min, int max) {
    public boolean contains(int number) {
        return number >= min && number <= max;
    }
}
{% endhighlight %}
The invocation is just like normal instance or static methods:
{% highlight shell %}
jshell> var r = new Range(30, 42)
r ==> Range[min=30, max=42]

jshell> r.contains(29)
$12 ==> false

jshell> r.contains(32)
$13 ==> true
{% endhighlight %}
It's **not possible to declare instance fields inside Records**. This is a by-design limitation to simplify the reasoning about Record's state. As opposed to instance fields, it's perfectly fine to declare static fields inside records.

## Wrapping Up
---
This was a very gentle introduction to [JEP 359](https://openjdk.java.net/jeps/359). I'm going to work on another more in-depth article about Records, their class representation, indy, and other more low-level details. So stay tuned!