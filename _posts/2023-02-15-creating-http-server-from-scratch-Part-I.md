---
layout: post
title: "Creating HTTP Server from scratch with NodeJS Part I"
---

In this post, we will be bulding a simple HTTP server from scratch using NodeJS and in doing so we will review the basic structure of HTTP requests and response and get a basic knowlege of how HTTP server works.

An HTTP (Hypertext Transfer Protocol) server is a type of server that responds to client requests for web content by sending data over the internet. When a client requests a web page from a server, the server responds with the requested information using HTTP, which is the protocol that governs how data is exchanged on the internet.

<h3>HTTP module in NodeJS</h3>

NodeJS provides a `HTTP` module that can be used to create a simple HTTP server. This server allows us to listen on an arbitrary port and provide a callback function that will be invoked on every incoming request.

The callback recieves two arguments: a request object and a response object. The request object provides properties for the request. The response object sends a response to the server.

The "hello world" of Node's `http` server:

{% highlight javascript %}
const http = require("http");
const server = http.createServer();
server.on("request", (req, res) => {
  res.setHeader("Content-Type", "text/plain");
  res.end("Hello World!");
});
server.listen(3000, () => {
  console.log("Server running on port 3000");
});

{% endhighlight %}

This starts a web server on port 3000. When a `GET` request comes in, this server returns "Hello, World!" at the port we are listening at.

<h3>Dissecting HTTP request</h3>

A sample HTTP request looks like this:

```
POST /somepath HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 21
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Host: localhost:8080
User-Agent: HTTPie/2.6.0

name=John&address=Doe
```

We can observe that:

- Each line is delimited by \r\n.

- The first line in the HTTP request is the request line. It compose of three parts:

  - The request method, `POST` in this case.
  - The request URI `/somepath`
  - The protocol version `HTTP/1.1`

  <br />

- Each following line is called request header file. Every header line is optional in the HTTP request. A request header is composed of a field seperated by a `:`.

- A line with only `\r\n` indicates that the end of the request headers. Request body comes after this.

- In the request headers the `Content-type` and the `Content-Length` of the request body is mentioned.

In the next section we will be creating our own HTTP server using Node's built-in `net` module, which allows us to create a TCP server. We can serve HTTP requests from this TCP server.

<h3>Receiving and parsing HTTP request</h3>

We will create a streaming TCP server using `net` module.

When we make a HTTP request reading from the socket will give us the text of the request. We will be listening for `data` event in the stream and then console log that data.

To make the HTTP request we will be using HTTPie.

{% highlight js %}
const net = require("net");
const server = net.createServer();
server.on("connection", handleConnection);
server.listen(3000);

function handleConnection(socket) {
  socket.on("data", (chunk) => {
    console.log("Received chunk:\n", chunk.toString());
  });
  socket.write("HTTP/1.1 200 OK\r\nServer: my-web-server\r\n\r\n");
}
{% endhighlight %}

To get started, create a new file called "server.js" and place the code in it. Once the file is saved, run it using the command "node server.js" from your terminal.

Next, open a new terminal window and use the http command to send a basic `GET` request to your server.

```
http -v GET :3000
```

We shouldd see a verbose output from HTTPie, that shows us the request and response headers. In the Node terminal we should se something like this:

```
Received chunk:
GET / HTTP/1.1
Host: localhost:3000
User-Agent: HTTPie/2.6.0
Accept-Encoding: gzip, deflate
Accept: */*
```

Let's make `POST` request now:

```
http -v -f POST :3000 name="Joe"
```

This produces an output like this in Node terminal:

```
Received chunk:
POST / HTTP/1.1
Host: localhost:3000
User-Agent: HTTPie/2.6.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Content-Length: 8

name=Joe
```

Now let's parse the HTTP request, so that we can access the headers and manipulate the request body according to that.

First, we create a HTTPRequestParser class with a constructor that will receive the http response data. Then we convert this raw HTTP response data to string for further processing.

We then dissect the HTTP request data to request line, headers and the request body.

{% highlight js%}
const net = require("net");
const server = net.createServer();

class HTTPRequestParser {
  constructor(rawHTTPResponse) {
    this.rawHTTPResponse = rawHTTPResponse;
    this.httpResponseString = rawHTTPResponse.toString();

    this.requestType = null;
    this.requestPath = null;
    this.httpVersion = null;
    this.headers = {};
    this.body = null;
    this.parsedBody = {};

    this.processHTTPResponseString();
    this.processBody();
  }

  processHTTPResponseString = () => {
    const splitResponse = this.httpResponseString.split("\r\n");

    const [requestType, requestPath, httpVersion] = splitResponse[0].split(" ");
    this.requestType = requestType;
    this.requestPath = requestPath;
    this.httpVersion = httpVersion;

    const headersSplit = splitResponse
      .slice(1, splitResponse.length - 2)
      .map((kv) => ({
        key: kv.split(":")[0],
        value: kv.split(":")[1].trimLeft(),
      }));
    headersSplit.forEach((kv) => {
      this.headers[kv.key] = kv.value;
    });

    this.body = splitResponse[splitResponse.length - 1];
  };

  getResponseBodyForPath = () => {
    switch (this.requestPath) {
      case "/details": {
        return `Hello ${this.parsedBody.name}, you live at address: ${this.parsedBody.address}`;
      }
      case "/": {
        return "Hello, world!";
      }
      default: {
        throw new Error("404: Path not found");
      }
    }
  };

  getResponse = () => {
    const body = this.getResponseBodyForPath();
    return `HTTP/1.1 200 OK\r\nServer: learning-server\r\nContent-Length: ${body.length}\r\n\r\n${body}`;
  };

  processBody = () => {
    const [mimeType, ...others] = this.headers["Content-Type"].split(";");

    switch (mimeType) {
      case "application/x-www-form-urlencoded": {
        this.body.split("&").forEach((element) => {
          const splitElement = element
            .split("=")
            .map((element) => element.replace(/\+/g, "%20"))
            .map(decodeURI);
          this.parsedBody[splitElement[0]] = splitElement[1];
        });
        break;
      }
      default: {
        throw new Error("500: Not Implemented");
      }
    }
  };
}

server.on("connection", (socket) => {
  socket.on("data", (data) => {
    const parser = new HTTPRequestParser(data);
    socket.write(parser.getResponse());
  });
});

server.listen(8080);
{% endhighlight %}

We can split the request line, headers and request body using the `\r\n` delimiter.

The headers are defined as key value pair. We then process the body according to the data in the headers field.
