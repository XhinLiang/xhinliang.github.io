title: HTTP Graceful Cancelation
date: 2024-04-24
tags: [XHR,HTTP,HttpClient,Nginx,HTTP/2,Cancelation,Cancel]
categories: Backend
toc: true
---

## Outline

Imagine we are writing an HTTP server as described below, where the endpoint takes a long time to complete.

When a client starts a request, it may cancel it before the long-running task is completed.

The question we're addressing here is: can the server be aware that the client has cancelled the request? If the server can be aware, it can stop the costly subsequent operation in advance to save server resources.

## Without Nginx

### HTTP/1.1

Let's start with a simple example using Go's Gin framework to serve the endpoint `/http1`.

```go
package main

import (
    "fmt"
    "time"
    "context"
    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()

    router.GET("/http1", func(c *gin.Context) {
        // Context from the Gin handler
        ctx := c.Request.Context()

        // Simulate a long-running task with periodic checks
        for i := 0; i < 10; i++ {
            // Costly operation
            if err := costlyOperation(ctx, i); err != nil {
                c.AbortWithStatusJSON(499, gin.H{"error": "Client has disconnected"})
                return
            }
        }

        // Complete the task
        c.JSON(200, gin.H{"status": "Task completed successfully"})
    })

    router.Run(":8080") // listen and serve on 0.0.0.0:8080
}

func costlyOperation(ctx context.Context, i int) error {
    select {
    case <-time.After(1 * time.Second): // simulate work by sleeping
        fmt.Println("Working...", i)
    case <-ctx.Done(): // check if the context is done
        fmt.Println("Client has disconnected. Stopping task.", i)
        return fmt.Errorf("client has disconnected")
    }
    return nil
}
```

#### cURL

We use local cURL via HTTP/1.1 to test it:

```shell
curl http://localhost:8080/http1
^C # after 2 seconds
```

The server log is:

```shell
[GIN-debug] Listening and serving HTTP on :8080
Working... 0
Working... 1
Working... 2
Client has disconnected. Stopping task.  3
[GIN] 2024/04/23 - 23:14:17 | 499 |  3.332915625s |       127.0.0.1 | GET      "/http1"
```

This means the server is aware that the client has cancelled the request.

#### XHR

We use XHR in a browser to test it:
```javascript
var xhr = new XMLHttpRequest();
xhr.open("GET", "http://localhost:8080/http1");
xhr.onreadystatechange = function() {
    if (xhr.readyState === XMLHttpRequest.DONE) {
        if (xhr.status === 200) {
            console.log("Response:", xhr.responseText);
        } else {
            console.error("Request failed with status:", xhr.status);
        }
    }
};
xhr.send();
xhr.abort(); // after 7 seconds
```

The server log is:
```
Working... 0
Working... 1
Working... 2
Working... 3
Working... 4
Working... 5
Working... 6
Working... 7
Client has disconnected. Stopping task.  8
```

#### How it works?

The context included with each HTTP request in Go is linked to the lifecycle of the request. If the underlying TCP connection is closed, Go’s HTTP server automatically cancels this context. The cancellation can occur due to client disconnection (TCP FIN or RST), server-side timeout, or if the server manually cancels the context for other reasons (like application logic deciding to abort the request processing).

### HTTP/2

Since Gin supports HTTP/2, let's use it to conduct our tests.

```go
package main

import (
    "fmt"
    "time"
    "context"
    "github.com/gin-gonic/gin"
)

func main() {
    // ... no changes in the router setup

    // To enable HTTP/2, replace the `Run` method with `RunTLS`
    router.RunTLS(":8081", "/tmp/server.crt", "/tmp/server.key") // listen and serve on 0.0.0.0:8081 using TLS, necessary for HTTP/2
}

// ... no changes in the handler
```

#### cURL

We use local cURL via HTTP/2 to test it:

```shell
curl https://localhost:8081/http2 --insecure --verbose
*   Trying 127.0.0.1:8081...
* Connected to

 localhost (127.0.0.1) port 8081 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* ALPN, server accepted h2 as the protocol
* Server certificate:
*  subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd
*  start date: Apr 23 15:35:12 2024 GMT
*  expire date: Apr 23 15:35:12 2025 GMT
*  issuer: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Using Stream ID: 1 (easy handle 0x14e80bc00)
> GET /http2 HTTP/2
> Host: localhost:8081
> user-agent: curl/7.86.0
> accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
```

The server log is:

```shell
Working... 0
Working... 1
Working... 2
Working... 3
Working... 4
Client has disconnected. Stopping task.  5
```

This indicates that the server is aware that the client has cancelled the request.

#### XHR

```javascript
var xhr = new XMLHttpRequest();
xhr.open("GET", "https://localhost:8081/http2");
xhr.onreadystatechange = function() {
    if (xhr.readyState === XMLHttpRequest.DONE) {
        if (xhr.status === 200) {
            console.log("Response:", xhr.responseText);
        } else {
            console.error("Request failed with status:", xhr.status);
        }
    }
};
xhr.send();
xhr.abort(); // after 6 seconds
```

The server log is:

```shell
Working... 0
Working... 1
Working... 2
Working... 3
Working... 4
Working... 5
Working... 6
Client has disconnected. Stopping task.  7
```

### Offline

When the user suddenly makes the device offline (without any network notification e.g., TCP FIN&RST or HTTP/2 RST), what will happen?

We can use another device to test it.

```sh
# 0. setup the WiFi
# 1. open the browser and go to http://localhost:8080/http1
# 2. close the WiFi
```

The server log is:

