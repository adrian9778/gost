# 11. 阶段 4：HTTP、SOCKS5、TLS 与 WebSocket

GOST 的节点 URL 经常把协议写在一起，例如：

```text
socks5://proxy:1080
http+tls://proxy:443
socks5+wss://proxy:443
```

理解方式不是背 URL，而是把它拆成两层：

```text
Connector：代理节点如何帮我访问目标
Dialer：我如何与代理节点建立传输通道
```

## 组合矩阵

| 示例 | Connector | Dialer | 含义 |
| --- | --- | --- | --- |
| `http://proxy:8080` | HTTP | TCP | TCP 到代理，再发 HTTP CONNECT |
| `http+tls://proxy:443` | HTTP | TLS | TCP+TLS 到代理，再发 HTTP CONNECT |
| `socks5://proxy:1080` | SOCKS5 | TCP | TCP 到代理，再做 SOCKS5 |
| `socks5+ws://proxy:80` | SOCKS5 | WebSocket | WebSocket 通道内做 SOCKS5 |
| `socks5+wss://proxy:443` | SOCKS5 | WSS | TLS WebSocket 通道内做 SOCKS5 |

实际短格式支持情况应以 `cmd.BuildConfigFromCmd` 和 registry 为准。

## 1. TCP Dialer

文件：`x/dialer/tcp/dialer.go`

核心：

```go
conn, err := options.Dialer.Dial(ctx, "tcp", addr)
```

这里的 `options.Dialer` 是 `x/internal/net/dialer.Dialer`，可带：

- 指定 interface；
- netns；
- socket mark；
- 前置 Route 的自定义 DialFunc。

之后可设置 TCP keepalive，并按需包装 PROXY protocol。

TCP Dialer 不实现额外 `Handshake`，所以：

```text
Transport.Dial     → 建 TCP
Transport.Handshake → 原样返回
```

## 2. TLS Dialer

文件：`x/dialer/tls/dialer.go`

`Dial` 仍先建立普通 TCP：

```go
options.Dialer.Dial(ctx, "tcp", addr)
```

`Handshake` 才执行：

```go
tlsConn := tls.Client(conn, TLSConfig)
tlsConn.HandshakeContext(ctx)
```

把 Dial 和 Handshake 分开很重要，因为多级链中“到当前节点的底层通道”可能由前一代理节点建立，而 TLS 必须在那条现有通道上进行。

TLS Handshake timeout 通过 socket deadline 与 context 共同约束。失败时关闭原连接。

## 3. WebSocket Dialer

文件：`x/dialer/ws/dialer.go`

### Dial

先建立 TCP 到代理节点。

### Handshake

构造 Gorilla WebSocket Dialer，但覆盖 `NetDial`：

```go
NetDial: func(...) (net.Conn, error) {
    return existingConn, nil
}
```

因此 WebSocket 库不会另开 TCP，而是在 Transport 已有的连接上发送 HTTP Upgrade。

```text
TCP connection
→ HTTP Upgrade: websocket
→ 101 Switching Protocols
→ WebSocket framed connection
→ ws_util.Conn 适配为 net.Conn
```

`wss` 还为 WebSocket Dialer提供 TLS client config，因此层次为：

```text
TCP → TLS → HTTP Upgrade → WebSocket frames
```

返回的 wrapper 实现 `net.Conn`，Connector 不必关心每次 Write 会被封装成 WebSocket frame。

Keepalive goroutine 定时发送 Ping，Pong handler 延长 read deadline。写 Ping 失败后 goroutine 返回。

## 4. HTTP Connector

文件：`x/connector/http/connector.go`

HTTP Connector 在已经通往代理节点的 `conn` 上构造：

```http
CONNECT target.example:443 HTTP/1.1
Host: target.example:443
Proxy-Connection: keep-alive
Proxy-Authorization: Basic ...   # 可选
```

代码过程：

```text
http.Request.Write(conn)
→ http.ReadResponse(bufio.NewReader(conn), req)
→ 检查 StatusCode == 200
→ 返回同一个 conn
```

为什么返回同一个 conn？

HTTP CONNECT 成功后，代理服务器将后续字节透明转发到目标，所以原连接的语义发生了变化：

```text
成功前：GOST ↔ HTTP proxy
成功后：GOST ↔ HTTP proxy tunnel ↔ target
```

对于 UDP，GOST 使用扩展 Header `X-Gost-Protocol: udp`，并包装成 UDP tunnel conn；这不是标准 HTTP CONNECT 的通用行为。

## 5. SOCKS5 Connector

文件：`x/connector/socks/v5/connector.go`

它分为 Handshake 与 Connect。

### Handshake：协商认证方法

支持的方法可能包括：

- No Auth；
- Username/Password；
- GOST 扩展 TLS；
- GOST 扩展 TLS + Auth。

```go
cc := gosocks5.ClientConn(conn, selector)
cc.Handleshake()
```

返回的 `cc` 仍是连接 wrapper。

### Connect：请求目标

