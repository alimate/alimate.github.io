---
layout: post
title: "On Generating Identity Hash Codes"
permalink: /blog/2020/7/15/hash-code
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-7-15-hash-code.md"
excerpt: "Have you ever wondered how does the HotSpot JVM generate identity hashcodes?"
image: https://alidg.me/images/15.jpg
subtitle: "she said: it's like they're going to Uranus!"
toc: true
---
This simple peice of code probably prints one of the most misunderstood outputs in Java:
{% highlight java %}
System.out.println(new Object());
{% endhighlight %}
And the output would be:
{% highlight bash %}
java.lang.Object@69663380
{% endhighlight %}
What does that part after `@` represent? Is this the memory address of the object? Is it the hashcode? How about both? or even none? 

Over the course of this article, we're going to see how the HotSpot JVM generates this value and what it represents. so, *Let's get started then!*

## Generation Strategies
---
As of this writing, HotSpot JVM has a few different strategies to generate [hashcodes](https://github.com/openjdk/jdk/blob/f8f35d30afacc4c581bdd5b749bb978020678917/src/hotspot/share/runtime/synchronizer.cpp#L942):
{% highlight cpp %}
static inline intptr_t get_next_hash(Thread* self, oop obj) {
  intptr_t value = 0;
  if (hashCode == 0) {
    // This form uses global Park-Miller RNG.
    // On MP system we'll have lots of RW access to a global, so the
    // mechanism induces lots of coherency traffic.
    value = os::random();
  } else if (hashCode == 1) {
    // This variation has the property of being stable (idempotent)
    // between STW operations.  This can be useful in some of the 1-0
    // synchronization schemes.
    intptr_t addr_bits = cast_from_oop<intptr_t>(obj) >> 3;
    value = addr_bits ^ (addr_bits >> 5) ^ GVars.stw_random;
  } else if (hashCode == 2) {
    value = 1;            // for sensitivity testing
  } else if (hashCode == 3) {
    value = ++GVars.hc_sequence;
  } else if (hashCode == 4) {
    value = cast_from_oop<intptr_t>(obj);
  } else {
    // Marsaglia's xor-shift scheme with thread-specific state
    // This is probably the best overall implementation -- we'll
    // likely make this the default in future releases.
    unsigned t = self->_hashStateX;
    t ^= (t << 11);
    self->_hashStateX = self->_hashStateY;
    self->_hashStateY = self->_hashStateZ;
    self->_hashStateZ = self->_hashStateW;
    unsigned v = self->_hashStateW;
    v = (v ^ (v >> 19)) ^ (t ^ (t >> 8));
    self->_hashStateW = v;
    value = v;
  }

  value &= markWord::hash_mask;
  if (value == 0) value = 0xBAD;
  assert(value != markWord::no_hash, "invariant");
  return value;
}
{% endhighlight %}
As shown above, the hashcode generation strategy is determined by a mysterious and yet well-named `hashCode` variable. Let's see where does this [variable](https://github.com/openjdk/jdk/blob/f8f35d30afacc4c581bdd5b749bb978020678917/src/hotspot/share/runtime/globals.hpp#L704) come from:
{% highlight cpp %}
experimental(intx, hashCode, 5, "(Unstable) select hashCode generation algorithm")  
{% endhighlight %}
So this `hashCode` variable is actually an experimental tuning flag with a default value of `5`:
{% highlight bash %}
$ java -XX:+UnlockExperimentalVMOptions -XX:+PrintFlagsFinal -version | grep hashCode
intx hashCode          = 5          {experimental} {default}
{% endhighlight %}
Put simply, we can use the `-XX:+UnlockExperimentalVMOptions -XX:hashCode=<i>` combination of tunables to change the strategy.

## Park–Miller/Lehmer RNG
---
The first hashcode generation strategy uses one of the most common random number generation strategies: a class of linear congruential generator (LCG) algorithms known as Lehmer RNG or even Park–Miller RNG:
{% highlight cpp %}
static inline intptr_t get_next_hash(Thread* self, oop obj) {
  intptr_t value = 0;
  if (hashCode == 0) {
    // This form uses global Park-Miller RNG.
    // On MP system we'll have lots of RW access to a global, so the
    // mechanism induces lots of coherency traffic.
    value = os::random();
  }
  // omitted
}
{% endhighlight %}

This algorithm starts with an initial *seed value*, <math xmlns="http://www.w3.org/1998/Math/MathML"><msub><mi>X</mi><mrow><mi>0</mi><mo>&#xA0;</mo></mrow></msub></math>. Then generates each random variable from the previous one as following:
<p>
<center>
<math xmlns="http://www.w3.org/1998/Math/MathML"><msub><mi>X</mi><mrow><mi>n</mi><mo>+</mo><mn>1</mn><mo>&#xA0;</mo></mrow></msub><mo>=</mo><mo>&#xA0;</mo><mo>(</mo><mi>a</mi><msub><mi>X</mi><mi>n</mi></msub><mo>&#xA0;</mo><mo>+</mo><mo>&#xA0;</mo><mi>c</mi><mo>)</mo><mo>&#xA0;</mo><mi>m</mi><mi>o</mi><mi>d</mi><mo>&#xA0;</mo><mi>m</mi></math>
</center></p>
Let's take a look at the `os::random()` [defintion](https://github.com/openjdk/jdk/blob/f8f35d30afacc4c581bdd5b749bb978020678917/src/hotspot/share/runtime/os.cpp#L853):
{% highlight cpp %}
volatile unsigned int os::_rand_seed = 1;
int os::random() {
  // Make updating the random seed thread safe.
  while (true) {
    unsigned int seed = _rand_seed;
    unsigned int rand = random_helper(seed);
    if (Atomic::cmpxchg(&_rand_seed, seed, rand) == seed) {
      return static_cast<int>(rand);
    }
  }
}
{% endhighlight %}
The `os::random()` tries to generate a random number from the global `_rand_seed` variable and then update that seed atomically. When multiple threads try to change the seed, only one of them can successfully change the seed and the `cmpxchg` will fail for others. Losing threads will retry the same operation until they succeed.

Therefore, in the presence of high contention, the rate of CAS failures will increase, hence this comment:
{% highlight cpp %}
// On MP system we'll have lots of RW access to a global, so the
// mechanism induces lots of coherency traffic.
{% endhighlight %}

## OOPs Based
---
The second and fifth strategies are using a function of memory address as the hashcode:
{% highlight cpp %}
else if (hashCode == 1) {
    // This variation has the property of being stable (idempotent)
    // between STW operations.  This can be useful in some of the 1-0
    // synchronization schemes.
    intptr_t addr_bits = cast_from_oop<intptr_t>(obj) >> 3;
    value = addr_bits ^ (addr_bits >> 5) ^ GVars.stw_random;
}
else if (hashCode == 4) {
    value = cast_from_oop<intptr_t>(obj);
}
{% endhighlight %}
So if use either of `-XX:hashCode=1` or `-XX:hashCode=4`, the hashcode will depend on the memory address.

## Static Number
---
Probably the coolest, least useful, and most efficient strategy is the third one:
{% highlight cpp %}
else if (hashCode == 2) {
    value = 1;            // for sensitivity testing
}
{% endhighlight %}
Which generates 1, all the time! If we run the same snippet with `-XX:+UnlockExperimentalVMOptions -XX:hashCode=2`:
{% highlight java %}
System.out.println(new Object());
{% endhighlight %}
Then it will print:
{% highlight bash %}
java.lang.Object@1
{% endhighlight %}
**So the part after `@` is always hashcode, at least!** I'm guessing they're using this strategy as a benchmark baseline. However, it's just a guess. If it's true, then this strategy ain't that useless after all.

## Sequential Numbers
---
The fourth strategy is basically an `auto-increment` for [hashcode generation](https://github.com/openjdk/jdk/blob/f8f35d30afacc4c581bdd5b749bb978020678917/src/hotspot/share/runtime/synchronizer.cpp#L845):
{% highlight cpp %}
struct SharedGlobals {
  // omitted
  DEFINE_PAD_MINUS_SIZE(1, DEFAULT_CACHE_LINE_SIZE, sizeof(volatile int) * 2);
  // Hot RW variable -- Sequester to avoid false-sharing
  volatile int hc_sequence;
  DEFINE_PAD_MINUS_SIZE(2, DEFAULT_CACHE_LINE_SIZE, sizeof(volatile int));
};
static SharedGlobals GVars;

// omitted
else if (hashCode == 3) {
    value = ++GVars.hc_sequence;
}
{% endhighlight %}
If we run the following code with `-XX:+UnlockExperimentalVMOptions -XX:hashCode=3`, we will probably see some consequetive hashcodes:
{% highlight java %}
System.out.println(new Object().hashCode()); // prints 317
System.out.println(new Object().hashCode()); // prints 318
System.out.println(new Object().hashCode()); // prints 319
{% endhighlight %}

## Marsaglia's Xor-Shift
---
As of this writing, if we pass anything more than 4 as the value of `-XX:hashCode`, this random number generator will be used:
{% highlight cpp %}
else {
    // Marsaglia's xor-shift scheme with thread-specific state
    // This is probably the best overall implementation -- we'll
    // likely make this the default in future releases.
    unsigned t = self->_hashStateX;
    t ^= (t << 11);
    self->_hashStateX = self->_hashStateY;
    self->_hashStateY = self->_hashStateZ;
    self->_hashStateZ = self->_hashStateW;
    unsigned v = self->_hashStateW;
    v = (v ^ (v >> 19)) ^ (t ^ (t >> 8));
    self->_hashStateW = v;
    value = v;
}
{% endhighlight %}
The implementation seems a bit complicated. However, the idea is simple. Instead of using some global shared mutable state as the seed, this is using a thread-specific state to generate the random number. Therefore, it will outperform the `os::random()` and the sequential approach, as there is no need for thread synchronization.

Currently, this is the default hasshcode generation strategy.

## Good Hashcodes
---
A hashcode implementation is a good one if it exhbits a uniform distrution and has a good performance. Let's evaluate each strategy with respect to these parameters:
 - The `os::random()` approach has a good uniformity and randomness. However, it won't perform that well in highly contended environments
 - The memory address based usually won't exhibit uniform distribution, which is very critical for hashcodes
 - The one that always returns 1 is fun!
 - The Marsaglia's Xor-Shift generates random numbers with good distribution and also, good performance

## Conclusion
---
Just to recap, the part after `@` **is definitly the identity hashcode**. The hashcode itself is usually a random number but can be a function of the object memory address. The identity hashcode, in the HotSpot JVM, consumes at most 31 bits of the object header, while the memory address may be up to 64 bits (No compressed OOPS). Therefore, the hashcode may not be equal to the memory address, even though it can be a function of it!

Before wrapping up, it's recommended to check out this [mailing list](http://mail.openjdk.java.net/pipermail/hotspot-runtime-dev/2013-January/005212.html) about the same subject.
