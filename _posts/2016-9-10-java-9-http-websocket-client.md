---
layout: post
title: "Java 9: High level HTTP and WebSocket API"
permalink: /blog/2016/9/10/java-9-http-websocket-client
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2016-9-10-java-9-http-websocket-client.md"
excerpt: "Although the flagship feature of Java 9 is Modularity, a large number of other enhancements are planned for this release. One of those is the new HTTP client API ..."
---
# Prologue
---
Although the flagship feature of Java 9 is *Modularity*, a large number of other enhancements are planned for this release. One of those is the new HTTP client API which supports HTTP/2 and WebSocket and, hopefully, will replace the legacy `HttpUrlConnection` API, the low level and
painful API. If you're wondering why that API was such a pain, consider reading [this post on stackoverflow][1].<br>
This new API is part of the `java.httpclient` module. So, if you're going to use this module, you should declare that your module *requires* the `java.httpclient` module. To do so, add the following to your `module-info.java` file:<br>
{% highlight java %}
module me.alidg {
    // Other declarations
    requires java.httpclient;
}
{% endhighlight %}

# HTTP Client API
---

First off, we should prepare the HTTP request. `HttpRequest` can be used to represent an HTTP request which can be sent to a server. In order to build a `HttpRequest`, use the `create` static factory method:<br>
{% highlight java %}
HttpRequest request = HttpRequest
                           .create(new URI("http://alidg.me/"))
                           .GET();
{% endhighlight %}

Then send the `GET` request and block until a response comes back:<br>
{% highlight java %}
HttpResponse response = request.response();
System.out.println(response.statusCode());
System.out.println(response.body(HttpResponse.asString()));
{% endhighlight %}

What about `POST` requests? The following snippet would send a `POST` request to some API with a JSON request body, expecting a JSON response with a 500 milliseconds timeout:<br>
{% highlight java %}
HttpRequest.create(new URI("http://some-api"))
                .header("Accept", "application/json")
                .header("Content-Type", "application/json")
                .timeout(TimeUnit.MILLISECONDS, 500)
                .body(HttpRequest.fromString(someJsonString))
                .POST();
{% endhighlight %}

There is a `PUT()` method for, well, `PUT` requests and `method(String methodName)` for others. Also `HttpRequest` provides an `responseAsync` method which returns an instance of `CompletableFuture<HttpResponse>`. Using this
method one can simply compose and combine multiple `HttpResponse`es, react when the `HttpResponse` is available or
even transform it to whatever she likes, all in a very declarative way:<br>
{% highlight java %}
/*
  Two web targets to consume asynchronously. List.of is part of Java 9, too.
 */
List<URI> targets = List.of(
        new URI("http://alidg.me"),
        new URI("http://stackoverflow.com")
);

/*
  Send two requests asynchronously, read the body in async fashion
  and print the body when it's available
 */
CompletableFuture<?>[] futures = targets.stream()
        .map(uri -> HttpRequest.create(uri).GET().responseAsync())
        .map(f -> f.thenCompose(r -> r.bodyAsync(HttpResponse.asString())))
        .map(f -> f.thenAccept(System.out::println))
        .toArray(CompletableFuture<?>[]::new);

/*
  Wait until all are done
 */
CompletableFuture.allOf(futures).join();
{% endhighlight %}

# WebSocket Client API
---

`java.httpclient` module also contains a client for WebSocket. `WebScoket` interface is the heart of this new
addition which contains four other abstractions to build, represent close codes, listen for events and messages and finally, handling
partial messages.<br>
For starters, we could implement the `WebSocket.Listener` interface, which, as its name suggests, is a listener for events and messages on a `WebSocket`. For example, here after receiving each message, we're sending a request for one more message and then printing the current message on console:<br>
{% highlight java %}
class EchoListener implements WebSocket.Listener {
    @Override
    public CompletionStage<?> onText(WebSocket webSocket,
                                     CharSequence message,
                                     WebSocket.MessagePart part) {
        // Gimme one more
        webSocket.request(1);

        // Print the message when it's available
        return CompletableFuture.completedFuture(message)
                                .thenAccept(System.out::println);
    }
}
{% endhighlight %}

Then using the `newBuilder` static factory method we can connect to the WebSocket API and continuously consume incoming messags: <br>
{% highlight java %}
WebSocket.newBuilder(new URI("ws://spring-sockets.herokuapp.com/stocks"),
                     new EchoListener()).buildAsync().join();
{% endhighlight %}

# Epilogue
---
The upcoming release of Java will be shipped with a new HTTP Client which is going to cover numerous problems of the existing `HttpURLConnection` API. For more details on its motivation and goals, checkout the original proposal [here][2]. The early access
javadocs contains more technical information about the API itself, which is available at [here][3]. And last but not least, you can
experiment with the API using the [Java 9 early access builds][4], have fun!  

[1]: http://stackoverflow.com/a/2793153/1393484
[2]: http://openjdk.java.net/jeps/110
[3]: http://download.java.net/java/jdk9/docs/api/index.html
[4]: https://jdk9.java.net/download/
