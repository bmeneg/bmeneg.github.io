---
title: "Network stack - Part 1: Application layer"
author: "Bruno Meneguele"
draft: true
---

One of the things I have been studying in the past few days is how the
networking stack works in our lives nowadays:

* What are the standard protocols?
* How these protocols work?
* How everything are integrated?

These questions guided me towards a "playing around" exercise: build a simple
HTTP server and follow all the steps a simple data packet is handled through
the networking stack in some different situations following a top-down
approach.

Some tooling necessary to either allow us to send a packet through the stack
and also to follow its steps downwards. Each tool are going to be presented as
required.

# Application layer

The application layer is basically where the user start requesting things in
order to get a result or a reply from something/one. Any action performed by
the user in a web browser or in a video streaming application ends up in a
packet being sent to a server or to another user somewhere in the internet.

That is where we are going to start our journey: right in the top of the
networking stack.

## Our HTTP server

I'm going to use the GO language to build our simplest HTTP server possible. I
won't get in details around the code nor how the server is implemented
underneath, although I might present some information about the protocol
itself.

The GO language already have a HTTP server implementation available to us,
making our lives far easier, considering we are not interested in how it is
implemented for now.

The following code will generate the executable named _http-server_:

```
package main

import (
    "fmt"
    "net/http"
)

func main() {
    fmt.Println("Starting HTTP server...")
    // request handler
    http.HandleFunc("/", func (wtr http.ResponseWriter, req *http.Request) {
        fmt.Fprintf(wtr, "You have requested: %s\n", req.URL.Path)
    })
    // Listen and serve localhost:80 for connections
    fmt.Println("Listening...")
    http.ListenAndServe(":8080", nil)
} 
```

With that code running you are already ready to receive requests:
    
```
$ go install
$ http-server
Starting HTTP server...
Listening...
```

And in another shell:

```
$ curl http://localhost:8080
You have requested: /
```

## Overview on HTTP protocol

Being able to send and receive requests is the first step we need to achieve to
start analysing the stack. Now we need to get our hands somewhat dirty within
the HTTP protocol itself to gather some useful information for the rest of our
journey.

HTTP workflow runs basically around two basic concepts: **requests** and
**responses** for those requests. Although it might seems pretty simple, the
protocol is a pretty extensible one, holding a lot of headers that may get
included within each request and/or response. It is out of the scope of this
article to get into details about HTTP protocol and, because of that, we are
going to present only an overview of these concepts.

### Requests

An example of HTTP request is:

![Got from MDN web docs](/imgs/http-request.png)

Which is composed of four main parts:

* **method**: a verb that defines the operation the user wish to perform to the
  server. Although usually it is limited to _GET_ and _POST_ methods to,
  respectively, fetch and set operations, it is also possible to have other
  ones for different purposes, i.e. _OPTIONS_;
* **path**: the path to which the user wants to fetch information:
* **version**: HTTP protocol version;
* **headers**: additional information to the server. It might inform different
  ways the server could handle the request;
* **body**: in case the method chosen is _POST_ a body may appear under the
  headers.

Using our brand new server we can check the request with:

    $ curl -v http://127.0.0.1:8080
    ...
    > GET / HTTP/1.1
    > Host: 127.0.0.1:8080
    > User-Agent: curl/7.61.1
    > Accept: */*
    >
    ...

Some other informations are printed out with this command, but the request is
only these four lines.

### Responses

Now, a response example is:

![Got from MDN web docs](/imgs/http-response.png)

Which is composed by the following fields:

* **version**: HTTP protocol version;
* **status**: an integer code that represents the status of the request;
* **message**: a message that shortly describes the status code;
* **headers**: and more headers that presents additional information that may be
  used by the user for any reason;
* **body**: the response content _per se_.

The same command used to check request headers we can use to check the
response's ones:

    $ curl -v http://127.0.0.1:8080
    ...
    < HTTP/1.1 200 OK
    < Date: Wed, 22 May 2019 04:06:21 GMT
    < Content-Length: 22
    < Content-Type: text/plain; charset=utf-8
    <
    You have requested: /
    ...

## Stack analysis

For this exercise in particular we are interested in a couple of informations
that are present in HTTP packet: being them the _Date_, that we are planning to
use as a timestamp as checking mechanism later on, and _Content-Length_ which
will help us to understand some lower level protocols behaviors in certain
situations.

You can keep listening your requests and responses in real-time with a tool
called _tcpflow_ which keep track of all TCP connections being performed
through one of your network interfaces:

    # tcpflow -c -i <interface>

While in the another shell (or even through a normal browser) you keep
sending requests to your server.

    # tcpflow -c -i lo
    ...
    ::1.40138-::1.08080: GET / HTTP/1.1
    Host: 127.0.0.1:8080
    User-Agent: curl/7.61.1
    Accept: */*


    ::1.08080-::1.40138: HTTP/1.1 200 OK
    Date: Wed, 22 May 2019 04:32:35 GMT
    Content-Length: 22
    Content-Type: text/plain; charset=utf-8

    You have requested: /

Keep in mind that HTTP is an *application protocol* and has no clue about IP
addresses. I mean, although it has the _Host_ header it is used to pass the
information to an underlying protocol known as Internet Protocol (IP), which
will actively work with it.

When the time is come new requests are going to happen and possible greater
messages as responses bodies as well, but we guarantee that nothing will change
in our server, no magic will be added, only a few characters to the response
content.

Just to finish this first part of the series I think it is worth to mention
that: so far I have used only IP addresses to refer to my _localhost_,
127.0.0.1, and it happened because I was avoiding to mix information of two
application layer protocols, HTTP and DNS.

In short, DNS is the _Domain Name System_ which is a mechanism to translate
server's names to numerical addresses (IP), since the machines only know about
the numbers, while names are interesting only to humans.

The idea here is not to discuss how neither DNS and HTTP works internally, but
how the network stack is bundled together. Because of that I'm stopping here
and leaving a link to the [next series part](/posts/net-stack-transport).
