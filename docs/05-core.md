# 05. `github.com/go-gost/core`：核心接口与抽象

本文对应版本：`github.com/go-gost/core v0.4.3`。

`core` 基本不负责完成具体协议，而是定义各组件之间的契约。阅读它的目标是建立一套“名词表”，之后看到 `x` 中的具体类型时，能知道该类型处于请求链的哪一层。

## 如何找到本机源码

```bash
go list -m -f '{{.Dir}}' github.com/go-gost/core
```

笔记中使用 `core/...` 表示上述命令返回目录中的相对路径。

## 请求处理所需的核心接口

```text
Service
├─ Listener：接受客户端连接
└─ Handler：处理一个客户端连接
     └─ Router：连接目标地址
          ├─ Chainer：选择一条 Route
          └─ Route：沿路由连接目标
               └─ Node.Transporter
                    ├─ Dialer：连接代理节点
                    └─ Connector：通过该节点请求下一跳/目标
```

## `service.Service`：一个可运行的代理服务

文件：`core/service/service.go`

```go
type Service interface {
    Serve() error
    Addr() net.Addr
    Close() error
}
```

它代表顶层运行单元。例如：

```yaml
services:
  - name: service-0
    addr: :8080
    handler:
      type: http
    listener:
      type: tcp
```

配置最终会产生一个 `Service`：

- `Serve` 启动 accept 循环；
- `Addr` 返回监听地址；
- `Close` 关闭 listener 等资源，让 `Serve` 返回。

注意有两个同名但不同的 `Service`：

| 接口 | 所在包 | 作用 |
| --- | --- | --- |
| `svc.Service` | `github.com/judwhite/go-svc` | 整个操作系统进程的 Init/Start/Stop |
| `service.Service` | `github.com/go-gost/core/service` | 一个 GOST 监听服务的 Serve/Addr/Close |

当前仓库的 `program` 实现前者；`x/service.defaultService` 实现后者。

## `listener.Listener`：接收客户端 TCP 连接

文件：`core/listener/listener.go`

```go
type Listener interface {
    Init(metadata.Metadata) error
    Accept() (net.Conn, error)
    Addr() net.Addr
    Close() error
}
```

它与标准库 `net.Listener` 很像，多了 `Init`。`Init` 让 loader 可以先构造 listener，再用配置 metadata 初始化它。

HTTP 代理使用 TCP listener 时，`Accept()` 返回的 `net.Conn` 代表：

```text
客户端浏览器  ←→  GOST 监听端口
```

它不是 GOST 到目标网站的连接。目标连接稍后由 Router 创建。整个请求同时涉及两条连接：

```text
clientConn: 浏览器 ←→ GOST
targetConn: GOST   ←→ 目标网站
```

## `handler.Handler`：解释客户端协议

文件：`core/handler/handler.go`

```go
type Handler interface {
    Init(metadata.Metadata) error
    Handle(context.Context, net.Conn, ...HandleOption) error
}
```

Listener 只负责接受字节流，不知道字节是 HTTP 还是 SOCKS5。Handler 负责：

- 从 `net.Conn` 读取协议报文；
- 认证客户端；
- 找出目标地址；
- 通过 Router 建立目标连接；
- 在客户端与目标之间转发数据。

对于专题中的 HTTP 请求，具体实现是 `x/handler/http.httpHandler`。

## `chain.Router`：Handler 的统一出口

文件：`core/chain/router.go`

```go
type Router interface {
    Options() *RouterOptions
    Dial(ctx context.Context, network, address string, opts ...DialOption) (net.Conn, error)
    Bind(ctx context.Context, network, address string, opts ...BindOption) (net.Listener, error)
}
```

Handler 不直接调用 `net.Dial`，而是调用：

```go
conn, err := router.Dial(ctx, "tcp", "example.com:80")
```

Router 在背后统一处理：

- DNS 或 hosts 映射；
- 选择直连还是代理链；
- 超时和重试；
- 指定网卡、network namespace、socket mark；
- 路由记录和指标。

这种设计让 HTTP、SOCKS5、端口转发等 Handler 复用同一套路由能力。

