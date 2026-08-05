# 08. 专题：服务器得到目标 URL 后，如何访问网站并返回响应

本文专门跟踪最常见的正向 HTTP 代理：

```bash
gost -L http://:8080
```

客户端使用 GOST 作为代理：

```bash
curl -v -x http://127.0.0.1:8080 http://example.com/news
curl -v -x http://127.0.0.1:8080 https://example.com/news
```

这两条命令走不同路径：

- HTTP URL：GOST 读取 HTTP 请求，再用 `http.Transport` 向网站发请求；
- HTTPS URL：客户端先发 `CONNECT`，GOST 建立 TCP 隧道，之后主要转发加密 TLS 字节。

本文对应：

- `gost` 当前工作区；
- `github.com/go-gost/x v0.12.5`；
- `github.com/go-gost/core v0.4.3`；
- `github.com/judwhite/go-svc v1.2.1`。

## 0. 先区分四个对象

以 HTTP 请求为例：

```text
curl/browser
    │
    │ clientConn
    ▼
GOST :8080
    │
    │ targetConn
    ▼
example.com:80
```

| 对象 | 含义 |
| --- | --- |
| `req` | GOST 从客户端读出的 `*http.Request` |
| `clientConn` | 客户端与 GOST 之间的 `net.Conn` |
| `targetConn` | GOST 与目标网站或代理链最后一跳之间的 `net.Conn` |
| `resp` | GOST 从目标网站读出的 `*http.Response` |

“返回响应”不是函数把 `resp` return 给浏览器。浏览器在另一个进程中，GOST 必须把响应序列化成字节并写入 `clientConn`。

## 1. 进程如何进入可接收请求的状态

### 1.1 `main()` 把生命周期交给 go-svc

当前仓库 `cmd/gost/main.go`：

```text
main
→ svc.Run(&program{})
```

### 1.2 go-svc 调用程序生命周期

```text
svc.Run
→ program.Init
→ program.Start
→ 等待 SIGINT/SIGTERM
→ program.Stop
```

### 1.3 `program.Start` 构建 service

当前仓库 `cmd/gost/program.go`：

```text
parser.Parse
→ loader.Load
→ registry.ServiceRegistry().GetAll()
→ 每个 service 启动 Serve goroutine
```

### 1.4 service 接受客户端连接

`x/service/service.go`：

```text
defaultService.Serve
→ listener.Accept
→ 创建 session context
→ 为该连接启动 goroutine
→ handler.Handle(ctx, clientConn)
```

`listener.Accept` 返回时，TCP 三次握手已经由操作系统完成。

## 2. HTTP Handler 如何取得 URL

文件：`x/handler/http/handler.go`

```go
br := bufio.NewReader(conn)
req, err := http.ReadRequest(br)
```

假设客户端发送：

```http
GET http://example.com/news?q=go HTTP/1.1
Host: example.com
User-Agent: curl/...
```

`http.ReadRequest` 解析出：

```text
req.Method        = "GET"
req.URL.Scheme    = "http"
req.URL.Host      = "example.com"
req.URL.Path      = "/news"
req.URL.RawQuery  = "q=go"
req.Host          = "example.com"
```

代理请求使用 absolute-form，即请求行中包含完整 URL。普通源站收到的请求通常使用 `/news?q=go` 这种 origin-form。

随后：

```text
Handle
→ handleRequest
→ normalizeRequest
```

`normalizeRequest` 取得网络和目标地址：

```text
network = "tcp"
addr    = "example.com:80"
```

注意这里用于“拨号”的是 `host:port`，完整 URL 中的 path/query 则仍保存在 `req.URL`，稍后由 HTTP Transport 发送。

## 3. 分流：HTTP 与 HTTPS 从这里分开

`handleRequest`：

```go
if req.Method != http.MethodConnect {
    return h.handleProxy(...)
}
return h.handleConnect(..., addr, ...)
```

