---
layout: post
title: "Thread Local Randoms in Java"
permalink: /blog/2020/4/24/thread-local-random
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-4-24-thread-local-random.md"
excerpt: "Benchmarking regular randoms against thread-local ones!"
image: https://alidg.me/images/random.jpg
subtitle: "Any one who considers arithmetical methods of producing random digits is, of course, in a state of sin <br>-- John Von Neumann"
---
Java 7 introduced the [`ThreadLocalRandom`](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/9b8c96f96a0f/src/share/classes/java/util/concurrent/ThreadLocalRandom.java#l64) to improve the random number generation throughput in highly contended environments. 

The rational behind `ThreadLocalRandom` is simple, instead of sharing one global `Random` instance, each thread maintains its own version of `Random`. This, in turn, reduces the contention and hence the better throughput.

Since this is such a simple idea, we should be able to roll our sleeves up and implement something like `ThreadLocalRandom` with a comparable performance, *right?*

*Let's see.*

## First Attempt
---
In our first attempt, we're going to use something as simple as `ThreadLocal<Random>`:
{% highlight java %}
// A few annotations
public class RandomBenchmark {

    private final Random random = new Random();
    private final ThreadLocal<Random> simpleThreadLocal = ThreadLocal.withInitial(Random::new);

    @Benchmark
    @BenchmarkMode(Throughput)
    public int regularRandom() {
        return random.nextInt();
    }

    @Benchmark
    @BenchmarkMode(Throughput)
    public int simpleThreadLocal() {
        return simpleThreadLocal.get().nextInt();
    }

    @Benchmark
    @BenchmarkMode(Throughput)
    public int builtinThreadLocal() {
        return ThreadLocalRandom.current().nextInt();
    }

    // omitted
}
{% endhighlight %}
In this benchmark, we're comparing the `Random`, our own `ThreadLocal<Random>` and the builtin `ThreadLocalRandom`:
{% highlight text %}
Benchmark                             Mode  Cnt           Score          Error  Units
RandomBenchmark.builtinThreadLocal   thrpt   40  1023676193.004 ± 26617584.814  ops/s
RandomBenchmark.regularRandom        thrpt   40     7487301.035 ±   244268.309  ops/s
RandomBenchmark.simpleThreadLocal    thrpt   40   382674281.696 ± 13197821.344  ops/s
{% endhighlight %}
The `ThreadLocalRandom` generated **around 1 billion random numbers per second**, dwarfing the `ThreadLocal<Random>` and plain `Random` approaches by a whopping margin.

## The Linear Congruential Method
---
By far the most popular random number generator in use today is known as *Linear Congruential Pseudorandom Number Generator*, introduced by D. H. Lehmer in 1949.

This algorithm starts with an initial *seed value*, <math xmlns="http://www.w3.org/1998/Math/MathML"><msub><mi>X</mi><mrow><mi>0</mi><mo>&#xA0;</mo></mrow></msub></math>. Then generates each random variable from the previous one as following:
<p>
<center>
<math xmlns="http://www.w3.org/1998/Math/MathML"><msub><mi>X</mi><mrow><mi>n</mi><mo>+</mo><mn>1</mn><mo>&#xA0;</mo></mrow></msub><mo>=</mo><mo>&#xA0;</mo><mo>(</mo><mi>a</mi><msub><mi>X</mi><mi>n</mi></msub><mo>&#xA0;</mo><mo>+</mo><mo>&#xA0;</mo><mi>c</mi><mo>)</mo><mo>&#xA0;</mo><mi>m</mi><mi>o</mi><mi>d</mi><mo>&#xA0;</mo><mi>m</mi></math>
</center></p>
In the real world, it's more practical to use just one variable to implement this. So, we should update the `seed` value every time we generate a number. 
Quite interestingly, the [`next(int)`](https://github.com/openjdk/jdk14u/blob/master/src/java.base/share/classes/java/util/Random.java#L198) method in the `Random` class is responsible for this:
{% highlight java %}
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
{% endhighlight %}
Since multiple threads can potentially update the `seed` value concurrently, we need some sort of *synchronization* to coordinate the concurrent access. Here Java is using a *Lock Free* approach with the help of *atomics*.

Basically, each thread tries to change the seed value to a new one atomically via `compareAndSet`. If a thread fails to do so, it will retry the same process until it could successfully commit the update.

When the contention is high, the number of <span title="Comapre and Set">CAS</span> failures increases. That's the main reason behind the poor performance of `Random` in concurrent environemnts.

## No More CAS
---
In implementations based on `ThreadLocal`, **the seed values are confined to each thread**. Therefore, we don't need atomics or any other form of synchronization for that matter as there is no *shared mutable state*:
{% highlight java %}
public class MyThreadLocalRandom extends Random {

    // omitted
    private static final ThreadLocal<MyThreadLocalRandom> threadLocal = 
        ThreadLocal.withInitial(MyThreadLocalRandom::new);

    private MyThreadLocalRandom() {}

    public static MyThreadLocalRandom current() {
        return threadLocal.get();
    }

    @Override
    protected int next(int bits) {
        seed = (seed * multiplier + addend) & mask;
        return (int) (seed >>> (48 - bits));
    }
}
{% endhighlight %}
If we run the same benchmark again:
{% highlight text %}
Benchmark                             Mode  Cnt           Score          Error  Units
RandomBenchmark.builtinThreadLocal   thrpt   40  1023676193.004 ± 26617584.814  ops/s
RandomBenchmark.lockFreeThreadLocal  thrpt   40   695217843.076 ± 17455041.160  ops/s
RandomBenchmark.regularRandom        thrpt   40     7487301.035 ±   244268.309  ops/s
RandomBenchmark.simpleThreadLocal    thrpt   40   382674281.696 ± 13197821.344  ops/s
{% endhighlight %}
`MyThreadLocalRandom` has almost twice the throughput of our simple `ThreadLocal<Random>`. 

The `compareAndSet` provides atomicity and memory ordering guarantees that we simply don't need in a thread confined context. Since those guarantees are costly and unnecessary, removing them increases the throughput substantially.

*However, we're still behind the builtin `ThreadLocalRandom`*!

## Removing Indirection
---
**As it turns out, each thread doesn't need a distinct and complete copy of `Random` for itself. It just needs the latest `seed` value, that's it**.

As of [Java 8](https://github.com/openjdk/jdk14u/blob/master/src/java.base/share/classes/java/lang/Thread.java#L2059), these values have been added to the `Thread` class itself:
{% highlight java %}
/** The current seed for a ThreadLocalRandom */
@jdk.internal.vm.annotation.Contended("tlr")
long threadLocalRandomSeed;

/** Probe hash value; nonzero if threadLocalRandomSeed initialized */
@jdk.internal.vm.annotation.Contended("tlr")
int threadLocalRandomProbe;

/** Secondary seed isolated from public ThreadLocalRandom sequence */
@jdk.internal.vm.annotation.Contended("tlr")
int threadLocalRandomSecondarySeed;
{% endhighlight %}
Instead of using something like `MyThreadLocalRandom`, each thread instance maintains its current `seed` value inside the `threadLocalRandomSeed` variable. 
As a result, the `ThreadLocalRandom` class becomes a [singleton](https://github.com/openjdk/jdk14u/blob/89deef4dd8b7aac7c3cea6e13c494a438d34d4c4/src/java.base/share/classes/java/util/concurrent/ThreadLocalRandom.java#L1077):
{% highlight java %}
/** The common ThreadLocalRandom */
static final ThreadLocalRandom instance = new ThreadLocalRandom();
{% endhighlight %}
Every time we call the `ThreadLocalRandom.current()`, it [lazily initializes](https://github.com/openjdk/jdk14u/blob/master/src/java.base/share/classes/java/util/concurrent/ThreadLocalRandom.java#L176) the `threadLocalRandomSeed` and then returns that singelton:
{% highlight java %}
public static ThreadLocalRandom current() {
    if (U.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
{% endhighlight %}

### Hashtable Lookup
With `MyThreadLocalRandom`, every call to the `current()` factory method, translates to a hash value computation for the `ThreadLocal` instance and a lookup in the underlying hashtable.

On the contrary, with this new Java 8+ approach all we have to do is to read the `threadLocalRandomSeed` value directly and update it afterward.
### The Unsafe
TODO

## False Sharing
---
TODO

## Conclusion
---
TODO