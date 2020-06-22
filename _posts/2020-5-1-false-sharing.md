---
layout: post
title: "False Sharing"
permalink: /blog/2020/5/1/false-sharing
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-5-1-false-sharing.md"
excerpt: "Measuring false-sharing effect on latency"
image: https://alidg.me/images/cpu.png
toc: true
---
Sometimes the most innocent looking pieces of codes can hurt the overall latency or throughput of the system. In this article, we're going to see **how *false sharing* can turn *multithreading* against us**.

Let's start with a simple contract for *counting:*
{% highlight java %}
public interface Counter {
    long inc(int index);
}
{% endhighlight %}
Each implementation encapsulates a collection of counters. The `inc` method is responsible for incrementing the counter at the given `index`.

## No Contention
---
Let's start with 8 distinct counters:
{% highlight java %}
public class SimpleCounter implements Counter {
    private volatile long v1;
    private volatile long v2;
    private volatile long v3;
    private volatile long v4;
    private volatile long v5;
    private volatile long v6;
    private volatile long v7;
    private volatile long v8;

    // omitted
}
{% endhighlight %}
We're going to increment each of these values atomically. Also, to avoid the extra memory footprint associated with the `AtomicLong`s, we will use `VarHandle`s [here](http://gee.cs.oswego.edu/dl/html/j9mm.html):
{% highlight java %}
private static final VarHandle V1;
private static final VarHandle V2;
// one VarHandle per counter

static {
    try {
        V1 = MethodHandles.lookup().findVarHandle(SimpleCounter.class, "v1", long.class);
        V2 = MethodHandles.lookup().findVarHandle(SimpleCounter.class, "v2", long.class);
        // omitted
    } catch (Exception e) {
        throw new ExceptionInInitializerError(e);
    }
}
{% endhighlight %}
The `inc(int index)` method increments the counter at the given `index`:
{% highlight java %}
public long inc(int index) {
    switch (index) {
        case 1: return (long) V1.getAndAdd(this, 1);
        case 2: return (long) V2.getAndAdd(this, 1);
        // omitted
    }

    return 0;
}
{% endhighlight %}

## First Benchmark
---
The following benchmark would run the `inc` method with 8 threads under heavy load. Each thread is going to call the method with a distinct `index`. Therefore, **there won't be any contention as the thread `i` operates on the *ith* counter**:
{% highlight java %}
@Threads(8)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class FalseSharingVictimBenchmark {

    private static final SimpleCounter simple = new SimpleCounter();
    private static final AtomicInteger threadCounter = new AtomicInteger(1);
    private static final ThreadLocal<Integer> id = ThreadLocal
        .withInitial(threadCounter::getAndIncrement)

    @Benchmark
    @BenchmarkMode(AverageTime)
    public long simple() {
        return simple.inc(id.get() % 8);
    }
}
{% endhighlight %}
Running this benchmark would result in something like:
{% highlight plain %}
Result "me.alidg.FalseSharingVictimBenchmark.simple":
  170.423 ±(99.9%) 4.913 ns/op [Average]
  (min, avg, max) = (155.715, 170.423, 187.665), stdev = 10.988
  CI (99.9%): [165.510, 175.335] (assumes normal distribution)
{% endhighlight %}
On average each `inc` call takes about 170 nano seconds to complete.