```text
GET/POST/PUT/... → 普通 HTTP 代理
CONNECT          → TCP 隧道，通常承载 HTTPS
```

---

# 路径 A：普通 HTTP URL

## A1. `handleProxy` 处理 keep-alive

文件：`x/handler/http/proxy.go`

第一次请求：

```text
handleProxy
→ proxyRoundTrip(req)
```

如果连接没有关闭，它继续：

```text
http.ReadRequest(clientConn)
→ proxyRoundTrip(nextReq)
→ 重复
```

因此一个客户端 TCP 连接可以承载多个 HTTP 请求。

## A2. 清理代理专用 Header

`proxyRoundTrip` 删除：

```text
Proxy-Authorization
Proxy-Connection
Gost-Target
X-Gost-Target
```

这些 Header 用于客户端与代理服务器之间，不应该泄露给目标网站。

## A3. 真正发出 HTTP 请求的位置

核心代码：

```go
resp, err := h.transport.RoundTrip(req.WithContext(ctx))
```

`h.transport` 在 Handler 初始化时创建：

```go
&http.Transport{
    DialContext: h.dial,
    // ...
}
```

职责分工：

| 组件 | 负责内容 |
| --- | --- |
| `proxyRoundTrip` | GOST 的认证、bypass、记录以及把响应写回客户端 |
| `http.Transport` | HTTP 报文发送、连接池、keep-alive、读取并解析响应 |
| `h.dial` | 按 GOST 路由规则建立底层连接 |

`RoundTrip` 会根据 `req.URL` 确定请求目标。当连接池没有可复用连接时，它调用 `DialContext`，也就是 `h.dial(ctx, "tcp", "example.com:80")`。

## A4. `h.dial` 进入 GOST Router

文件：`x/handler/http/connect.go`

```go
conn, err = h.options.Router.Dial(ctx, network, addr)
```

Router 返回的 `net.Conn` 随后交给标准库 `http.Transport`。Transport 在该连接上完成：

```text
写入目标请求
→ 等待目标响应状态行和 Header
→ 按 Content-Length / chunked / 连接关闭读取 Body
→ 组装 *http.Response
```

## A5. Router 如何选择直连或代理链

文件：`x/chain/router.go`

```text
Router.Dial
→ Resolve 域名/hosts
→ Chain.Route 选择路线
→ 没有路线则 DefaultRoute
→ route.Dial
```

### A5.1 没有 `-F`：直接连接

`x/chain/route.go`：

```text
DefaultRoute.Dial
→ internal net Dialer.Dial
→ 操作系统 connect(example.com:80)
```

得到：

```text
targetConn = GOST ←→ example.com:80
```

### A5.2 配置 `-F`：经过代理节点

假设有两级代理：

```text
GOST → Node A → Node B → example.com:80
```

`chainRoute` 执行：

```text
Node A Transport.Dial
→ Node A Transport.Handshake
→ Node A Transport.Connect(Node B)
→ Node B Transport.Handshake
→ Node B Transport.Connect(example.com:80)
```

最后返回的 `net.Conn` 对上层仍像一条普通连接。`http.Transport` 不必知道底层穿过了几个节点。

## A6. 请求如何到达目标网站

从 GOST 代码看，调用点是 `RoundTrip`；具体 HTTP/1.x 写入由 Go 标准库 `net/http.Transport` 完成。

概念上，它会把代理请求调整为目标服务器能理解的请求。例如客户端发给代理的是：

```http
GET http://example.com/news?q=go HTTP/1.1
Host: example.com
```

目标服务器看到的请求行通常是：

```http
GET /news?q=go HTTP/1.1
Host: example.com
```

URL 的 scheme/host 用于选择连接目标；path/query 用于目标请求行；`Host` 用于 HTTP 虚拟主机。

## A7. 目标响应如何返回客户端

目标网站返回：

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: ...

