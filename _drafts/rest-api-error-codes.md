---
layout: post
title: "RESTful API Design: How to handle errors?"
permalink: /blog/2016/8/16/rest-api-error-handing
---

# Do care about your clients
----
"*We all do*". The well known first reaction of all software developers when they first come to this sentence. But,
of course, not when we're going to handle errors.<br>
We, as software developers, like to only consider happy paths in our scenarios and consequently, tend to forget the fact
that *Error Happens*, even more than those ordinary happy cases. Honestly, I can't even remember when was the last time I got an ok
response from a payment API, those are *fail by default*.<br> How did you handle errors while designing and implementing
your very last REST API? How did you handle validation errors? How did you deal with uncaught exceptions in your services? These are
some of questions you should consider answering before designing a new REST API.
<br>Of course, The most well known and unfortunate approach is *let the damn exception propagates* until our beloved client sees the
beautiful stacktrace on her browser! If she can't handle our `NullPointerException`, she shouldn't call herself a developer.

# Forget your platform: It's an *Integration Style*
----
In their amazing book, [Enterprise Integration Patterns][1], Gregor Hohpe and Bobby Woolf described one of the most important aspects of applications as the following:

> Interesting applications rarely live in isolation. Whether your sales application must interface
> with your inventory application, your procurement application must connect to an auction site,
> or your PDA’s PIM must synchronize with the corporate calendar server, it seems like any
> application can be made better by integrating it with other applications.

Then they introduced four *Integration Styles*. Regardless of the fact that the book was mainly about *Messaging Patterns*, we're going to focus on *Remote Procedure Invocation*. Remote Procedure Invocation is an umbrella term for all the approaches that expose some of application
functionalities through a *Remote Interface*, like REST and RPC. <br>
Now that we've established that REST is an Integration Style:

> **Do Not** expose your platform specific conventions into the REST API

The whole point of using an Integration Style, like REST, is to let different applications developed in different platforms work together,
hopefully in a more seamless and less painful fashion. The last thing a python client developer wanna see is a bunch of *Pascal Case URLs*, just because the API was developed with C#. Don't shove all those *Beans* into your message bodies just because you're using Java.<br>
For a moment, forget about the platform and strive to provide an API aligned with mainstream approaches in REST API design, like the
conventions provided by [Zalando team][2].

# Use HTTP status codes until you bleed!
----
Let's start with a simple rule:

{% highlight http %}
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Wed, 31 Aug 2016 20:51:25 GMT

{"status": "failed"}
{% endhighlight %}
> **Never ever** convey error information in 2xx and 3xx response bodies!

Many so-called RESTful services out there are using HTTP status codes incorrectly. One fundamentally wrong approach is to convey
error information through a `200 OK` response.

# Not enough information?
----

# Document everything!
----

# "Talk is cheap, show me the code"
----


# Further Reading
----

[1]: https://www.amazon.com/Enterprise-Integration-Patterns-Designing-Deploying/dp/0321200683
[2]: https://zalando.github.io/restful-api-guidelines/TOC.html
