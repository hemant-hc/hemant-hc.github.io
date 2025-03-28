---
layout: post
title: "Creating HTTP Server from scratch with NodeJS Part II: Handling TCP Streams"
---

In Part I, we created our first, very basic HTTP server using Node.js's `net` module. We learned how to accept raw TCP connections.

```js
server.on("connection", (socket) => {
  socket.on("data", (data) => {
    try {
        const parser = new HTTPRequestParser(data);
        socket.write(parser.getResponse());
    } catch (e)
  });
});
```

This approach doesn’t work because of how TCP (Transmission Control Protocol) operates. Since HTTP relies on TCP, it inherits its behavior TCP delivers data as a steady, reliable stream of bytes, not as separate, predefined messages

In Part II, we will fix this issue by correctly handling the TCP data stream. We'll explore what TCP streaming means, and implement the necessary buffering logic in our Node.js `net` server.

<h3>Understanding TCP Streams</h3>

1.  **Stream:** When a computer sends data to another using TCP, it can be thought of as pouring bytes into its end of the pipe. The receiving computer collects bytes from its end. TCP guarantees that the bytes arrive in the same order they were sent, but it doesn't guarantee that they arrive in the same groupings or chunks.
2.  **Network Fragmentation:** The network itself might break the data into smaller packets or TCP segments for transmission and reassemble them. A single `write` operation on the sender's side could result in multiple `data` events on the receiver's side, or multiple `write` operations could potentially be bundled into a single `data` event (though less common for typical HTTP).
3.  **Node.js `net.Socket`:** The `socket` object in our `net` server directly represents this TCP stream. The `data` event fires whenever the operating system has received *some* data from the network for that connection. The `chunk` argument passed to the event handler is a `Buffer` containing whatever bytes have arrived since the last `data` event. It could be a tiny part of an HTTP request, the whole request, or even multiple small requests (though HTTP pipelining makes the last case complex and less common).
4.  **The `end` Event:** The TCP connection signals that the sender has finished sending data (e.g., the client closed its sending side) by triggering an `end` event on the receiving socket. This is our cue that we should have received all the data for the current logical message(s) sent over that connection.

<h3>Why We need TCP Streaming for Our HTTP Server</h3>

Because TCP provides just a stream of bytes, our simple HTTP server receiving data via `socket.on('data')` has no guarantee that a single `data` event contains a complete HTTP request. An HTTP request consists of: A request line (e.g., `GET / HTTP/1.1\r\n`), Headers (e.g., `Host: example.com\r\nUser-Agent: ...\r\n`), A blank line (`\r\n`), and An optional message body

A `data` event might contain only the request line, or the request line and some headers, or the headers and part of the body. Attempting to parse this incomplete data as a full HTTP request will result in failure.

Therefore, our server must buffer the incoming data chunks from the TCP stream until it has received the entire HTTP request. Only then can it reliably parse the request line, headers, and body.

HTTP-level rules are used to figure out when HTTP request ends within the TCP stream, primarily using the `Content-Length` header or the `Transfer-Encoding: chunked` mechanism. The server needs to parse the headers first and then determine the body's length. However, before parsing the headers, we need to buffer enough data from the TCP stream to ensure we have all the headers and the crucial blank line (`\r\n\r\n`) that marks their end.

In this part: we will buffer all data from the TCP stream until the connection signals `end`. This guarantees we have the complete data sent by the client in that session before we attempt any HTTP parsing.

<h3>Implementing TCP Stream Handling</h3>

Let's modify our `server.on('connection', ...)` handler to implement this buffering strategy. We'll use `Buffer.concat()` to efficiently append incoming chunks and the `socket.on('end')` event to trigger processing.

```js
const net = require('net');
const { Buffer } = require('buffer');

function generateHttpResponse(
  statusCode,
  statusText,
  contentType = 'text/plain',
  body = ''
) {
  const bodyBuffer = Buffer.from(body, 'utf-8');
  const headers = [
    `HTTP/1.1 ${statusCode} ${statusText}`,
    `Server: learning-server-p2`,
    `Date: ${new Date().toUTCString()}`,
    `Content-Type: ${contentType}`,
    `Content-Length: ${bodyBuffer.length}`,
    `Connection: close`,
  ];
  const headerString = headers.join('\r\n') + '\r\n\r\n';
  return Buffer.concat([Buffer.from(headerString, 'utf-8'), bodyBuffer]);
}

const server = net.createServer();

server.on('connection', socket => {
  let requestDataBuffer = Buffer.alloc(0);
  const remoteAddress = `${socket.remoteAddress}:${socket.remotePort}`;
  console.log(`Connection opened: ${remoteAddress}`);

  socket.on('data', chunk => {
    console.log(
      `Received TCP chunk (${chunk.length} bytes) from ${remoteAddress}`
    );
    requestDataBuffer = Buffer.concat([requestDataBuffer, chunk]);
  });

  socket.on('end', () => {
    console.log(
      `TCP stream ended by client ${remoteAddress}. Total bytes buffered: ${requestDataBuffer.length}`
    );

    if (requestDataBuffer.length === 0) {
      console.log(`Empty request received from ${remoteAddress}. Closing.`);
      socket.end();
      return;
    }

    try {
      const tempBody = `Server buffered ${
        requestDataBuffer.length
      } total bytes from the TCP stream. \n\nData (first 500 bytes):\n${requestDataBuffer.toString(
        'utf-8',
        0,
        500
      )}...`;
      socket.write(generateHttpResponse(200, 'OK', 'text/plain', tempBody));
    } catch (error) {
      console.error(
        `Error during placeholder processing for ${remoteAddress}:`,
        error
      );
      if (!socket.writableEnded) {
        socket.write(
          generateHttpResponse(
            500,
            'Internal Server Error',
            'text/plain',
            'Internal server error.'
          )
        );
      }
    } finally {
      if (!socket.writableEnded) {
        socket.end();
      }
    }
  });

  socket.on('error', err => {
    console.error(`Socket error on connection ${remoteAddress}:`, err.message);
    socket.destroy();
  });

  socket.on('close', hadError => {
    console.log(
      `Connection fully closed for ${remoteAddress}. Had error: ${hadError}`
    );
  });
});

const PORT = 8080;
server.listen(PORT, '0.0.0.0', () => {
  console.log(`Part 2 Server listening on http://localhost:${PORT}`);
});

server.on('error', err => {
  if (err.code === 'EADDRINUSE') {
    console.error(
      `Error: Port ${PORT} is already in use. Please close the other process or choose a different port.`
    );
  } else {
    console.error('Server startup error:', err);
  }
  process.exit(1);
});
```

This code correctly handles the underlying TCP stream. It collects all the bytes until the client indicates it's done sending on that connection (`end` event).

<h3>Testing the Stream Handling</h3>

```bash
http GET :8080/

http -f POST :8080/submit data="Some test data" moredata="even more"
```

<h3>Next Steps</h3>

In Part III we will focus on Refactoring our `HTTPRequestParser` and the processing logic within the `end` handler to accurately parse the buffered request data into its components (method, path, headers, body). 