<html>...</html>
```

Transport 把它解析成 `resp`。GOST 随后：

```go
defer resp.Body.Close()
err = resp.Write(clientConn)
```

`http.Response.Write` 会：

1. 写状态行；
2. 写 Header；
3. 写空行；
4. 从 `resp.Body` 读取内容并写到 `clientConn`。

完整回程：

```text
example.com
→ targetConn
→ http.Transport 解析为 *http.Response
→ resp.Write(clientConn)
→ 客户端读取状态、Header、Body
```

Body 不必先全部载入内存。`resp.Write` 可以边从目标响应 Body 读取，边写给客户端，形成流式转发。

## A8. 出错时返回什么

如果 `RoundTrip` 返回错误，预先构造的响应默认状态是：

```text
503 Service Unavailable
```

代码调用 `res.Write(clientConn)`，让客户端收到 HTTP 错误响应，而不是只在服务端日志中看到错误。

---

# 路径 B：HTTPS URL / CONNECT 隧道

## B1. 客户端为什么先发 CONNECT

客户端想访问：

```text
https://example.com/news
```

为了让代理建立到目标 443 端口的通道，先发送：

```http
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
```

此时 GOST 通常看不到 `/news`。真正的 HTTPS 请求在后续 TLS 加密通道内部。

## B2. GOST 先连接目标

文件：`x/handler/http/connect.go`

```go
cc, err := h.dial(ctx, "tcp", "example.com:443")
```

这里与普通 HTTP 走同一个 Router：

```text
h.dial
→ Router.Dial
→ DefaultRoute 或 chainRoute
→ 得到 targetConn(cc)
```

若失败，GOST 向客户端写 `503 Service Unavailable`。

## B3. 告诉客户端隧道建立成功

成功后写：

```http
HTTP/1.1 200 Connection established
Proxy-Agent: gost/3.0
```

收到 200 后，客户端开始在原来的 `clientConn` 上发送 TLS ClientHello。

## B4. 默认情况下只搬运字节

未启用 sniffing/MITM 时：

```go
xnet.Pipe(ctx, clientConn, targetConn, ...)
```

Pipe 同时进行两个方向的复制：

```text
客户端 TLS 字节 ─────────────→ 目标网站
客户端          ←───────────── 目标网站 TLS 字节
```

等价于同时运行两个 `io.Copy`：

```text
copy(targetConn, clientConn)
copy(clientConn, targetConn)
```

因此：

- TLS 握手主要发生在客户端与目标网站之间；
- GOST 知道 `example.com:443`，但通常不知道加密后的 `/news`；
- 目标网站响应也保持加密，GOST 只是转发字节；
- 客户端负责 TLS 解密并解析 HTTP 响应。

若启用 sniffing/MITM，路径会进入 `sniffAndHandle`，可能识别 TLS 并用证书进行协议感知处理。这是高级功能，不应与默认隧道混为一谈。

---

# 两条路径对照

| 问题 | HTTP URL | HTTPS URL（默认 CONNECT） |
| --- | --- | --- |
| GOST 是否看到完整 URL | 是 | 通常只看到 CONNECT 的 host:port |
| 出站建立方式 | `http.Transport` 需要连接时调用 `h.dial` | Handler 直接调用 `h.dial` |
| 谁发送 HTTP 请求 | GOST 的 `http.Transport` | 客户端在 TLS 隧道内发送 |
| 谁读取目标 HTTP 响应 | GOST 的 `http.Transport` | 客户端解密后读取 |
| 返回方式 | `resp.Write(clientConn)` | `xnet.Pipe` 双向复制加密字节 |
| 可否复用连接 | Transport 和客户端 keep-alive 管理 | 隧道生命周期内持续复用 |

# 一条完整的 HTTP 调用链

```text
cmd/gost.main
→ svc.Run(program)
→ program.Start
→ loader.Load
→ program.run
→ service.Serve                         x/service/service.go
→ listener.Accept                       x/listener/tcp/listener.go
→ handler.Handle                        x/handler/http/handler.go
→ http.ReadRequest
→ handleRequest
→ normalizeRequest                      x/handler/http/util.go
→ handleProxy                           x/handler/http/proxy.go
→ proxyRoundTrip
→ http.Transport.RoundTrip
→ Transport.DialContext = h.dial
→ Router.Dial                           x/chain/router.go
→ Route.Dial                            x/chain/route.go
→ 操作系统连接目标（或经过代理链）
→ http.Transport 写 request、读 response
→ proxyRoundTrip 得到 *http.Response
→ response.Write(clientConn)
→ 客户端收到响应
```

# 一条完整的 HTTPS 调用链

```text
service.Serve
→ listener.Accept
→ handler.Handle
→ http.ReadRequest(CONNECT example.com:443)
→ handleRequest
→ handleConnect                         x/handler/http/connect.go
→ h.dial
→ Router.Dial
→ Route.Dial
→ 得到 targetConn
→ clientConn.Write("200 Connection established")
→ xnet.Pipe(clientConn, targetConn)
→ TLS 字节双向流动
```

# 如何实际验证

## 打开 trace 日志

```bash
go run ./cmd/gost -DD -L http://127.0.0.1:8080
```

另一个终端：

```bash
curl -v -x http://127.0.0.1:8080 http://example.com/
curl -v -x http://127.0.0.1:8080 https://example.com/
```

观察差异：

- HTTP 请求日志中会出现 `GET http://...`；
- HTTPS 首先出现 `CONNECT example.com:443`；
- HTTPS 建立后，若未开启 sniffing，后续是不可直接按 HTTP 解读的 TLS 数据。

