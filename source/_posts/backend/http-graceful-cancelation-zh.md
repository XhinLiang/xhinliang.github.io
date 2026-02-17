title: HTTP 请求的优雅取消（Graceful Cancellation）
date: 2024-04-24 01:00:00
tags: [Chinese,XHR,HTTP,HttpClient,Nginx,HTTP/2,Cancelation,Cancel]
categories: Backend
toc: true
---

我们经常会写一些“很慢”的 HTTP 接口：比如触发导出、跑一段复杂计算、调用外部服务、或者生成大文件。

问题在于：**客户端可能在任务完成前就取消了请求**（关闭页面、点了取消、网络切换、超时等）。

那服务端能不能“及时知道”客户端已经取消？如果能知道，就可以尽早停止后续的昂贵操作，省 CPU/IO/下游资源。

这篇文章用一个简单的 Go + Gin 示例，把常见场景拆开说清楚：

- 不经过 Nginx：HTTP/1.1、HTTP/2
- 客户端“主动取消” vs “突然断网离线”
- 经过 Nginx 反向代理时：Nginx→Server 用 HTTP/1.1 或 HTTP/2

核心结论先说：

1. **Go 服务端可以通过 `Request.Context()` 感知“连接生命周期”的结束**，从而在 handler 里停止工作。
2. **HTTP/2/HTTP/3 支持显式取消（RST_STREAM）**，通常能更“及时地”通知服务端。
3. **突然离线（没有 FIN/RST/RST_STREAM）时，服务端往往感知不到**，除非你自己做了超时、心跳、或应用层中断机制。

## 不经过 Nginx

### HTTP/1.1：用 request context 做中断

下面用 Gin 写一个慢接口 `/http1`：每秒做一次“昂贵操作”，并在每一轮检查 `ctx.Done()`。

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()

    router.GET("/http1", func(c *gin.Context) {
        ctx := c.Request.Context()

        for i := 0; i < 10; i++ {
            if err := costlyOperation(ctx, i); err != nil {
                // 499 并不是标准 HTTP code，但在 Nginx 世界里常用来表示 client closed request
                c.AbortWithStatusJSON(499, gin.H{"error": "client disconnected"})
                return
            }
        }

        c.JSON(200, gin.H{"status": "ok"})
    })

    router.Run(":8080")
}

func costlyOperation(ctx context.Context, i int) error {
    select {
    case <-time.After(1 * time.Second):
        fmt.Println("Working...", i)
        return nil
    case <-ctx.Done():
        fmt.Println("Client disconnected. Stop.", i)
        return fmt.Errorf("client disconnected")
    }
}
```

#### 用 cURL 测试（HTTP/1.1）

```sh
curl http://localhost:8080/http1
^C  # 过几秒手动中断
```

你会看到服务端在某个循环里打印到 `ctx.Done()` 的分支，说明 **服务端感知到连接已经关闭**。

#### 用浏览器 XHR 测试

```js
var xhr = new XMLHttpRequest();
xhr.open('GET', 'http://localhost:8080/http1');
xhr.send();

