---
layout: post
title: "Java 9: High level HTTP and WebSocket API"
permalink: /blog/2016/9/10/java-9-http-websocket-client
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2016-9-10-java-9-http-websocket-client.md"
---
Although the flagship feature of Java 9 is *Modularity*, a large number of other enhancements are planned for this release. One of those is the new HTTP client API that supports HTTP/2 and WebSocket and, hopefully, will replace the legacy `HttpUrlConnection` API. <br>
This new API is part of the `java.httpclient` module. So, if you're going to use this module, you should declare that your module *requires* the `java.httpclient` module:
{% highlight java %}
module me.alidg {
    requires java.httpclient;
}
{% endhighlight %}

`HttpRequest` can be used to represent an HTTP request which can be sent to a server. In order to build a `HttpRequest`, you can use the `create` static factory method:
{% highlight java %}
HttpRequest request = HttpRequest
                           .create(new URI("http://alidg.me/"))
                           .GET();
{% endhighlight %}

Then you can send the `GET` request and block until a response come back:
{% highlight java %}
HttpResponse response = request.response();
System.out.println(response.statusCode());
System.out.println(response.body(HttpResponse.asString()));
{% endhighlight %}

Also `HttpRequest` provides an `responseAsync` method which returns an instance of `CompletableFuture<HttpResponse>`. Using this
method one can simply compose and combine multiple `HttpResponse`es, react when the `HttpResponse` is available or
even transform it, all in a very declarative way:
{% highlight java %}
/*
  Two web targets to consume asynchronously
 */
List<URI> targets = Arrays.asList(
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

`java.httpclient` module also contains a client for WebSocket. `WebScoket` interface is the heart of this new
addition which contains four other abstractions to build, represent close codes, listen for events and messages and finally, handling
partial messages.<br>
For starters, we could implement the `WebSocket.Listener` interface, which, as its name suggests, is a listener for events and messages on a `WebSocket`. For example, here after receiving each message, we're sending a request for one more message and then printing the current message on console:
{% highlight java %}
class EchoListener implements WebSocket.Listener {
    @Override
    public CompletionStage<?> onText(WebSocket webSocket,
                                     CharSequence message,
                                     WebSocket.MessagePart part) {
        webSocket.request(1);
        return CompletableFuture.completedFuture(message)
                                .thenAccept(System.out::println);
    }
}
{% endhighlight %}

Then using the `newBuilder` static factory method we can connect to the WebSocket API:
{% highlight java %}
WebSocket.newBuilder(new URI("ws://localhost:8080/trades"), new EchoListener())
         .buildAsync().join();
{% endhighlight %}
