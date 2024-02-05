---
title: "It Is (Almost) Always Possible"
date: 2024-02-03
---
From time-to-time when I'm trying to find an answer about how to
perform some task, change some behaviour, etc., I'll run into some variant of "it can't be done".
Having been a software developer for about 25 years now I find this annoying. 
In my experience it is usually the case that just about anything can be done,
it's just a matter of how much work and whether the effort is worth the cost
**to the person who would perform the work**.

Let's look at a couple examples.

# Handle http and https requests to Spring Boot embedded web server... on the same port

First of all, what problem am I trying to solve?

I'm trying to build a web app using Spring Boot. 
When I start the web app and enter `http://localhost:8080` into the address bar, I see this result:

<div style="text-align:center">
    <img alt="Tomcat returns error in response to http request at port 8080" src="/assets/it_is_almost_always_possible/tomcat-http.png"/>
</div>

What I expect to see is the default Thymeleaf landing page:

<div style="text-align:center">
    <img alt="Tomcat returns expected response to https request at port 8080" src="/assets/it_is_almost_always_possible/tomcat-https.png"/>
</div>

This is a terrible user experience (interestingly Jetty is worse, by default). I know I could redirect to a different port but 
that doesn't solve anything--I can still manually enter `http://localhost:8443` and get an error.

To confirm that this problem can be solved, let's use Wireshark to
look at how a browser makes a request to a web server.

Here is the conversation between the browser and the web server when I enter `http://info.cern.ch` into the address bar.

<div style="text-align:center">
    <img alt="Packet exchange between browser and info.cern.ch for http request" src="/assets/it_is_almost_always_possible/info.cern.ch-http.png"/>
</div>

And here is the conversation between the browser and the web server when I enter `https://info.cern.ch` into the address bar.

<div style="text-align:center">
    <img alt="Packet exchange between browser and info.cern.ch for https request" src="/assets/it_is_almost_always_possible/info.cern.ch-https.png"/>
</div>

Now these conversations are obviously different, but the first few messages are the same: they establish a TCP connection to a server.
Once the TCP connection is established, the browser sends a different message (`GET / HTTP/1.1` for http vs `Client Hello` for https).
Since it is the browser that controls whether to initiate a TLS handshake, the server must be able to respond to either case.

From the Tomcat screenshots we can see that Tomcat is capable of handling both cases, 
but instead of correctly handling the TLS handshake, Tomcat returns an error. 
I mean, I guess that one could argue that this response is "correctly" handling the case,
but still not very user-friendly.

Getting back to the original question, see [this answer on ServerFault](https://serverfault.com/a/359465/169143).
Now, this answer "This is not going to be possible with Apache..." is technically correct. 
But even without looking at the Tomcat source it is clear that 
the reason for this behaviour is not because of any technical shortcoming. 
The longer answer really is:

> Tomcat and is obviously capable of handling http and https requests, but the current architecture doesn't support this behaviour on the same port. Since Tomcat is open-source I suppose it would be possible to rewrite how Tomcat establishes connections, but it would probably be a lot of work.

And in fact it turns out that Jetty has [done just this](https://stackoverflow.com/a/24891007).

# Calculating a partial derivative of a function in a third-party library

(I'm an engineer, not a mathematician, please forgive my poor explanation and notation)

This problem has come up at least a couple times when numerically calculating 
a partial derivative of a function in some library. 

A numeric derivative is calculated by calculating the value of a function at 
some point(s) around the point at which you want the derivative.
It doesn't matter if you use forward, backward, central, or whatever difference to calculate the derivative
but let's assume central difference:

$$ f'(x) = \frac{f(x + h/2) - f(x - h/2)}{h} $$

If the function $f(x)$ contains any `if` statements or other logic that depend on some state 
that is not passed into the function as an argument, then the function is probably
piecewise and (hopefully) continuous. To put it another way, the function is defined differently
in regions that are controlled by this external state. So:

$$ f(x) = \begin{cases} u(x), & \text{if A=0} \\ v(x), & \text{if A=1} \end{cases} $$ 

This type of function is **very common** when calculating pressure drop of multiphase flow--pressure gradient
in differs dramatically for different flow regimes.

If $x$ happens to be close to the boundary of one of these regions, close enough say that the step
used to calculate the numeric derivative is in a different region, then the calculated derivative 
will be garbage. Instead of calculating, for example:

$$ f'(x) = \frac{u(x + h/2) - u(x - h/2)}{h} $$

what you actually calculate could be:

$$ f'(x) = \frac{u(x + h/2) - v(x - h/2)}{h} $$

which is wrong and will likely cause all kinds of hard-to-debug problems with Newton convergence.

The only way to solve this problem is to control the external state that defines the different regions of the function.
But in my experience, this external state is usually itself depends on some other state and, in any case, 
is probably not exposed in the public API of the library.

**And here is where you will find the answer "it can't be done".**

If you have the source code for the library though, it is usually really very easy to add a couple functions 
to the library to a) get the current state; and b) set a new state. Getting back to the example of calculating
pressure drop of multiphase flow, to correctly calculate the derivative you need to get the flow regime 
calculated at point $x$, then explicitly set this flow regime when calculating $f(x+h/2)$ and $f(x-h/2)$.
With any luck you'll be able to calculate the numeric derivative correctly.

# Wrap-up

These examples, and my personal experience, have shown that it is usually possible to work around
any problem _if you have the source code_. It's just a matter of determination.

Estimating cost and value on the other hand are a matter of experience.