```text
将 target host:port 编码为 SOCKS5 Addr
→ 写 CmdConnect request
→ 读 reply
→ Rep == Succeeded
→ 返回 conn
```

如果 reply 非成功，当前代码统一返回 `host unreachable`，上层无法从该错误文本区分所有 SOCKS5 reply 原因。

### SOCKS5 与 HTTP CONNECT 对照

| 项目 | HTTP Connector | SOCKS5 Connector |
| --- | --- | --- |
| 首次方法协商 | 无 | 有 Handshake |
| 目标请求 | HTTP CONNECT 文本 | SOCKS5 二进制 request |
| 认证 | Proxy-Authorization | 方法协商后的认证子协议 |
| 成功判定 | HTTP 200 | Reply `Succeeded` |
| 成功后的连接 | 原连接成为 tunnel | 原连接成为 tunnel |

## 6. Connector Handshaker 与 Dialer Handshaker

`Transport.Handshake` 的顺序：

```go
if dialer implements Handshaker {
    conn = dialer.Handshake(conn)
}
if connector implements Handshaker {
    conn = connector.Handshake(conn)
}
```

例如 `socks5+tls`：

```text
TCP Dial
→ TLS Dialer Handshake
→ SOCKS5 Connector Handshake
→ SOCKS5 Connector Connect(target)
```

顺序不能互换：必须先建立安全传输，才能在其中发送 SOCKS5 方法协商和认证。

## 7. 协议套娃示例

`socks5+wss://proxy.example:443` 访问 `target.example:80`：

```text
物理层（简化）
└─ TCP
   └─ TLS
      └─ HTTP WebSocket Upgrade
         └─ WebSocket data frames
            └─ SOCKS5 handshake
               └─ SOCKS5 CONNECT target.example:80
                  └─ 目标 HTTP 字节
```

每一层通过 wrapper 继续暴露 `net.Conn`，上层只调用 `Read/Write`。

## 8. 入站 Handler 与出站 Connector 不要混淆

两者都可能叫 HTTP 或 SOCKS5：

```text
客户端 → GOST Handler → Router/Chain → Connector → 上游代理
```

- HTTP Handler：GOST 对下游客户端表现为 HTTP 代理；
- HTTP Connector：GOST 把上游节点当成 HTTP 代理；
- SOCKS5 Handler：GOST 对下游客户端表现为 SOCKS5 代理；
- SOCKS5 Connector：GOST 把上游节点当成 SOCKS5 代理。

例如：

```bash
gost -L http://:8080 -F socks5://proxy:1080
```

含义是：

```text
浏览器 --HTTP--> GOST --SOCKS5--> 上游代理 --> 目标
```

协议可以在入站和出站两侧不同。

## 9. HTTPS CONNECT 与 TLS Dialer 也不同

- HTTP Handler 的 CONNECT：客户端请求 GOST 建立到目标的隧道；
- TLS Dialer：GOST 与某个上游代理节点之间使用 TLS。

可能同时存在：

```text
浏览器
→ CONNECT target:443 到 GOST
→ GOST 用 TLS 连接上游代理
→ GOST 在 TLS 内用 SOCKS5 请求 target:443
→ 浏览器与 target 的 TLS 又运行在整个隧道内部
```

此时存在两层独立 TLS：

1. GOST ↔ 上游代理，用于保护代理传输；
2. 浏览器 ↔ 目标网站，用于 HTTPS。

## 10. Deadline 的层次

协议实现分别设置不同 deadline：

- Router：整个一次 Route.Dial 的 context timeout；
- TLS Dialer：TLS handshake deadline；
- WebSocket Dialer：upgrade handshake timeout/read deadline；
- HTTP Connector：CONNECT request/response deadline；
- SOCKS5 Connector：方法协商和 connect deadline；
- Pipe：空闲 read timeout。

完成握手后通常恢复：

```go
conn.SetDeadline(time.Time{})
```

零时间表示取消 deadline。若忘记恢复，连接可能在业务传输阶段意外超时。

## 11. 推荐的抓包顺序

从无加密组合开始：

1. `http+tcp`：能直接看到 CONNECT 文本；
2. `socks5+tcp`：能看到二进制握手；
3. `http+ws`：先看到 HTTP Upgrade，再看到 WebSocket frame；
4. 最后加 TLS/WSS，此时抓包只能看到 TLS record。

加密前先理解明文协议，学习成本更低。

## 阶段完成标准

- [ ] 能从节点 URL 拆出 Dialer 与 Connector。
- [ ] 能解释 TLS 为什么在 Handshake 而不全在 Dial。
- [ ] 能解释 WebSocket 如何复用已有 TCP connection。
- [ ] 能描述 HTTP CONNECT 的 request/response。
- [ ] 能描述 SOCKS5 方法协商与 CONNECT request。
- [ ] 能说出 Transport 中两个 Handshaker 的执行顺序。
- [ ] 能区分入站 Handler、出站 Connector 和传输 Dialer。
- [ ] 能解释两层 TLS 同时存在的场景。
