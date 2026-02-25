title: HTTP Graceful Cancellation
date: 2024-04-24
tags: [XHR,HTTP,HttpClient,Nginx,HTTP/2,Cancellation,Cancel]
categories: Backend
toc: true
---

Suppose we’re writing an HTTP server with an endpoint that takes a long time to finish.

A client may start a request and then cancel it before the handler completes (closing the tab, navigating away, hitting “stop”, etc.).

The question is: can the server detect that the client has canceled the request, and do so early enough to stop expensive downstream work?

## Without Nginx

### HTTP/1.1

Let’s start with a simple example using Go’s Gin framework to serve the endpoint `/http1`.

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

Test with local curl over HTTP/1.1:

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

This shows the server notices the disconnect and cancels the request context.

#### XHR

Now test the same endpoint from a browser using XHR.

In real code you would call `xhr.abort()` from a timer or a user action; the snippet below just highlights the cancellation call site.

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

#### How does it work?

In Go, each request carries a `context.Context` that is tied to the request lifecycle. If the underlying connection is closed, Go’s HTTP server cancels that context, and frameworks like Gin surface it via `c.Request.Context()`.

This cancellation can be triggered by a client disconnect (TCP FIN/RST), a server-side timeout, or application logic choosing to abort the request processing.

### HTTP/2

Gin can also serve HTTP/2 when TLS is enabled, so we can repeat the same experiment over HTTP/2.

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

Test with local curl over HTTP/2:

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

Again, the server detects the cancellation while the handler is still running.

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

What if the client disappears without a clean close—for example, the device loses Wi‑Fi or switches to airplane mode? In that case the server might not receive a TCP FIN/RST (or an HTTP/2 reset) immediately.

We can use another device to test it:

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

In this run, the server keeps working and completes the handler; from the server’s perspective, nothing was canceled.

### Summary

At a high level, HTTP/1.1 doesn’t have an explicit, per-request “cancel” signal. What you usually get is “the connection went away”, and the server only learns that once the TCP stack (or the application runtime) notices the close.

HTTP/2 and HTTP/3 are different: they support stream-level cancellation. A client can reset a single stream (e.g., via a `RST_STREAM` frame), which makes it much easier for the server to stop work promptly.

In practice:

- **HTTP/1.1**: “Cancellation” is typically the client closing the TCP connection. The server may only notice on read/write, or when the runtime surfaces the disconnect (for Go, that means `Request.Context()` is canceled).
- **HTTP/2 and HTTP/3**: The client can explicitly cancel a request by resetting the stream, so the server can stop work sooner and more reliably.

No matter the protocol, cancellation is only useful if your handler checks for it during long-running work.

## With Nginx

We usually use Nginx as a reverse proxy between the client and the server.

Since HTTP/2 is commonly used on the client-to-Nginx side, after integrating Nginx the traffic flow typically looks like this:

```shell
Client - HTTP/2 -> Nginx - HTTP/1.1 -> Server
```
or
```shell
Client - HTTP/2 -> Nginx - HTTP/2 -> Server
```

So we prepare an Nginx configuration in `nginx.conf`:

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

Above (without a proxy), we saw that both curl and XHR can trigger graceful cancellation.

In this section, to keep things simple, we’ll only use curl to test cancellation through Nginx.

#### Nginx - HTTP/1.1 -> Server

The curl command is:

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

This shows the upstream server still observes the cancellation.

#### Nginx - HTTP/2 -> Server

The curl command is:

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

This shows the upstream server still observes the cancellation.

### Summary

When there is an Nginx reverse proxy sitting between the client and the server, handling client requests and forwarding them to the server, the behavior upon cancellation can vary depending on the protocols used between the client and Nginx, and between Nginx and the server. Let's consider the two scenarios:

#### Scenario 1: Client (HTTP/2) -> Nginx -> Server (HTTP/1.1)

1. Client Cancels Request: The client sends a request using HTTP/2 and cancels it by sending a RST_STREAM frame to Nginx.
2. Nginx Behavior: Nginx can observe the reset from the client. Toward the upstream (HTTP/1.1), it can’t forward a per-request reset frame, so it either keeps the upstream request running and drops the response later, or it closes the upstream connection to force the server to observe a disconnect. Which path it takes depends on configuration and internal behavior.
3. Server Side: If Nginx closes the upstream connection, the server sees a disconnect and can stop work (e.g., via request context cancellation). If Nginx keeps the upstream request running, the server may do unnecessary work even though the client is already gone.

#### Scenario 2: Client (HTTP/2) -> Nginx -> Server (HTTP/2)

1. Client Cancels Request: As before, the client sends a request using HTTP/2 and cancels it by sending a RST_STREAM frame.
2. Nginx Behavior: Because both legs use HTTP/2, Nginx can forward the cancellation to the upstream by resetting the corresponding stream.
3. Server Side: The server receives an explicit stream reset and can stop work immediately, which is both clearer and more efficient.
