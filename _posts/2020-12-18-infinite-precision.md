---
layout: post
title: "Infinite Precision"
permalink: /blog/2020/12/18/infinite-precision
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-12-18-infinite-precision.md"
excerpt: "The art, science, and pitfalls of measurement!"
image: https://alidg.me/images/starry_night.jpg
toc: true
subtitle: "For my part I know nothing with any certainty, but the sight of the stars makes me dream!"
---
I genuinely enjoy listening to [Neil Degrasse Tyson](https://twitter.com/neiltyson?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor). Regardless of the subject, sometimes, I find myself losing the track of time while listening, watching, or reading his works!

Probably, being a *space geek* is one of the most obvious reasons for this!
<p style="text-align:center">
  <img src="/images/in-a-hurry.jpeg" alt="NDT's Book Cover"><br>
  <em><sup>The cover of Astrophysics for People in a Hurry</sup></em>
</p>
But *the passion*, *the excitement*, and the way he articulates complex subjects in a very approachable manner are equally as effective.

I remember once, in particular, he was talking about measurements. Basically, he was saying that there is no such thing as..! Well, What's the fun in me telling the story? Let's see what he actually said, instead of just mentioning the gist of it here!


## Measurement Uncertainty
---
"Oh, of my favorite subjects to think about is measurement, something we so take for granted every day. **There are measurements all the time that we're exposed to**, that we're handed, that we do ourselves."

<p style="text-align:center">
  <img src="/images/measure-height.webp" alt="Measuring height at home"><br>
  <em><sup>Having someone help you to measure your height may improve accuracy :D (https://bit.ly/3gYdtWo)</sup></em>
</p>

"What's a typical one? Oh, how tall are you? Oh, I'm 5' 8", you say. Well, I say, well, how do you know that? Well, I was just at the doctor's. I just had a checkup last week. 

OK, how did they measure your height? Well, I took off my shoes. And I stood on this little thing. And there's a little rod that comes up and bends down over the top of my head. OK, fine. And they made the reading. There it was. 5' 8". I might ask, was it 5' 8" and 1/2? No, it was right on 5' 8". 

OK, you're content with that answer. But it was a measurement. **And every measurement ever made, and ever will be made, has uncertainties built into it.** You may see that term in books called measurement error. But it's not an error in the sense that you made a mistake. That's why I stay away from the word in that context. It's a measurement uncertainty."

To conclude, "all measurement in life comes down to the approximation that you're comfortable with". In fact, "**This is true for anything you will measure. There is no precise answer. There's only the answer that you'll be happy with.**"


## Measuring Device
---
I, personally, like to think of this concept this way: **Every measurement you make, about anything, is as accurate as your measuring device.** For instance, this one should ring a bell for all of us:
{% highlight java %}
var startedAt = System.currentTimeMillis();
// do something
var duration = System.currentTimeMillis() - startedAt;
{% endhighlight %}
Here we've created a very simple measuring device, to measure the execution time of the surrounded block of code.

Even in this simple example, our measurement device can be wrong or off for a lot of reasons. Did we account for NTP resets here? How about GC pauses? What if our process get preempted while executing the code? What if compiler threads compete for too much resources? CPU migrations? False sharing? CPU caches? 

These questions, among many others, are the reasons why our *measurement* might be off. Most of the time, we are so concerned about *what we're measuring*. This is a very valid concern, to be fair. However, **knowing our measuring device is equally as important as what we're measuring.**

Enough with the concepts, let's see a more concrete example.


## To Measure
---
First, we need to implement something that we can measure its performance. To that end, let's implement a simple gRPC service the echos back whatever it receives. Here is the Protobuf description:
{% highlight protobuf %}
service EchoService {
  rpc echo (Message) returns (Message) {}
}

message Message {
  string content = 1;
}
{% endhighlight %}
And the corresponding service implementation:
{% highlight java %}
public final class EchoService extends EchoServiceImplBase {

    @Override
    public void echo(Message request, StreamObserver<Message> responseObserver) {
        responseObserver.onNext(request);
        responseObserver.onCompleted();
    }
}
{% endhighlight %}
To bootstrap the gRPC server, we can do something like:
{% highlight java %}
var server = ServerBuilder
    .forPort(9000)
    .addService(new EchoService())
    .build();

server.start();

System.out.println("I'm listening to port 9000");
server.awaitTermination();
{% endhighlight %}
Here, we're registering our simple `EchoService` as one of the service implementations. After that, we're making sure that the server listens to port `9000` for the incoming requests.

Now that we have a working server, let's measure its throughput.


## Measure Twice
---
For starters, let's send 10 thousand requests to our gRPC server. First, the connection:
{% highlight java %}
var requests = 10_000;
var executorService = Executors.newWorkStealingPool();
var channel = NettyChannelBuilder
    .forAddress("localhost", 9000)
    .usePlaintext()
    .executor(executorService)
    .build();
var client = EchoServiceGrpc.newFutureStub(channel);
{% endhighlight %}
Here we're using the *work stealing* pool to handle the client requests. The second step is to initialize a `CountDownLatch` to the number of requests:
{% highlight java %}
var latch = new CountDownLatch(requests);
{% endhighlight %}
This way we can wait for all requests to finish.
<p style="text-align:center">
  <img src="/images/archers.jpg" alt="Elf archers are ready to go"><br>
  <em><sup>Open fire? (https://bit.ly/3p4k7gi)</sup></em>
</p>
Now let's send some requests:
{% highlight java %}
var request = Message.newBuilder().setContent("Hello").build();
var startedAt = System.nanoTime();
for (int i = 0; i < requests; i++) {
    client.echo(request).addListener(latch::countDown, executorService);
}
latch.await();
{% endhighlight %}
Here we're sending 10000 requests. Each time we receive a response from the server, we *decrement* the latch by one. After sending all requests, we use the `latch.await()` to wait for all responses to arrive.

Finally, let's print our measurements:
{% highlight java %}
var duration = System.nanoTime() - startedAt;
System.out.println("Took: " + duration / 1_000_000.0 + " millis");
System.out.println("RPS: " + requests * 1_000_000_000.0 / duration);
{% endhighlight %}

### Benchmark Result
---
The first time we run the benchmark, the result would be something like:
{% highlight plain %}
Took: 2989.744137 millis
RPS: 3344.767826866383
{% endhighlight %}
Probably, the server didn't warm up properly, yet. Re-running the same benchmark a few more times and the result would be more stable:
{% highlight plain %}
Took: 1543.706665 millis
RPS: 6477.914636716296
{% endhighlight %}
It's better but I do believe we can do much better!

### Golang
---
Let's implement the same benchmark using Golang. First off, we should generate the Proto stubs via the `protoc` compiler:
{% highlight go %}
//go:generate protoc -I src/main/proto --go_out=plugins=grpc:. echo_service.proto
package main

import (
	"context"
	"flag"
	"fmt"
	echo "github.com/alimate/measurement/g/grpc"
	"google.golang.org/grpc"
	"log"
	"sync"
	"time"
)
{% endhighlight %}
Next step, we should connect to the server:
{% highlight go %}
var requests int
flag.IntVar(&requests, "requests", 10_000, "The number of requests")
flag.Parse()

conn, err := grpc.Dial("localhost:9000", grpc.WithInsecure())
if err != nil {
	log.Fatal("Failed to connect to the server", err)
}
{% endhighlight %}
Then we should prepare the `WaitGroup` (latch implementation in Golang):
{% highlight go %}
wg := &sync.WaitGroup{}
wg.Add(requests)
msg := &echo.Message{
	Content: "Hello",
}
{% endhighlight %}
<p style="text-align:center">
  <img src="/images/fire.jpg" alt="Elf archers are ready to go"><br>
  <em><sup>Elf archers are ready to go!</sup></em>
</p>
And fire:
{% highlight go %}
startedAt := time.Now()
for i := 0; i < requests; i++ {
	go func() {
		_, err := echo.NewEchoServiceClient(conn).Echo(context.Background(), msg)
		if err != nil {
			fmt.Println("Failed to handle a request", err)
		}
		wg.Done()
	}()
}

wg.Wait()
duration := time.Since(startedAt).Milliseconds()
fmt.Println("Took: ", duration)
fmt.Println("RPS: ", float64(requests)*1000.0/float64(duration))
{% endhighlight %}

Here we're launching 10000 goroutines to send the benchmark requests. Each time a goroutine performs its duty, we decrement the `WaitGroup`. Different syntax but the logic is the same as the Java code. Now let's see the result:
{% highlight plain %}
Took:  291
RPS:  34364.26116838488
{% endhighlight %}
So the same gRPC server just handled 34K RPS! Something doesn't quite add up there.

## Cold Starts
---
So what can be the culprit for such a huge difference in the throughput of the same server? Maybe Golang is just much more powerful than JVM:
<p style="text-align:center">
  <img src="/images/powerful-go.jpg" alt="Powerful Go!"><br>
  <em><sup>Goroutines and Channels for the win!</sup></em>
</p>
Golang might found a better way to wait for IO! But let's compare these measuring devices for a moment to find a better explanation:
 - Java is a compiled language, which compiles to an intermediate language. JVM is responsible for executing the compiled IL or bytecode. The HotSpot implementation is using a technique called *Profile Guided Optimization (PGO)* to optimize the code while executing them.
 - Golang is a compiled language, as well. However, the code will be compiled to a native binary that is directly executable on the machine. So all the optimizations are done at compile-time. Consequently, there is no PGO there.

Two valid technical choices with dramatic effects on the runtime. The more the HotSpot JVM executes a piece of code, the more the optimizations, and possibly a better performance. On the other hand, Golang artifacts already packaged with the native instructions. Therefore, there isn't much happening at runtime, in terms of compiler optimizations.

A few moments ago, we did send a few thousands of requests to the server, just to warm up the Java server (*what we were measuring!*). On the contrary, we didn't give any chance to the Java client to warm up properly! 
<p style="text-align:center">
  <img src="/images/warmup.png" alt="Warm up!"><br>
  <em><sup>Once I warm up, bring it on!</sup></em>
</p>
To test this hypothesis, let's perform a few rounds of warm up:
{% highlight java %}
for (int j = 0; j < 10; j++) {
    var latch = new CountDownLatch(requests);
    var request = Message.newBuilder().setContent("Hello").build();
    var startedAt = System.nanoTime();
    for (int i = 0; i < requests; i++) {
        client.echo(request).addListener(latch::countDown, executorService);
    }

    latch.await();
    var duration = System.nanoTime() - startedAt;
    System.out.println("Took: " + duration / 1_000_000.0 + " millis");
    System.out.println("RPS: " + requests * 1_000_000_000.0 / duration);
    System.out.println("------------------");

    Thread.sleep(1000);
}
{% endhighlight %}
Here we're running the same benchmark 10 times in the same JVM instance. And the result:
{% highlight plain %}
Took: 1830.082399 millis
RPS: 5464.234837439142
------------------
Took: 440.567319 millis
RPS: 22698.006794280627
------------------
Took: 390.688221 millis
RPS: 25595.85741900317
------------------
Took: 318.143291 millis
RPS: 31432.377431463738
------------------
Took: 256.045263 millis
RPS: 39055.594635234476
------------------
Took: 313.399405 millis
RPS: 31908.165237263293
------------------
Took: 377.708606 millis
RPS: 26475.43593433505
------------------
Took: 251.95215 millis
RPS: 39690.0760719843
------------------
Took: 255.048815 millis
RPS: 39208.18059868265
------------------
Took: 253.419225 millis
RPS: 39460.305349761846
------------------
{% endhighlight %}
Look at the sudden jump in RPS between the first and second iterations, 20K RPS! And at the end, the warmed up client managed to record 39K requests per second. Paying some attention to the *measuring device* paid off!

## Closing Remarks
---
To reiterate, every measurement we make, about anything, is as accurate as our measuring device. Therefore, knowing our measuring device is as important as knowing what we're measuring. 

This knowledge helps us to understand the *measurement uncertainties* much better. For instance, we saw how does the runtime characteristics affect our measurement result. 

Does that mean we did consider everything in our example? Absolutely not! That was only the compiler effect. We didn't consider the GC effect on both Golang and Java. Golang does not need that much of a warmup but the underlying machine does! We did run both the server and benchmarks on the same machine. Therefore competing resources (CPU, RAM, FDs, etc.) on the client and server-side might affect the result. This list goes on and on!

The bottom line, **there is no such thing as absolute and infinite precision. There's only the precision that we'll be happy with!** 

Also, If you're interested in watching the NDT talk that inspired this article, [go for it](https://www.youtube.com/watch?v=8IuBLd3qoUo)! Moreover, the source code is available on [GitHub](https://github.com/alimate/measure-twice)! 