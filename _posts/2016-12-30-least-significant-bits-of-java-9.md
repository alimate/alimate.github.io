---
layout: post
title: "Least significant bits of Java&nbsp;9"
permalink: /blog/2016/12/30/least-significant-bits-of-java-9
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2016-12-30-least-significant-bits-of-java-9.md"
excerpt: "An overview of small and yet extremely useful new additions to Java 9..."
---

# Prologue
---
Although the flagship feature of Java 9 is Modularity, a large number of other enhancements are planned for this release. This article will provide an overview of those features that are scheduled for Java 9 release but are not as famous and glorious as the Jigsaw.   

# Reactive Streams
---
[*Reactive Streams*][rs] is a contract for asynchronous stream processing with non-blocking back pressure. `Publisher` and `Subscriber` are two key concepts in the Reactive Streams specification. `Publisher` is a producer of items (and related control messages) that will be received by `Subscriber`s, all these are done in a non-blocking way. Back pressure allows to control the amount of inflight data. That is, it will regulate the transfer between a slow publisher and a fast consumer and a fast publisher and a slow consumer.

Reactive Streams specification is comprised of four interfaces: `Publisher`, `Subscriber`, `Subscription` and `Processor`.
{% highlight java %}
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription subscription);
    void onNext(T item);
    void onError(Throwable throwable);
    void onComplete();
}

public interface Subscription {
    void request(long n);
    void cancel();
}

public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
{% endhighlight %}
`Publisher`  is a producer of items which `Subscriber`s can subscribe to.  `Subscriber` provides four callbacks: `onNext` will be called every time a new item is available. There are also two terminal signals, `onComplete` to signal the successful completion of the subscription and `onError` to signal an unrecoverable error encountered by a `Publisher` or `Subscription`. `Subscription` encapsulates the communication between `Publisher` and `Subscriber`: one can request zero or more elements from the publisher or even cancel the subscription altogether. `Processor` is a component that acts as both a `Subscriber` and `Publisher`.

Reactive Streams will be integrated in Java 9. In `java.util.concurrent` there is a `Flow` class that contain these four interfaces. Note that these are just the interfaces, Java 9 (or possibly any version of Java) won't be shipped with an implementation for this specification. So in order to actually use a reactive API, at some point you should use an implementation. Current notable implementations of this specification on the JVM are [Project Reactor][pr] (which will be integrated in Spring 5), [Akka Streams][as] and [RxJava][rx].

# More Concurrency Updates
---
Suppose we have a `CompletableFuture` that going to fetch some movie recommendations from our fictional recommendation service. Now we want to add the capability of loading some static recommendations if the service couldn't provide the expected result in a timely manner:
{% highlight java %}
Supplier<List<Movie>> callTheRecommendationService = // call the slow service
CompletableFuture
        .supplyAsync(callTheRecommendationService)
        .completeOnTimeout(Collections.singletonList(fightClub), 1, TimeUnit.SECONDS)
        .thenAccept(showTheRecommendationsToUser);
{% endhighlight %}
Here if the recommendation service could provide the recommendations in less than 1 second, we'll show those recommendations to the end user. Otherwise, we'll make the poor user to watch *Fight Club* one more time. The new `completeOnTimeout(T value, long timeout, TimeUnit unit)` method completes the `CompletableFuture` with the given value if it's not completed before the given timeout.

If you don't want to provide the static recommendation, you can simply raise a `TimeoutException` with the `orTimeout(long timeout, TimeUnit unit)` method:
{% highlight java %}
CompletableFuture
        .supplyAsync(callTheRecommendationService)
        .orTimeout(1, TimeUnit.SECONDS)
        .thenAccept(showTheRecommendationsToUser);
{% endhighlight %}

Java 9 also will be shipped with some other enhancements to the `CompletableFuture` API, which was introduced in Java 8. There was a `completedFuture` static factory method in Java 8 to create a new `CompletableFuture` that is already completed with the given value. If you want to create an already failed `CompletableFuture`, you can use the new `failedFuture` static factory method:
{% highlight java %}
CompletableFuture.failedFuture(new IllegalArgumentException());
{% endhighlight %}
Also there will be `completedStage` and `failedStage` methods which will create new `CompletionStage`s that is already completed or failed, respectively.

# Convenience Factory Methods for Collections
---
The goal of [JEP 269][jep-269] is to:
<blockquote>
Define library APIs to make it convenient to create instances of collections and maps with small numbers of elements, so as to ease the pain of not having collection literals in the Java programming language.
</blockquote>

