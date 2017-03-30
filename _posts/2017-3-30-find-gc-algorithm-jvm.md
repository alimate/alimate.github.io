---
layout: post
title: "Which GC algorithm is being used for a specific JVM instance?"
permalink: /blog/2017/3/30/find-gc-algorithm-jvm
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2017-3-30-find-gc-algorithm-jvm.md"
excerpt: "Using the monitoring tools provided by the JDK, we can gain insights into the JVM itself. One of such insights that may come useful in some situations, is the GC algorithm..."
---
Using the monitoring tools provided by the JDK, we can gain insights into the JVM itself. One of such insights that may come useful in some situations, is the GC algorithm used by a specific JVM instance. First off, we should somehow identify that JVM instance. In order to do that, we can use the `jps` utility to find all the running JVMs in the current machine or a machine specified with a `host-id`. <br>
For example, running the following command in my local machine:
{% highlight bash %}
jps -l
{% endhighlight %}
Would produce the following result:
{% highlight bash %}
2912 org.jetbrains.jps.cmdline.Launcher
1985
2913 me.alidg.Application
2915 sun.tools.jps.Jps
2175 org.jetbrains.idea.maven.server.RemoteMavenServer
{% endhighlight %}
As you can spot from the output, the *Process Id* for my trivial Spring Boot app running with the `me.alidg.Application` main entry is `2913`. You find this process id using the typical `ps` (or whatever OS specific tool you have at your disposal) but it needs more filtering to find your desired JVM instance.<br>
After finding the process id of the JVM instance, you can use the `jmap` utility. Basically `jmap` will provides heap dumps and other information about JVM memory usage:
{% highlight bash %}
jmap [option] <pid>
{% endhighlight %}
So knowing that our pid is `2913`, we can use the following command to print java heap summary:
{% highlight bash %}
jmap -heap 2913
{% endhighlight %}
The output of this command would be:
{% highlight bash %}
Attaching to process ID 2913, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.121-b13

using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC
// Truncated
{% endhighlight %}
As you can spot from the output, this JVM instance is using the *Concurrent Mark-Sweep GC* or *CMS* GC algorithm. If I run my Spring Boot app with the `-XX:+UseG1GC` flag, the output for the same command would be:
{% highlight bash %}
Attaching to process ID 2961, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.121-b13

using thread-local object allocation.
Garbage-First (G1) GC with 8 thread(s)
// Truncated
{% endhighlight %}
`jmap -heap` also will provide information about the JIT compiler, Heap configuration, Interned strings statistics and Heap usage.