## `chain.Chainer` 与 `chain.Route`

文件：

- `core/chain/chain.go`
- `core/chain/route.go`

`Chainer.Route` 根据目标选择一条路线：

```go
Route(ctx, network, address, opts...) Route
```

`Route.Dial` 沿选中的路线连接目标：

```go
Dial(ctx, network, address, opts...) (net.Conn, error)
```

可以用导航类比：

- Chainer 是路线规划器；
- Route 是已经选定的一条路线；
- Node 是路线上的代理节点；
- 最后的 `address` 是目标网站。

没有配置代理链时，`x/chain.DefaultRoute` 直接连接目标。

## `chain.Node` 与 `Transporter`

文件：

- `core/chain/node.go`
- `core/chain/transport.go`

Node 保存：

- 节点名字和地址；
- Transporter；
- resolver、hosts、bypass；
- 节点选择和 HTTP/TLS 设置。

Transporter 是一个节点的实际通信能力：

```go
type Transporter interface {
    Dial(ctx, addr) (net.Conn, error)
    Handshake(ctx, conn) (net.Conn, error)
    Connect(ctx, conn, network, address) (net.Conn, error)
    // ...
}
```

三个步骤要严格区分：

1. `Dial`：先建立到代理节点的底层连接；
2. `Handshake`：完成 TLS、WebSocket 等传输层握手；
3. `Connect`：通过 HTTP CONNECT、SOCKS5 CONNECT 等代理协议，请求节点连接下一跳或最终目标。

## `dialer.Dialer` 与 `connector.Connector`

文件：

- `core/dialer/dialer.go`
- `core/connector/connector.go`

Dialer：

```go
Dial(ctx, proxyNodeAddr, opts...) (net.Conn, error)
```

Connector：

```go
Connect(ctx, connToProxyNode, network, targetAddr, opts...) (net.Conn, error)
```

例如代理链节点是 `socks5://proxy.example:1080`，目标是 `example.com:443`：

```text
Dialer.Dial
    建立 GOST → proxy.example:1080 的 TCP 连接

Connector.Connect
    在这条连接上发送 SOCKS5 CONNECT example.com:443
```

直连是一种特殊情况：`directConnector` 不使用“到代理节点的连接”，而是用传入的网络 dialer 直接连接目标。

## 为什么大量使用 `net.Conn`

`net.Conn` 的核心是：

```go
Read([]byte) (int, error)
Write([]byte) (int, error)
Close() error
```

TCP、TLS、WebSocket 隧道以及各种 wrapper 都可以表现为 `net.Conn`。上层代码因此能用统一方式读写，而不必知道底层经过几层包装。

这种模式也解释了为何流量统计、限速等功能常写成 wrapper：

```text
真实 TCP Conn
→ metrics Conn
→ limiter Conn
→ stats Conn
→ Handler
```

每一层仍实现 `net.Conn`，但在 `Read`/`Write` 前后附加功能。

## 接口值与 nil

阅读 `core` 时要注意 Go 的接口值由“动态类型 + 动态值”组成。一个装有 nil 指针的接口不一定等于 nil：

```go
var p *MyRouter = nil
var r chain.Router = p

// r != nil
```

因此代码中的 `if router == nil` 只能判断完全为 nil 的接口。实际项目一般通过构造流程避免把带 nil 指针的接口放进去。

## `registry.Registry[T]`：泛型注册表契约

文件：`core/registry/registry.go`

```go
type Registry[T any] interface {
    Register(name string, v T) error
    Unregister(name string)
    IsRegistered(name string) bool
    Get(name string) T
    GetAll() map[string]T
}
```

`T any` 表示注册表可以保存任意指定类型。`core` 只定义行为，线程安全的具体实现位于 `x/registry`。

## 本章要记住的主线

```text
Listener.Accept
→ Handler.Handle
→ Router.Dial
→ Chainer.Route
→ Route.Dial
→ Transporter / DefaultRoute
→ 得到通往目标的 net.Conn
```

接口回答“需要具备什么能力”；具体如何实现，要进入 `github.com/go-gost/x`。