For example, in order to create a `List` with few `String`s in it:
{% highlight java %}
List<String> geeks = List.of("Fowler", "Beck", "Evans");
{% endhighlight %}
Or to create a `Map` from singer name to his/her band name:
{% highlight java %}
Map<String, String> singerToBand = Map.of(
                "Amy Lee", "Evanescence",
                "Chris Martin", "Coldplay",
                "Jared Leto", "Thirty Seconds to Mars",
                "Matt Bellamy", "Muse"
        );
{% endhighlight %}
Same code can be refactored as:
{% highlight java %}
Map<String, String> singerToBand = Map.ofEntries(
                Map.entry("Amy Lee", "Evanescence"),
                Map.entry("Chris Martin", "Coldplay"),
                Map.entry("Jared Leto", "Thirty Seconds to Mars"),
                Map.entry("Matt Bellamy", "Muse")
        );
{% endhighlight %}

# The Java Shell
---
Java Shell or *JShell* is the *Read Eval Print Loop (REPL)* environment for Java. That is, an interactive tool to evaluate declarations, statements, and expressions of the Java programming language. To start experimenting with JShell, open up a terminal and just type `jshell` (assuming `jshell` is accessible from your `PATH`):
{% highlight bash %}
> cd $JAVA9_HOME/bin
> jshell
|  Welcome to JShell -- Version 9-ea
|  For an introduction type: /help intro


jshell>
{% endhighlight %}
JShell is an interactive Java shell that accepts your Java code snippets and evaluates them in real time and tells you the result. So there is no need to create a `public static void main` to test an API, anymore!