```shell
Working... 0
Working... 1
Working... 2
Working... 3
Working... 4
Working... 5
Working... 6
Working... 7
Working... 8
Working... 9
[GIN] 2024/04/23 - 23:49:49 | 200 | 10.011190667s |   192.168.0.105 | GET      "/http1"
```

It means the server is not aware that the client has cancelled the request.

### Summary

In the context of HTTP/1.1, cancellation is not explicitly handled by the protocol itself. If a client sends a request and then closes the connection before receiving the response, the server might detect that the connection has been closed when it tries to write the response. However, by that time, the server might have already processed or even completed processing the request. This type of cancellation is detected through TCP/IP connection management rather than through HTTP protocol features.

For HTTP/2 and HTTP/3, the situation is different. Both protocols support explicit request cancellation. In HTTP/2 and HTTP/3, a client can send a RST_STREAM frame to the server, which effectively tells the server to stop processing a specific stream (which corresponds to a request). This allows for more efficient cancellation and resource management on the server, as it can immediately stop processing a request upon receiving a cancellation notice.

Here’s how each protocol handles cancellation:

- **HTTP/1.1**: Depends on whether the TCP connection is reused or not. 
	- If reused: the server may notice that the client

 has disconnected when it attempts to send a response and fails due to a broken TCP connection. The server might also implement timeouts or other mechanisms to detect that a client has stopped responding.
	- If not reused: means the TCP connection has been closed by the client. The server can detect that the client has disconnected and stop processing the request.
- **HTTP/2 and HTTP/3**: The client can send a RST_STREAM frame to explicitly cancel a request. This lets the server know immediately that the request should be aborted, which is more efficient and can help in managing server resources effectively.

Servers that support HTTP/2 or HTTP/3 are thus better equipped to handle request cancellations gracefully. In any case, how well cancellation is handled can also depend on how the server's application logic is implemented, such as whether it periodically checks for connection status or interruptions during lengthy operations.

## With Nginx

We usually use Nginx as a reverse proxy between the client and the server.

Since we commonly use HTTP/2 on the Nginx side, so after integrating with Nginx, the process flow will be like:

```shell
Client - HTTP/2 -> Nginx - HTTP/1.1 -> Server
```
or
```shell
Client - HTTP/2 -> Nginx - HTTP/2 -> Server
```

So we prepared the nginx configuration file in the `nginx.conf`

```conf
events {
    worker_connections 1024;
}
# HTTP block
http {
  error_log /tmp/https_error.log debug;  # Specify a custom path and log level

    # HTTPS server
    server {
        listen 443 ssl http2;
        server_name local_nginx_http_2;

        ssl_certificate /tmp/server.crt;
        ssl_certificate_key /tmp/server.key;

        location /http1 {
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $host;
            proxy_http_version 1.1;
        }

        location /http2 {
            proxy_pass https://127.0.0.1:8081;
            proxy_set_header Host $host;
            proxy_ssl_verify off;  # Add this line to disable SSL verification for the proxy
        }        
    }
}
```

In the above part, we have a conclusion that both cURL and XHR can perform a graceful cancellation.

So in this part, to simplify, we will just use a cURL to test these cases.

#### Nginx - HTTP/1.1 -> Server

The cURL is like:

```shell
curl https://localhost/http1 --insecure
^C # after 6 seconds
```

The server log is:

```shell
Working... 0
Working... 1
Working... 2
Working... 3
Working... 4
Working... 5
Client has disconnected. Stopping task.  6
```

It means the server is aware that the client has cancelled the request.

#### Nginx - HTTP/2 -> Server

The cURL is like:

```shell
curl https://localhost/http2 --insecure
^C # after 6 seconds
```

The server log is:

```shell
Working... 0
Working... 1
Working... 2
Working... 3
Working... 4
Working... 5
Client has disconnected. Stopping task.  6
```

It means the server is aware that the client has cancelled the request.

### Summary

When there is an Nginx reverse proxy sitting between the client and the server, handling client requests and forwarding them to the server, the behavior upon cancellation can vary depending on the protocols used between the client and Nginx, and between Nginx and the server. Let's consider the two scenarios:

#### Scenario 1: Client (HTTP/2) -> Nginx -> Server (HTTP/1.1)

1. Client Cancels Request: The client sends a request using HTTP/2 and cancels it by sending a RST_STREAM frame to Nginx.
2. Nginx Behavior: Upon receiving the RST_STREAM frame, Nginx recognizes that the client has cancelled the request. Given that the connection to the server is over HTTP/1.1, which doesn’t support RST_STREAM, Nginx has the option to either continue processing the request or terminate the TCP connection to the server. The action Nginx takes can depend on how it’s configured: it might be set to close the connection to free up resources more quickly or to simply drop the response if the server completes the request.
3. Server Side: If Nginx decides to close the TCP connection, the server might detect this as a connection error or failure when attempting to send a response. This abrupt closing can signal to the server that it should stop processing, albeit indirectly. Without a connection to send a response, the server may halt operations, reducing

 unnecessary resource usage.

#### Scenario 2: Client (HTTP/2) -> Nginx -> Server (HTTP/2)

1. Client Cancels Request: As before, the client sends a request using HTTP/2 and cancels it by sending a RST_STREAM frame.
2. Nginx Behavior: Since the connections both to the client and from Nginx to the server use HTTP/2, Nginx can forward the cancellation directly by sending another RST_STREAM frame to the server. This maintains protocol integrity and allows for seamless communication of the client's intention to cancel.
3. Server Side: The server, upon receiving the RST_STREAM frame from Nginx, knows immediately that the request has been cancelled and can stop processing right away. This protocol-supported method of cancellation is efficient and clear, minimizing wasted resources.