setTimeout(() => xhr.abort(), 7000);
```

一般也能触发服务端的取消感知（取决于浏览器/网络栈，通常会导致连接关闭或复用连接上的中断）。

#### 这背后发生了什么？

在 Go 里，每个请求的 `Context` 会绑定到请求/连接的生命周期。

- 当底层 TCP 连接被关闭（FIN/RST），Go 的 net/http 会取消这个 context
- handler 里只要在耗时操作之间不断检查 `ctx.Done()`，就能“尽快”停下来

注意：Go 不会神奇地“打断”你正在执行的 CPU 密集型函数，它只会把 `ctx.Done()` 变成可读。**是否及时停下来，取决于你是否在关键路径里检查并传播 context。**

### HTTP/2：更明确的取消信号

Gin 在 TLS 下可以跑 HTTP/2（本地实验可以用自签证书）：

```go
// 为了启用 HTTP/2，把 Run 换成 RunTLS
router.RunTLS(":8081", "/tmp/server.crt", "/tmp/server.key")
```

#### 用 cURL（HTTP/2）测试

```sh
curl https://localhost:8081/http2 --insecure --verbose
^C  # 中断
```

HTTP/2 支持对单个 stream 发送 RST_STREAM 来取消请求，所以服务端通常能更早收到“取消”的明确语义。

### 突然离线（Offline）：最容易误判的场景

如果用户不是“点取消/关闭连接”，而是直接断网（例如手机关 Wi‑Fi、飞行模式），可能不会马上产生 TCP FIN/RST，也可能不会立刻发 HTTP/2 RST_STREAM。

结果就是：**服务端还以为请求正常进行**，你会看到循环完整跑完，最后写响应时才可能发现对端不可达（甚至写响应时也可能没立刻报错，取决于内核缓冲和重传）。

所以：

- “能否感知取消” ≠ “用户不想等了”
- 仅靠连接状态，无法覆盖所有“用户已离线/不再关心”的情况

### 小结（不经过 Nginx）

- HTTP/1.1：更多是依赖 TCP 连接状态（FIN/RST），通常在你下一次读写或检查 context 时才知道
- HTTP/2/HTTP/3：协议层有显式取消（RST_STREAM），更及时、更明确
- Offline：可能没有任何显式信号，服务端往往无法立刻知道

## 经过 Nginx（反向代理）

现实里，大多数服务端前面还有 Nginx：

- Client → Nginx：常见是 HTTP/2
- Nginx → Server：可能是 HTTP/1.1，也可能是 HTTP/2

这会影响“取消信号”是否能传递到你的应用。

下面是一个示意配置（仅用于说明）：

```conf
events { worker_connections 1024; }

http {
  error_log /tmp/https_error.log debug;

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
      proxy_ssl_verify off;
    }
  }
}
```

### 场景 1：Client(HTTP/2) → Nginx → Server(HTTP/1.1)

- 客户端取消：对 Nginx 来说是 HTTP/2 RST_STREAM
- 但 Nginx 到后端是 HTTP/1.1，**没有“RST_STREAM”这种语义**
- Nginx 只能选择：
  - 关闭到后端的 TCP 连接（让后端通过连接断开间接感知）
  - 或者继续把后端请求跑完但丢弃响应（浪费后端资源）

是否会“帮你断后端连接”，取决于 Nginx 行为和配置，以及后端写回时机。

### 场景 2：Client(HTTP/2) → Nginx → Server(HTTP/2)

- 客户端取消：Nginx 收到 RST_STREAM
- Nginx 到后端也是 HTTP/2：**可以把取消语义转发**（再发一个 RST_STREAM）
- 后端能更快、更明确地停止工作

### 小结（经过 Nginx）

如果你的业务非常在意“取消要尽快释放资源”，优先保证：

- 后端代码层面：长任务支持 `context` 贯穿（DB/HTTP/RPC 调用都能被中断/超时）
- 链路层面：尽量让 Nginx→Server 也跑 HTTP/2（或至少让连接断开能被后端及时感知）

## 实战建议（避免“以为能取消，但其实没停”）

1. **所有慢操作都要接收并传播 `context.Context`**：DB 查询、HTTP 调用、队列 publish、文件上传等。
2. **为长任务设置上限**：服务端超时（`context.WithTimeout`）比“等客户端取消”可靠。
3. **把“请求取消”当成优化而不是正确性依赖**：离线、网络抖动、代理行为都可能让你感知不到。
4. **对昂贵任务用异步化**：把任务丢到队列，HTTP 只返回 task id；客户端取消不影响任务一致性，由你控制是否可取消。

---

如果你想我补一段更贴近生产的示例（例如：Gin handler 里同时调用 DB + 下游 HTTP，并在中断时做清理/指标上报/日志关联 trace id），我也可以基于这个文章继续扩展。