For starters, let's declare a variable:
{% highlight bash %}
jshell> String txt = "foo bar"
txt ==> "foo bar"
{% endhighlight %}
As you can see *semicolons* are optional. JShell supports <kbd>Tab</kbd> *completion*, just type a few chars and hit tab and JShell tries its best to complete that name:
{% highlight bash %}
jshell> txt.sub
subSequence(   substring(
{% endhighlight %}
When you did choose a method name, you can hit the <kbd>Shift</kbd> and <kbd>Tab</kbd> to see all overloaded versions of the chosen method:
{% highlight bash %}
jshell> txt.substring(
String String.substring(int beginIndex)
String String.substring(int beginIndex, int endIndex)
<press shift-tab again to see javadoc>
{% endhighlight %}
Hit it one more time to see the Javadoc:
{% highlight bash %}
jshell> txt.substring(
String String.substring(int beginIndex)
Returns a string that is a substring of this string.The substring begins with
the character at the specified index and extends to the end of this string.
Examples:
     "unhappy".substring(2) returns "happy"
     "Harbison".substring(3) returns "bison"
     "emptiness".substring(9) returns "" (an empty string)


Parameters:
beginIndex - the beginning index, inclusive.

Returns:
the specified substring.
-- Press space for next javadoc, Q to quit. --
{% endhighlight %}
If you evaluate an expression and don't assign it to a variable, it will be assigned to a *Pseudo Variable*, which starts with `$`:
{% highlight bash %}
jshell> txt.substring(0, 3)
$3 ==> "foo"
{% endhighlight %}
Here the `$3` pseudo variable will hold the `foo` value, you can make sure of that fact by:
{% highlight bash %}
jshell> $3
$3 ==> "foo"
{% endhighlight %}
Here's a `List` of three programming languages:
{% highlight bash %}
jshell> List<String> langs = List.of("Java", "JS", "Scala")
langs ==> [Java, JS, Scala]
{% endhighlight %}
Now we're gonna just keep those languages that their name starts with the letter `J`:
{% highlight bash %}
jshell> langs.stream().filter(n -> n.startsWith("J")).collect(Collectors.toList())

|  Error:
|  cannot find symbol
|    symbol:   variable Collectors
|  langs.stream().filter(n -> n.startsWith("J")).collect(Collectors.toList())
|                                                        ^--------^
{% endhighlight %}
Here JShell couldn't find the `Collectors` symbol. By default JShell provides a set of common imports:
{% highlight bash %}
jshell> /imports
|    import java.util.*
|    import java.io.*
|    import java.math.*
|    import java.net.*
|    import java.util.concurren
|    import java.util.prefs.*
|    import java.util.regex.*
{% endhighlight %}
Since `Collectors` isn't part of those default imports, we should bring it to the scope:
{% highlight bash %}
jshell> import java.util.stream.*
jshell> langs.stream().filter(n -> n.startsWith("J")).collect(Collectors.toList())
$11 ==> [Java, JS]
{% endhighlight %}
Anyway JShell is all about exploratory programming with features to ease interaction including: a history with editing, tab-completion, automatic addition of needed terminal semicolons, and configurable predefined imports and definitions.

# New IO Features
---
Probably the most familiar code snippet for every single Java developer is the [following][is-to-ba-so]:
{% highlight java %}
InputStream is = ...
ByteArrayOutputStream buffer = new ByteArrayOutputStream();
int nRead;
byte[] data = new byte[16384];
while ((nRead = is.read(data, 0, data.length)) != -1) {
  buffer.write(data, 0, nRead);
}
buffer.flush();

byte[] bytes = buffer.toByteArray();
{% endhighlight %}
In Java 9 there is a `readAllBytes` method for `InputStream` which reads all remaining bytes from the input stream and
returns a byte array containing the bytes read from that input stream. Finally [After 20 years][j9-rab-so] we can simply write:
{% highlight java %}
InputStream is = ...
byte[] bytes = is.readAllBytes();
{% endhighlight %}
There is also a `readNBytes(byte[] b, int off, int len)` method which reads the requested number of bytes from the input stream into the given
byte array.

Do you ever want an easy way to write contents of a Java `InputStream` to an `OutputStream`? The newly added `transferTo(OutputStream out)` method will read all bytes from the input stream and writes the bytes to the given output stream in the order that they are read. So in order to write contents of a Java `InputStream` to an `OutputStream`, one can:
{% highlight java %}
InputStream is = ...
OutputStream os = ...
is.transferTo(os);
{% endhighlight %}

# New HTTP Client
---
 The other new addition to Java 9 is the new HTTP client API which supports HTTP/2 and WebSocket. This client isn't in the `java.base` module, so we should declare a dependency to the `java.httpclient` module in our `module-info.java`:
 {% highlight java %}
 module me.alidg {
     // Other declarations
     requires java.httpclient;
 }
{% endhighlight %}
There is a `HttpRequest` to represent an HTTP request which can be sent to a server:
{% highlight java %}
HttpRequest request = HttpRequest
                           .create(new URI("http://alidg.me/"))
                           .GET();
{% endhighlight %}
By calling the `response()` method you can send the request and block until a response comes from the server encapsulated as a `HttpResponse`:
{% highlight java %}
HttpResponse response = request.response();
System.out.println(response.statusCode());
System.out.println(response.body(HttpResponse.asString()));
{% endhighlight %}
For more details on this new API, you can checkout the [Java 9: High level HTTP and WebSocket API][link-to-http-post] post.

# And More!
---
`Stream` and `Optional`, which first were introduced in Java 8, hasn't undergone any significant changes but there will be some new and useful methods there. `Stream` would have `dropWhile` and `takeWhile` methods to drop or take only elements that match the given `Predicate`. New `stream()` method in `Optional` can be used to convert the `Optional` to a `Stream` of at most one element. Also, `ifPresentOrElse(Consumer<? super T> action, Runnable emptyAction)` method is useful for scenarios when you want to consume the optional value if it's present or supply a empty action if it's not.

`LocalDate` will have a `datesUntil(LocalDate endExclusive)` method which returns a sequential ordered stream of dates, starting from this  date and goes to `endExclusive` (exclusive) by an incremental step of 1 day. For example, in order to count number of leap years between my birthday and current date:
{% highlight java %}
LocalDate birthday = LocalDate.of(1989, Month.FEBRUARY, 11);
long leapYears = birthday
                    .datesUntil(LocalDate.now())
                    .map(d -> Year.of(d.getYear()))
                    .distinct()
                    .filter(Year::isLeap)
                    .count();
{% endhighlight %}
For this particular scenario, using a 1 year step is more plausible (instead of default 1 day):
{% highlight java %}
long leapYears = birthday
                    .datesUntil(LocalDate.now(), Period.ofYears(1))
                    .map(d -> Year.of(d.getYear()))
                    .filter(Year::isLeap)
                    .count();
{% endhighlight %}

Sometimes in Java we had to write statements like:
{% highlight java %}
String media = request.getHeader("Accept");
media = media != null ? media : "Default Value";
{% endhighlight %}
languages like groovy are providing an *Elvis* operator which is a shortening of the ternary operator:
{% highlight groovy %}
def media = request.getHeader("Accept") ?: "Default Value";
{% endhighlight %}
Java doesn't provide such a syntactic sugar but `Objects` utility class will have `requireNonNullElse` and `requireNonNullElseGet` methods to kinda (not as smooth and cool as Elvis operator) alleviates that problem:
{% highlight java %}
import static java.util.Objects.requireNonNullElse;
// Omitted stuff
String media = requireNonNullElse(request.getHeader("Accept"), "Default Value");
{% endhighlight %}


[rs]: http://www.reactive-streams.org/
[is-to-ba-so]: http://stackoverflow.com/a/1264737/1393484
[j9-rab-so]: http://stackoverflow.com/a/37681322/1393484
[jep-269]: http://openjdk.java.net/jeps/269
[link-to-http-post]: https://alidg.me/blog/2016/9/10/java-9-http-websocket-client
[pr]: https://projectreactor.io/
[as]: http://doc.akka.io/docs/akka/2.4.16/scala/stream/index.html
[rx]: https://github.com/ReactiveX/RxJava
