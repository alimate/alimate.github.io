---
layout: post
title: "Java 9: Effectively final variables in try-with-resources"
permalink: /blog/2017/8/8/try-with-resources-effectively-final
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2017-8-8-effective-final-try-with-resources.md"
excerpt: "The final version of try-with-resources statement in Java SE 7 requires a fresh variable to be declared for each resource being managed by the statement..."
---
Try with resources, first introduced in Java 7, is a *Loan Pattern* implementation baked in the
language which enables us to borrow a [*AutoCloseable*][1] resource, mess with it as we wish and in the
meanwhile, not to worry about freeing the resource; because it's automatically being taken care of by
the compiler. Before this feature being rolled out, the common idiom to use a resource was kinda like the following:

{% highlight java %}
InetSocketAddress birthPort = new InetSocketAddress(1989);
DatagramChannel udpServer = DatagramChannel.open().bind(birthPort);
try {
    udpServer.receive(buffer);
    // Process the packet
} catch (Exception e) {
    // handle the exception
} finally {
    if (udpServer != null) {
        try {
            udpServer.close();
        } catch (Exception ignored) {}
    }
}
{% endhighlight %}
*Try with resources* reduces the preceding code to:
{% highlight java %}
InetSocketAddress birthPort = new InetSocketAddress(1989);
try(DatagramChannel udpServer = DatagramChannel.open().bind(birthPort)) {
    udpServer.receive(buffer);
    // Process the packet
} catch (Exception e) {
    // handle the exception
}
{% endhighlight %}
There is no need to explicitly free the borrowed resource, which is nice, but the resource declaration
is a bit ugly and it's even more uglier if we try to manage multiple resources:
{% highlight java %}
InetSocketAddress birthPort = new InetSocketAddress(1989);
try (DatagramChannel udpServer = DatagramChannel.open().bind(birthPort);
     Selector selector = Selector.open()) {
    // go non-blocking and register channel with selector
    udpServer.receive(buffer);
    // Process the packet
} catch (Exception e) {
    // handle the exception
}
{% endhighlight %}
The problem is, In Java 7, the resources to be managed by a try-with-resources must be fresh variables
declared like:
{% highlight java %}
try (Resource resource = ...)
{% endhighlight %}
In Java 9, the try-with-resources statement [refined][2] in a way that it can accept `final` and `effectively final`
variables. So you can write this in Java 9:
{% highlight java %}
InetSocketAddress birthPort = new InetSocketAddress(1989);
DatagramChannel udpServer = DatagramChannel.open().bind(birthPort);
Selector selector = Selector.open();

try (udpServer;selector) {
    // go non-blocking and register channel with selector
    udpServer.receive(buffer);
    // Process the packet
} catch (Exception e) {
    // handle the exception
}
{% endhighlight %}


[1]: https://docs.oracle.com/javase/8/docs/api/java/lang/AutoCloseable.html
[2]: http://openjdk.java.net/jeps/213
