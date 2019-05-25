---
title: "Network stack - Part 2: Transport layer"
author: "Bruno Meneguele"
draft: true
---

Ok, continuing our journey through the network stack downwards we just reached
the _transport layer_, where some of the _magics_ behind the scenes happen.
This layer is where two of the most well known protocols are, being them TCP
(_Transmission Control Protocol_) and UDP (_User Datagram Protocol_).

These two protocols are used to the same purpose: **provide a way to two
different applications (i.e. a HTTP client and a server) to communicate between
them accross the whole internet (or even in its local system)**.

With that said, the single question arises: _how the hell these guys work?_.
And that is exactly what I am going to try to explain and show you.

## TCP and UDP internals

These two protocols are intended to work for the same main purpose, as stated
earlier, but they defer tremendously when comes to "controlling" the connection
itself, being that one - TCP - tries to keep the connection as reliable as
possible and transparent to the application, while the other - UDP - just give
up from everything on behalf of simplicity and performance, leaving the burden
of controlling mechanisms to the user's application if desired.

The way both protocols keep track of application's address is through **port
numbers**, which are controlled by the operating system and have a finite
amount available (16 bits = 65535). These ports are basically how house/street
numbers work: if you wish to talk to someone in a specific house, you need to
reach him in that specific number, but, at the same time, the application must
be waiting (listening) for someone in that specific port to allow the
communication to happen.

But wait, before we start to digress about that in a really high level, lets
take a look in how these information are organized, one protocol at a time, to
highlight the differences between them.

### TCP - Overview

The first thing to know is that each step downwards the stack a few bytes are
added to our whole packet size. That is due to header fields that are being
prefixed/suffixed each time a protocol manipulates it. For instance, TCP
protocol might add to our packet's length up to 60 bytes of information.

Below is an image representing all header fields available:

<center>![TCP Headers](/imgs/tcp-headers.png)</center>

As the protocol's name already states, TCP is basically a protocol to control
the delivery of packets through the network to end user applications. What most
shines herein is the fact that TCP guarantees certain principles to the application
layer, these being:

* Reliable data transfer, through:
    * _sequence number_ (SEQ) header field;
    * _acknowledgement number_ (ACK) header field;
    * Flow control mechanism:
        * Using _window_ field.
* Bandwidth congestion control mechanism, through:
    * ACK field and some internal timers.

As you can see, all the headers are basically used to control and handle a
transparent communication channel among applications within the internet.

But at the same time, **why is these guarantees even necessary?** Well,
basically because the internet itself is not a reliable system: it has
uncountable failures down the road with bogus systems, malfunctioning
equipments, broken or totally filled/busy links, and so forth. It is pretty
hard or even impossible to reach a destination application, geographically
distance, without hitting a single issue and that is the reasoning behind all
these control flags, fields and mechanisms.

It is also valid to remember that although the network stack still have more
protocols underneath transport layer, TCP is the one caring the responsibility
to deliver a transparent and reliable network to the application.

Now, lets talk about these two guarantees.

### TCP - Reliable Data Transfer

As I said before, there are three variables being monitored to keep the data
transfer between applications reliable: packet's sequence number,
acknowledgement number and a mechanism of flow control, which basically checks
for the window size field together with the other two variables.

#### Sequence number

The idea is not to get in too much details on that, thus what I can say about
sequence numbers is that it basically represents the order in which such packet
was sent, being the number the last byte of that packet counted since the
beginning of the whole data being sent.

For example, consider a data of a thousand bytes, which was decided fragmented
in ten smaller packets of a hundred bytes each:

`$$1000{bytes} = 10\times 100{bytes}$$`

## Recap from application layer packet

We are going to take the results we got in [part 1](/posts/net-stack-app) of
this series and literally encapsulate it within the, what is called, *payload*
from transport layer protocol's *segments* (TCP) or *datagram* (UDP).

Using HTTP client side of our application and remembering that HTTP protocols
uses TCP protocol as its transport layer, the request content that will be held
by TCP's payload is:

```
$ curl -v http://127.0.0.1:8080
...
> GET / HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.61.1
> Accept: */*
>
...
```

And to manually check this payload content to the most low level perspective we
can use the _tcpdump_ tool to monitor the network interface (**lo** in this
case, since we are working with _localhost_) for all incoming and outgoing
packets in a transport layer point of view, while sending requests to the
server:

```
# tcpdump -i lo -X -n -v 'port 8080'
...
15:04:42.901353 IP (tos 0x0, ttl 64, id 54427, offset 0, flags [DF], proto TCP (6), length 130)
    127.0.0.1.60016 > 127.0.0.1.webcache: Flags [P.], cksum 0xfe76 (incorrect -> 0xaa4b),
        seq 1:79, ack 1, win 512, options [nop,nop,TS val 2295614788 ecr 2295614788], length 78: HTTP, length: 78
	GET / HTTP/1.1
	Host: 127.0.0.1:8080
	User-Agent: curl/7.61.1
	Accept: */*
	
	0x0000:  4500 0082 d49b 4000 4006 67d8 7f00 0001  E.....@.@.g.....
	0x0010:  7f00 0001 ea70 1f90 8fe5 7395 d2ee c21d  .....p....s.....
	0x0020:  8018 0200 fe76 0000 0101 080a 88d4 4d44  .....v........MD
	0x0030:  88d4 4d44 4745 5420 2f20 4854 5450 2f31  ..MDGET./.HTTP/1
	0x0040:  2e31 0d0a 486f 7374 3a20 3132 372e 302e  .1..Host:.127.0.
	0x0050:  302e 313a 3830 3830 0d0a 5573 6572 2d41  0.1:8080..User-A
	0x0060:  6765 6e74 3a20 6375 726c 2f37 2e36 312e  gent:.curl/7.61.
	0x0070:  310d 0a41 6363 6570 743a 202a 2f2a 0d0a  1..Accept:.*/*..
	0x0080:  0d0a                               
...
```

> NOTE: just as a matter of clarification, you can see the string
> "127.0.0.1.webcache" representing the packet destination address (server IP
> address and TCP port): _webcache_ is a default port name set in
> */etc/services* file as alias to the port number **8080** that is being
> listened by our HTTP server.
>
>```
>$ grep webcache /etc/services
>webcache        8080/tcp        http-alt        # WWW caching service
>webcache        8080/udp        http-alt        # WWW caching service
>```

Well, you might be asking me what are all these values in the _tcpdump_ output,
what are their meanings, why are they used, and so on. Indeed, I just
presented a proof that the request in being placed within TCP segment. Now lets
breath for a second and start depicting some of the most important aspects of
transport layer protocols.

