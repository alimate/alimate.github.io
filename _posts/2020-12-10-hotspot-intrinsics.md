---
layout: post
title: "HotSpot Intrinsics"
permalink: /blog/2020/12/10/hotspot-intrinsics
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-12-10-hotspot-intrinsics.md"
excerpt: "How Does the HotSpot JVM Intrinsify Some Method Implementations?"
image: https://alidg.me/images/observer.png
toc: true
---
Sometimes, compilers have special treatments for some function implementations. Put simply, they replace the default implementation with another, possibly optimized, implementation. Such functions are known as *intrinsic functions* in compiler theory.

In this article, we'll walk through a few examples to see how intrinsic functions are working in the HotSpot JVM.

## A Tale of Two Logs
---
The `Math.log()` method in Java computes the *natural logarithm* of any given number. Typical high school stuff, nothing fancy! Here's what the implementation of this method looks like in [OpenJDK](https://github.com/openjdk/jdk/blob/5f0334121153332873f952fdc6828b9926171bbe/src/java.base/share/classes/java/lang/Math.java#L310):
{% highlight java %}
@IntrinsicCandidate
public static double log(double a) {
    return StrictMath.log(a); // default impl. delegates to StrictMath
}
{% endhighlight %}
As shown above, the `Math.log()` method itself calls another method called [`StrictMath.log()`](https://github.com/openjdk/jdk/blob/5f0334121153332873f952fdc6828b9926171bbe/src/java.base/share/classes/java/lang/StrictMath.java#L247) under the hood. Despite this delegation, we usually tend to use the `Math.log()` instead of the strict and more direct one!

## Premature-ization
---
Despite Donald Knuth's efforts, one might propose to use the `StrictMath` implementation, mainly to avoid the *unnecessary* indirection and being more *sympathetic* to the underlying *mechanics!*

Well, we all know that when the `Math.log()` method *gets hot enough* (i.e. being called frequently enough), then the HotSpot JVM will inline this delegation. Therefore, it's only natural to expect that both method calls exhibit similar performance characteristics, *at least when the performance matters!*.

To prove this hypothesis, let's conduct a simple benchmark comparing the two implementations:
{% highlight java %}
@State(Scope.Benchmark)
@BenchmarkMode(Mode.Throughput)
public class IntrinsicsBenchmark {

    @Param("12346545756.54634")
    double value;

    @Benchmark
    public double indirect() {
        return Math.log(value); // Calls the StrictMath.log(value) under the hood.
    }

    @Benchmark
    public double direct() {
        return StrictMath.log(value);
    }

    // typical stuff
}
{% endhighlight %}
*The result should be so predictable, right?*

## The Observer Effect
---
If we package the benchmark and run the following command:
{% highlight bash %}
>> java -jar intrinsics.jar -f 2 -t 8
{% endhighlight %}
After a while, JMH will print the benchmark result like the following:
{% highlight plain %}
Benchmark                               (value)   Mode  Cnt          Score          Error  Units
IntrinsicsBenchmark.direct    12346545756.54634  thrpt   20  151571897.277 ±  7878104.343  ops/s
IntrinsicsBenchmark.indirect  12346545756.54634  thrpt   20  309745064.598 ± 12678366.349  ops/s
{% endhighlight %}
*We didn't see that coming, did we?!* The indirect `Math.log()` implementation **outperforms the direct and supposedly more performant implementation by almost 105% in terms of throughput!**

## Pandora's Box
---
Let's take a closer look at the `Math.log()` implementation once again, just to make sure we didn't missed something there:
{% highlight java %}
@IntrinsicCandidate
public static double log(double a) {
    return StrictMath.log(a); // default impl. delegates to StrictMath
}
{% endhighlight %}
The delegation exists, for sure. Quite interestingly, there is also a `@IntrinsicCandidate` annotation on the method. Before going any further, it's worth mentioning that before Java 16, the same method did look like this:
{% highlight java %}
@HotSpotIntrinsicCandidate
public static double log(double a) {
    return StrictMath.log(a); // default impl. delegates to StrictMath
}
{% endhighlight %}
So basically, as of Java 16, the `jdk.internal.HotSpotIntrinsicCandidate` is repackaged and renamed as `jdk.internal.vm.annotation.IntrinsicCandidate`.

Anyway, the `@IntrinsicCandidate` may reveal the actual reason behind this shocking benchmark result. Let's take a peek at the annotation [Javadoc](https://github.com/openjdk/jdk/blob/fd5f6e2e196f1b83c15e802d71aa053d9feab20f/src/java.base/share/classes/jdk/internal/vm/annotation/IntrinsicCandidate.java#L124):
{% highlight java %}
/**
 * The {@code @IntrinsicCandidate} annotation is specific to the
 * HotSpot Virtual Machine. It indicates that an annotated method
 * may be (but is not guaranteed to be) intrinsified by the HotSpot VM. A method
 * is intrinsified if the HotSpot VM replaces the annotated method with hand-written
 * assembly and/or hand-written compiler IR -- a compiler intrinsic -- to improve
 * performance. The {@code @IntrinsicCandidate} annotation is internal to the
 * Java libraries and is therefore not supposed to have any relevance for application
 * code.
 *
 * @since 16
 */
@Target({ElementType.METHOD, ElementType.CONSTRUCTOR})
@Retention(RetentionPolicy.RUNTIME)
public @interface IntrinsicCandidate {
}
{% endhighlight %}
Well, based on this, **the HotSpot JVM *may* replace the `Math.log()` Java implementation with a possibly more efficient compiler intrinsic to improve the performance.**

## Down the Rabbit Hole
---
As it turns out, there actually is an intrinsic for the `Math.log()` method!

The HotSpot JVM defines all its intrinsics in the [`vmIntrinsics.hpp`](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/classfile/vmIntrinsics.hpp) file<sup>1</sup>. In the HotSpot, there are two types of intrinsics:
 - Library intrinsics: These are typical compiler intrinsics as they will replace the method implementations.
 - Bytecode intrinsics: These methods won't be replaced but instead would have special treatments.

The HotSpot JVM source code [documents](https://github.com/openjdk/jdk/blob/46c9a860b6516c119f11edee2deb3d9f62907323/src/hotspot/share/classfile/vmIntrinsics.hpp#L72) these two types as follows:
{% highlight cpp %}
// There are two types of intrinsic methods: (1) Library intrinsics and (2) bytecode intrinsics.
//
// (1) A library intrinsic method may be replaced with hand-crafted assembly code,
// with hand-crafted compiler IR, or with a combination of the two. The semantics
// of the replacement code may differ from the semantics of the replaced code.
//
// (2) Bytecode intrinsic methods are not replaced by special code, but they are
// treated in some other special way by the compiler. For example, the compiler
// may delay inlining for some String-related intrinsic methods (e.g., some methods
// defined in the StringBuilder and StringBuffer classes, see
// Compile::should_delay_string_inlining() for more details).
{% endhighlight %}
Right after this, they list all the possible [VM intrinsics](https://github.com/openjdk/jdk/blob/46c9a860b6516c119f11edee2deb3d9f62907323/src/hotspot/share/classfile/vmIntrinsics.hpp#L149) one after another. For instance:
{% highlight cpp %}
// Here are all the intrinsics known to the runtime and the CI.
// omitted
/* Math & StrictMath intrinsics are defined in terms of just a few signatures: */           \
do_class(java_lang_Math,                "java/lang/Math") 
/* here are the math names, all together: */                                                \
do_name(abs_name,"abs")       do_name(sin_name,"sin")         do_name(cos_name,"cos")       \
do_name(tan_name,"tan")       do_name(atan2_name,"atan2")     do_name(sqrt_name,"sqrt")     \
do_name(log_name,"log")       do_name(log10_name,"log10")     do_name(pow_name,"pow")       \
do_name(exp_name,"exp")       do_name(min_name,"min")         do_name(max_name,"max")       \
do_name(floor_name, "floor")  do_name(ceil_name, "ceil")      do_name(rint_name, "rint")
do_intrinsic(_dlog, java_lang_Math, log_name, double_double_signature, F_S)
{% endhighlight %}
As shown by the last line, there is actually an intrinsic replacement for the `Math.log()`. For instance, on x86-64 architectures, the `Math.log()` will be *intrinsified* as [follows](https://github.com/openjdk/jdk/blob/0a3e446ad95b09de2facee7107f7c1206339ee0d/src/hotspot/cpu/x86/stubGenerator_x86_64.cpp#L6335):
{% highlight cpp %}
if (vmIntrinsics::is_intrinsic_available(vmIntrinsics::_dlog)) {
    StubRoutines::_dlog = generate_libmLog();
}

// the generator
address generate_libmLog() {
    StubCodeMark mark(this, "StubRoutines", "libmLog");

    address start = __ pc();

    const XMMRegister x0 = xmm0;
    const XMMRegister x1 = xmm1;
    const XMMRegister x2 = xmm2;
    const XMMRegister x3 = xmm3;

    const XMMRegister x4 = xmm4;
    const XMMRegister x5 = xmm5;
    const XMMRegister x6 = xmm6;
    const XMMRegister x7 = xmm7;

    const Register tmp1 = r11;
    const Register tmp2 = r8;

    BLOCK_COMMENT("Entry:");
    __ enter(); // required for proper stackwalking of RuntimeStub frame

    __ fast_log(x0, x1, x2, x3, x4, x5, x6, x7, rax, rcx, rdx, tmp1, tmp2);

    __ leave(); // required for proper stackwalking of RuntimeStub frame
    __ ret(0);

    return start;

}
{% endhighlight %}
The `vmIntrinsics.hpp` only defines the fact that some methods may have intrinsic implementations. The actual intrinsic routine is provided somewhere else and usually depends on the underlying architecture. In the above example, the `src/hotspot/cpu/x86/stubGenerator_x86_64.cpp` is responsible for providing the actual intrinsic for the 64-bit x86 architecture.

In addition to being architecture-specific, intrinsics can be disabled. Therefore, [the JVM compiler](https://github.com/openjdk/jdk/blob/0a3e446ad95b09de2facee7107f7c1206339ee0d/src/hotspot/share/compiler/abstractCompiler.hpp#L133) (C1 or C2) checks these two conditions before applying the intrinsic:
{% highlight cpp %}
virtual bool is_intrinsic_available(const methodHandle& method, DirectiveSet* directive) {
    return is_intrinsic_supported(method) &&
           !directive->is_intrinsic_disabled(method) &&
           !vmIntrinsics::is_disabled_by_flags(method);
}
{% endhighlight %}
Basically, an intrinsic is available if:
 - The intrinsic is enabled, usually by using a tunable flag.
 - The underlying platform supports the intrinsic.

Let's see more about those tunables.

## Tunables
---
Similar to many other aspects of the JVM, we can control the intrinsics to some extent using tunable flags. 

For starters, the combination of `-XX:+UnlockDiagnosticVMOptions` and `-XX:+PrintIntrinsics` make the HotSpot to print all intrinsics while introducing them. For instance, if we run the same benchmark with these flags, we will see a lot of `Math.log()` related *logs:*
{% highlight bash %}
>> java -XX:+UnlockDiagnosticVMOptions -XX:+PrintIntrinsics -jar intrinsics.jar -f 2 -t 8
// truncated logs
@ 4   java.lang.Math::log (5 bytes)   (intrinsic)
@ 4   java.lang.Math::log (5 bytes)   (intrinsic)
@ 55  java.lang.Math::min (11 bytes)   (intrinsic)
@ 58  java.lang.System::arraycopy (0 bytes)   (intrinsic)
{% endhighlight %}
Also, we can disable all the Math related intrinsics using the `-XX:-InlineMathNatives` tunable:
{% highlight bash %}
>> java -XX:+UnlockDiagnosticVMOptions -XX:-InlineMathNatives -jar intrinsics.jar -f 1 -t 8
Benchmark                               (value)   Mode  Cnt          Score          Error  Units
IntrinsicsBenchmark.direct    12346545756.54634  thrpt   20  171611762.349 ±  4203913.645  ops/s
IntrinsicsBenchmark.indirect  12346545756.54634  thrpt   20  169765587.934 ±  9555128.466  ops/s
{% endhighlight %}
As shown above, since the JVM no longer applies the intrinsics for the `Math.log()`, the throughputs are almost the same!

Using a simple `grep`, as always, we can see all the tunables related to a particular subject:
{% highlight bash %}
>> java -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -version | grep Intrinsic
bool CheckIntrinsics                          = true                                   
ccstrlist DisableIntrinsic                    =                              
bool PrintIntrinsics                          = false                               
bool UseAESCTRIntrinsics                      = true                                   
bool UseAESIntrinsics                         = true                                 
bool UseAdler32Intrinsics                     = false                               
bool UseBASE64Intrinsics                      = false                                    
bool UseCRC32CIntrinsics                      = true                                   
bool UseCRC32Intrinsics                       = true                                   
bool UseCharacterCompareIntrinsics            = false                               
bool UseGHASHIntrinsics                       = true                                   
bool UseLibmIntrinsic                         = true                            
bool UseMathExactIntrinsics                   = true                               
bool UseMontgomeryMultiplyIntrinsic           = true                               
bool UseMontgomerySquareIntrinsic             = true                                
bool UseMulAddIntrinsic                       = true                               
bool UseMultiplyToLenIntrinsic                = true                                
bool UseSHA1Intrinsics                        = false                                  
bool UseSHA256Intrinsics                      = true                                   
bool UseSHA512Intrinsics                      = true                                 
bool UseSSE42Intrinsics                       = true                                 
bool UseSquareToLenIntrinsic                  = true                               
bool UseVectorizedMismatchIntrinsic           = true                                  
{% endhighlight %}
And, one more thing:
{% highlight bash %}
>> java -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -version | grep Native
bool CriticalJNINatives                       = true                                     
bool InlineClassNatives                       = true                                
bool InlineMathNatives                        = true                                  
bool InlineNatives                            = true                                   
bool InlineThreadNatives                      = true                                   
{% endhighlight %}

## Closing Remarks
---
In this article, we saw how the JVM may replace some critical Java methods with more efficient implementations at runtime. Of course, the JVM compiler is a complex piece of software. 

Therefore, covering all the details related to intrinsics is both beyond the scope of this article and certainly beyond the writer's knowledge. However, I hope this serves as a good starting point for the curious!

As always, the source code is available on [GitHub](https://github.com/alimate/intrinsics)!

---
## *footnotes*
*1. Over the years, the file responsible for declaring the VM intrinsics has changed. For instance, before the `vmIntrinsics.hpp`, the `vmSymbols.hpp` was the home for all intrinsics.*

*2. The cover image is from [lls-ceilap](http://www.lls-ceilap.com/vi-jornadas---english.html) on Quantum Observer Effect.*