## 用本地目标站避免互联网干扰

终端一：

```bash
python3 -m http.server 9000
```

终端二：

```bash
go run ./cmd/gost -DD -L http://127.0.0.1:8080
```

终端三：

```bash
curl -v -x http://127.0.0.1:8080 http://127.0.0.1:9000/
```

这能清楚区分两个 TCP 连接：

```text
curl:随机端口 → 127.0.0.1:8080
GOST:随机端口 → 127.0.0.1:9000
```

# 阅读时最常见的误区

1. **“服务器得到 URL 后直接调用 `net.Dial(URL)`。”**  
   错。Dial 使用 `host:port`；path/query 由 HTTP Transport 编码进请求。

2. **“HTTP 和 HTTPS 都由 `RoundTrip` 转发。”**  
   错。默认 HTTPS 走 CONNECT + Pipe，GOST 不解析隧道内的加密 HTTP。

3. **“`directDialer` 就是直接连接目标。”**  
   在这套 Transport 设计中不完整。真实直连可能走 `DefaultRoute`，而 direct chain transport 的目标连接发生在 `directConnector.Connect`。

4. **“`Response.Write` 只是写 Header。”**  
   它也会读取并写出 Body，因此是响应返回客户端的关键调用。

5. **“只有一条 net.Conn。”**  
   代理至少有入站和出站两条逻辑连接；经过代理链时，底层还可能包含更多节点连接或复用流。

# 本专题的最终答案

服务器取得 URL 后，不是由 `program` 或 `go-svc` 访问网站。真正流程是：

```text
HTTP Handler 从客户端连接解析 URL
→ 提取 host:port
→ 通过 Router 选择直连或代理链
→ 建立出站连接
```

对于普通 HTTP，Go 的 `http.Transport.RoundTrip` 在出站连接上发送请求并读出 `http.Response`，随后 GOST 调用 `resp.Write(clientConn)` 把状态行、Header 和 Body 返回客户端。

对于 HTTPS，GOST 根据 CONNECT 的 `host:port` 建立出站 TCP 连接，回复 `200 Connection established`，再通过双向 Pipe 在客户端和目标网站之间转发 TLS 字节。默认情况下，HTTPS URL 的 path 和响应内容都由客户端在加密隧道中处理。