*Let's see if we can improve this situation.*
## Know Your Object Layout
---
If we run the following:
{% highlight java %}
ClassLayout.parseClass(SimpleCounter.class).toPrintable()
{% endhighlight %}
Then [JOL](http://openjdk.java.net/projects/code-tools/jol/) will print the object layout for the `SimpleCounter`:
{% highlight plain %}
me.alidg.SimpleCounter object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0    12        (object header)                           N/A
     12     4        (alignment/padding gap)                  
     16     8   long SimpleCounter.v1                          N/A
     24     8   long SimpleCounter.v2                          N/A
     32     8   long SimpleCounter.v3                          N/A
     40     8   long SimpleCounter.v4                          N/A
     48     8   long SimpleCounter.v5                          N/A
     56     8   long SimpleCounter.v6                          N/A
     64     8   long SimpleCounter.v7                          N/A
     72     8   long SimpleCounter.v8                          N/A
Instance size: 80 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
{% endhighlight %}
We can visualize this layout as something like:
<p style="text-align:center">
  <img src="/images/simple-counter-ol.svg" alt="Object Layout for SimpleCounter">
</p>
**It follows the typical *OOP* or *Ordinay Object Pointer* structure in [JVM](https://www.baeldung.com/jvm-compressed-oops#1-object-memory-layout): an object header immediately followed by zero or more references to instance fields.** The header itself consists of one mark word, one *klass* word, and a 32-bit array length which is 12 bytes in total. Also, JVM may add up to 32-bits of padding for alignment purposes.

Immediately after object header and padding, there are 8 references to those 8 `long` instance fields.

We can also verify this layout using the lovely `sun.misc.Unsafe`:
{% highlight java %}
private final SimpleCounter counter = new SimpleCounter();
private final Unsafe unsafe = getUnsafe();

@Test
void verifyingSimpleCounterMemoryLayout() {
    assertEquals(16, offsetOf("v1"));
    assertEquals(24, offsetOf("v2"));
    assertEquals(32, offsetOf("v3"));
    assertEquals(40, offsetOf("v4"));
    assertEquals(48, offsetOf("v5"));
    assertEquals(56, offsetOf("v6"));
    assertEquals(64, offsetOf("v7"));
    assertEquals(72, offsetOf("v8"));
}

private long offsetOf(String fieldName) {
    return unsafe.objectFieldOffset(getField(counter.getClass(), fieldName));
}
{% endhighlight %}
## Padding Effect
---
Let's see how adding some paddings between counters affect the latency. First, isolating the counters from the object header by adding 6 `long` values before `v1`:
{% highlight java %}
public class Padded1Counter implements Counter {

    // Isolating v1 from the object header and 4 bytes of padding
    private long p01, p02, p03, p04, p05, p06 = 0;
    private volatile long v1;

    // omitted
}
{% endhighlight %}
6 `long` values plus 12 bytes of object header plus 4 bytes of padding added by JVM is equal to 64 bytes (More on this 64 magic number later).
Then we're going to separate each counter by one `long` value:
{% highlight java %}
public class Padded1Counter implements Counter {

    private long p01, p02, p03, p04, p05, p06 = 0;

    private volatile long v1;
    private long p11 = 0;

    private volatile long v2;
    private long p21 = 0;

    // repeat this
}
{% endhighlight %}
Here's how JVM lays out the `Padded1Counter` in heap:
<p style="text-align:center">
  <img src="/images/padded1-ol.svg" alt="Object Layout for Padded1Counter">
</p>
If we benchmark `Padded1Counter` against the `SimpleCounter`:
{% highlight plain %}
Benchmark                              Mode  Cnt    Score   Error  Units
FalseSharingVictimBenchmark.padded1    avgt   60  120.444 ± 4.263  ns/op
FalseSharingVictimBenchmark.simple     avgt   60  170.423 ± 4.913  ns/op
{% endhighlight %}
Quite surprisingly, the `Padded1Counter` outperforms the `SimpleCounter` in terms of latency.

Let's add another 8 bytes between counters:
{% highlight java %}
public class Padded2Counter implements Counter {

    private long p01, p02, p03, p04, p05, p06 = 0;
    private volatile long v1;
    private long p11, p12 = 0;

    private volatile long v2;
    private long p21, p22 = 0;

    // you get the idea
}
{% endhighlight %}
The memory layout turns out to be like this:
<p style="text-align:center">
  <img src="/images/padded2-ol.svg" alt="Object Layout for Padded1Counter">
</p>
And the benchmark result is again interesting: 
{% highlight plain %}
Benchmark                              Mode  Cnt    Score   Error  Units
FalseSharingVictimBenchmark.padded1    avgt   60  120.444 ± 4.263  ns/op
FalseSharingVictimBenchmark.padded2    avgt   60  107.887 ± 2.729  ns/op
FalseSharingVictimBenchmark.simple     avgt   60  170.423 ± 4.913  ns/op
{% endhighlight %}
If we continue to add more paddings until we reach 7 bytes of padding, the memory layout would as isolated as:
<p style="text-align:center">
  <img src="/images/padded7-ol.svg" alt="Object Layout for Padded1Counter">
</p>
Each counter resides on its own 64 bytes isolated area. Let's see how's the benchmark looks like after all this:
{% highlight plain %}
Benchmark                              Mode  Cnt    Score   Error  Units
FalseSharingVictimBenchmark.padded1    avgt   60  120.444 ± 4.263  ns/op
FalseSharingVictimBenchmark.padded2    avgt   60  107.887 ± 2.729  ns/op
FalseSharingVictimBenchmark.padded3    avgt   60   91.285 ± 2.015  ns/op
FalseSharingVictimBenchmark.padded4    avgt   60   88.319 ± 3.045  ns/op
FalseSharingVictimBenchmark.padded5    avgt   60   84.910 ± 4.142  ns/op
FalseSharingVictimBenchmark.padded6    avgt   60   72.000 ± 6.060  ns/op
FalseSharingVictimBenchmark.padded7    avgt   60   65.932 ± 3.191  ns/op
FalseSharingVictimBenchmark.simple     avgt   60  170.423 ± 4.913  ns/op
{% endhighlight %}

## Performance Monitoring Counter
---
To instrument low-level CPU events, such as cycles, stall cycles, instructions, or memory loads/stores, one can program special hardware registers on the processors. 

As it turns out, tools like *perf* or *BPF* are already using this approach to expose useful metrics. As of Linux 2.6.31, perf is the standard Linux profiler capable of exposing useful *Performance Monitoring Counters* or *PMCs*.

We can *perf_events* to see what's going on at the CPU level when running each of those benchmarks. For instance, if we run:
{% highlight bash %}
>> perf stat -d java -jar benchmarks.jar simple 
{% endhighlight %}
Then `perf` would run the benchmark against the `SimpleCounter` and then expose some CPU PMCs as follows:
{% highlight plain %}
161657.133662      task-clock (msec)         #    3.951 CPUs utilized
         9321      context-switches          #    0.058 K/sec
          185      cpu-migrations            #    0.001 K/sec
        20514      page-faults               #    0.127 K/sec
            0      cycles                    #    0.000 GHz
 219476182640      instructions
  44787498110      branches                  #  277.052 M/sec
     37831175      branch-misses             #    0.08% of all branches
  91534635176      L1-dcache-loads           #  566.227 M/sec
   1036004767      L1-dcache-load-misses     #    1.13% of all L1-dcache hits
{% endhighlight %}
The `SimpleCounter` benchmark had 1,036,004,767 (~ 1 billion) L1 data cache misses. If we run the same profiling with the `Padded7Counter`:
{% highlight plain %}
161742.243922      task-clock (msec)         #    3.955 CPUs utilized
         9041      context-switches          #    0.056 K/sec
          220      cpu-migrations            #    0.001 K/sec
        21678      page-faults               #    0.134 K/sec
            0      cycles                    #    0.000 GHz
 692586696913      instructions
 138097405127      branches                  #  853.812 M/sec
     39010267      branch-misses             #    0.03% of all branches
 291832840178      L1-dcache-loads           # 1804.308 M/sec
    120239626      L1-dcache-load-misses     #    0.04% of all L1-dcache hits
{% endhighlight %} 
The `Padded7Counter` encouuntered 120,239,626 cache misses, almost 900 million less cache misses compared to the simple approach.

What is the cause of such a difference between almost two identical implementations?

## Cache Line and Coherency
---

## False Sharing
---

## How Much Padding?
---

## @Contended
---

## Conclusion
---
