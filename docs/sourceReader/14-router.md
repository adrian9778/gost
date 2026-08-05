# 14. 专题补充：Router 的功能与完整调用链

Router 是 GOST 请求转发链中的“出站总入口”。Handler 已经知道目标地址后，通常不直接调用 `net.Dial`，而是调用 Router：

```go
conn, err := router.Dial(ctx, "tcp", "example.com:443")
```

Router 负责把“我要访问哪个地址”转换为“一条已经连接到目标、可以读写的 `net.Conn`”。

本文重点讲：

```text
github.com/go-gost/core/chain.Router
github.com/go-gost/x/chain.Router
```

## 1. 项目中有两个不同的 Router

这是阅读时最容易混淆的地方。

| 名称 | import 路径 | 核心方法 | 主要用途 |
| --- | --- | --- | --- |
| 出站连接 Router | `core/chain.Router` | `Dial`、`Bind` | Handler/Listener 访问目标或创建反向监听 |
| 系统路由表 Router | `core/router.Router` | `GetRoute` | TUN、router handler 查询目标的 gateway |

### 出站连接 Router

```go
import "github.com/go-gost/core/chain"

type Router interface {
    Options() *RouterOptions
    Dial(ctx, network, address, opts...) (net.Conn, error)
    Bind(ctx, network, address, opts...) (net.Listener, error)
}
```

本篇前半部分说的 Router 都是它。

### 系统路由表 Router

```go
import "github.com/go-gost/core/router"

type Router interface {
    GetRoute(ctx, dst, opts...) *Route
}
```

它查询的是：

```text
目标 IP/网段 → gateway
```

不是建立目标连接。本文最后单独说明。

## 2. 出站 Router 在请求链的位置

```text
客户端
  ↓
Listener.Accept
  ↓
Handler.Handle
  ↓ 得到 network + address
Router.Dial
  ├─ resolver / hosts
  ├─ timeout / retries
  ├─ Chainer.Route
  ├─ interface / netns / socket mark
  └─ Route.Dial
       ├─ DefaultRoute：直连
       └─ chainRoute：经过代理节点
  ↓
net.Conn
  ↓
Handler 转发请求或 Pipe 字节
```

它位于应用协议和具体连接路线之间：

- Handler 理解 HTTP、SOCKS5、DNS 等入站协议；
- Router 理解出站策略；
- Route/Transport 理解具体怎么建立连接。

## 3. 为什么 Handler 不直接使用 `net.Dial`

若每个 Handler 自行拨号，就要重复实现：

- DNS 与 hosts 映射；
- 多级代理链；
- 重试和超时；
- 指定出站网卡；
- network namespace；
- Linux socket mark；
- route 日志；
- recorder；
- UDP 连接适配。

Router 把这些横切能力集中起来。HTTP、SOCKS5、端口转发等 Handler 只需表达：

```text
network = tcp
address = example.com:443
```

## 4. Router 在哪里创建并注入

文件：`x/config/parsing/service/parse.go`

每个 Service 通常创建两套独立的 `x/chain.Router`。

### Listener Router

```go
listener.RouterOption(
    xchain.NewRouter(listenerRouterOptions...),
)
```

用于需要主动连接、反向绑定或上游通信的 Listener。它使用：

- service resolver/hosts；
- listener chain；
- interface/netns/socket mark；
- dial timeout。

### Handler Router

```go
handler.RouterOption(
    xchain.NewRouter(handlerRouterOptions...),
)
```

它负责普通客户端请求的出站连接，额外包含：

- `cfg.Handler.Retries`；
- Handler chain/chain group；
- recorders。

第 8 篇 HTTP 专题中的：

```go
h.options.Router.Dial(...)
```

就是这里注入的 Handler Router。

### 为什么不是一个全局 Router

每个 Service 的配置可能不同：

```text
Service A → 直连、resolver-A、网卡 eth0
Service B → chain-B、resolver-B、网卡 eth1
```

Router 跟随 Service 构造，使每个监听服务拥有独立出站策略。

## 5. RouterOptions

`core/chain.RouterOptions` 保存：

| 字段 | 作用 |
| --- | --- |
| `Retries` | 失败后的额外重试次数 |
| `Timeout` | 每次 Dial/Bind 尝试的超时 |
| `IfceName` | 绑定出站 interface/IP |
| `Netns` | 使用的 network namespace |
| `SockOpts.Mark` | Linux `SO_MARK` |
| `Chain` | Chainer，负责选择代理 Route |
| `Resolver` | 自定义 DNS 解析器 |
| `HostMapper` | 静态 host 映射 |
| `Recorders` | 记录拨号地址和错误 |
| `Logger` | Router 日志 |

构造默认值：

```text
Timeout == 0 → 15 秒
Logger == nil → 默认 kind=router logger
```

这意味着配置没有显式 `dialTimeout` 时，并非无限等待，而是每次尝试默认最多约 15 秒。

## 6. `Dial` 的入口工作

文件：`x/chain/router.go`

```go
func (r *Router) Dial(
    ctx context.Context,
    network string,
    address string,
    opts ...chain.DialOption,
) (net.Conn, error)
```

例如：

```text
network = "tcp"
address = "example.com:443"
```

入口首先从 `host:port` 提取 host：

```text
example.com:443 → example.com
```

并尝试写入 `RecorderServiceRouterDialAddress`。

随后给 logger 增加当前 SID，调用内部 `r.dial`。

如果最终失败，再记录：

```text
RecorderServiceRouterDialAddressError
```

## 7. 重试次数如何计算

```go
count := Retries + 1
if count <= 0 {
    count = 1
}
```

因此：

| `Retries` | 总尝试次数 |
| ---: | ---: |
| 0 | 1 |
| 1 | 2 |
| 2 | 3 |
| 负数 | 至少 1 |

“Retries”表示失败后再试几次，不是总次数。

每次尝试会重新进行：

```text
创建 timeout context
→ Resolve
→ Chainer.Route
→ Route.Dial
```

所以重试可能重新选择 Node。

## 8. 每次尝试的 timeout

每轮：

```go
ctx, cancel = context.WithTimeout(parent, Timeout)
defer cancel()
```

内部匿名函数的作用是让每轮的 `defer cancel()` 在本轮结束时立即执行，而不是积累到整个重试循环结束。

最坏总时间近似：

```text
(Retries + 1) × Timeout
```

但 parent context 若更早取消，所有子 context 都会提前结束。

## 9. Resolve：从域名得到实际地址

```go
ipAddr, err := xnet.Resolve(
    ctx,
    "ip",
    address,
    Resolver,
    HostMapper,
    log,
)
```

它综合：

- 原 address；
- 自定义 Resolver；
- HostMapper；
- context；
- 日志。

例如：

```text
输入：example.com:443
输出：93.184.216.34:443
```

Router 后续通常把 `ipAddr` 传给 Route。

与此同时，选择 Route 时通过：

```go
chain.WithHostRouteOption(address)
```

保留原始地址，使 Node filter/matcher 仍有机会根据域名而不是只根据解析后的 IP 做选择。

## 10. Chainer 选择 Route

```go
route = Chain.Route(
    ctx,
    network,
    ipAddr,
    WithHostRouteOption(originalAddress),
)
```

Chainer 可能是：

- 单条 `x/chain.Chain`；
- 多条 Chain 的 `chainGroup`；
- nil。

Chain 会在每个 Hop 中选择 Node，生成 `chainRoute`。

如果：

```text
Router.options.Chain == nil
```

或 Chainer 返回 nil，Router 使用：

```go
DefaultRoute
```

也就是直连。

## 11. Route 日志如何生成

Router 将路线写入 buffer：

```text
node-0@proxy-a:8080 > node-1@proxy-b:1080 > 93.184.216.34:443
```

`routePath` 不只遍历当前 Route 的 Node，还递归展开 Multiplex Transport 中保存的前置 Route：

```go
routePath(tr.Options().Route)
```

因此日志可以还原被 route splitting 隐藏的完整物理/逻辑路径。

HTTP Handler 调用 Router 前把 `bytes.Buffer` 放入 context：

```go
Router.Dial(ContextWithBuffer(ctx, &buf), ...)
```

Router 填好后，Handler 把：

```text
ro.Route = buf.String()
```

写入 recorder object。

## 12. interface 的 `auto` 模式

Router 支持 `IfceName` 中的精确 `auto` token。

它从 context 的 `DstAddr` 读取入站连接本地地址：

```text
客户端连接进入 192.0.2.10:8080
→ DstAddr host = 192.0.2.10
→ auto 替换为 192.0.2.10
```

目的为“source-in source-out”：

> 从哪个本地地址接收到连接，就尽量从对应地址出站。

代码只替换逗号分隔列表中精确等于 `"auto"` 的元素，不替换任意子字符串。

若入站地址为：

```text
0.0.0.0、:: 或不存在
```

就不能推导具体接口地址，保留字面量并记录 debug 日志。

## 13. DialOptions 如何下传

Router 给 Route 追加：

```text
InterfaceDialOption
NetnsDialOption
SockOptsDialOption
LoggerDialOption
```

然后再追加调用者传入的 `callerOpts`：

```go
append(defaultOptions, callerOpts...)...
```

Functional Option 按顺序执行。如果调用者传入同类 option，后执行的 caller option 通常可以覆盖 Router 默认值。

## 14. 直连 Route

`DefaultRoute.Dial`：

```text
解析 DialOptions
→ 构造 internal/net/dialer.Dialer
→ 设置 Interface、Netns、Mark、Logger
→ netd.Dial(ctx, network, address)
```

最终由操作系统建立 socket：

```text
GOST → target
```

Router 本身不读写 HTTP，也不搬运响应；它只返回连接。

## 15. 代理链 Route

`chainRoute.Dial`：

```text
connect()
  → Dial 第一 Node
  → Handshake 第一 Node
  → 前一 Node Connect 下一 Node
  → 下一 Node Handshake
  → 重复
→ 最后 Node Connect 最终 target
→ 返回 net.Conn
```

Router 不需要知道 Node 使用 HTTP、SOCKS5、TLS 还是 WebSocket。协议差异被封装在 Transport 的 Dialer 和 Connector 中。

## 16. 返回 UDP 时的适配

Router.Dial 成功后检查：

```go
if network is udp && conn does not implement net.PacketConn {
    return &packetConn{conn}
}
```

这个 adapter：

- `ReadFrom` 调用底层 `Read`，地址使用 `RemoteAddr()`；
- `WriteTo` 忽略传入地址，调用底层 `Write`。

它让“基于连接的 UDP tunnel”在上层表现得像 `net.PacketConn`。

这不等价于一个可以向任意地址发送数据的原生 UDP socket；目标语义仍受底层 tunnel 约束。

## 17. Router 的 `Bind`

`Bind` 与 Dial 方向不同：

- Dial：主动访问目标；
- Bind：通过 Route 创建监听端点，等待远端连接，用于反向代理/隧道。

流程：

```text
Retries + 1 次
→ 每次创建 timeout context
→ Chain.Route
→ 配置了 Chain 但 Route 为空：ErrEmptyRoute
→ 无 Chain：DefaultRoute.Bind
→ 有 Chain：chainRoute.Bind
```

DefaultRoute.Bind 根据 network：

- TCP → `net.ListenTCP`；
- UDP → `net.ListenUDP` 后包装为 UDP Listener；
- Unix → `net.ListenUnix`。

chainRoute.Bind 则先连接最后代理节点，再调用最后 Transport 的 `Bind`。只有 Connector 实现 `connector.Binder` 才支持，否则返回 `ErrBindUnsupported`。

## 18. Dial 与 Bind 对空 Route 的不同处理

这是细节差异：

### Dial

Chainer 返回 nil：

```text
回退 DefaultRoute → 直连
```

### Bind

只要明确配置了 Chain，但 Route nil 或没有 Node：

```text
返回 ErrEmptyRoute
```

因为反向 Bind 若悄悄变成本地监听，可能与配置意图和安全边界完全不同。

## 19. Router 不负责什么

出站 Router 不负责：

- 从客户端读取 HTTP URL；
- HTTP/SOCKS5 客户端认证；
- 写 HTTP Response；
- 在连接之间 `Pipe`；
- 解释目标网站响应；
- 长期管理 Handler 的 client connection。

职责边界：

```text
Handler：决定要访问谁、怎样处理协议
Router：返回怎样到达目标的连接
Handler：http.RoundTrip 或 Pipe
```

## 20. 一次 HTTP GET 中的完整 Router 跟踪

```text
httpHandler.proxyRoundTrip
→ http.Transport.RoundTrip
→ Transport 需要新 TCP connection
→ DialContext = httpHandler.dial
→ h.options.Router.Dial("tcp", "example.com:80")
→ record dial address
→ Resolve("example.com:80")
→ Chain.Route 或 DefaultRoute
→ Route.Dial
→ 返回 net.Conn 给 http.Transport
→ Transport 写 HTTP request
→ Transport 读取 response
→ Handler 将 response 写回客户端
```

Router 的生命周期结束点是“返回连接”，不是“收到目标响应”。

## 21. 一次 HTTPS CONNECT 中的完整 Router 跟踪

```text
httpHandler.handleConnect
→ h.dial("tcp", "example.com:443")
→ Router.Dial
→ Resolve
→ Route.Dial
→ 返回 targetConn
→ Handler 向客户端写 200 Connection established
→ Pipe(clientConn, targetConn)
```

Router 同样只负责获得 targetConn。

## 22. 失败如何传播

```text
Resolve error
    ↓
本轮失败，可能重试

Route.Dial error
    ↓
Node marker/chain metrics 由 Route 层处理
    ↓
Router 记录 route(retry=N) error
    ↓
所有尝试失败
    ↓
记录 RouterDialAddressError
    ↓
返回 Handler
    ↓
HTTP Handler 通常向客户端返回 503
```

Router 不把所有错误转成统一错误类型，因此日志和 `errors.Is` 能否识别具体原因取决于 Resolver、Dialer、Connector 是否正确包装错误。

## 23. 同名的系统路由表 Router

文件：

- `core/router/router.go`
- `x/router/router.go`
- `x/config/parsing/router/parse.go`

它的接口：

```go
GetRoute(ctx, dst, IDOption(id))
```

返回：

```go
type Route struct {
    Dst     string
    Gateway string
}
```

用途主要包括：

- TUN listener/handler；
- router handler；
- 按目标 IP 查 gateway；
- Linux 上按需写入系统 route table。

它的数据来源可以是：

- 配置中的静态 routes；
- 文件；
- Redis；
- HTTP loader；
- HTTP plugin；
- gRPC plugin。

本地实现用 `sync.RWMutex` 保护路由切片，并可通过 ticker 周期 reload。

匹配规则按 slice 顺序遍历：

```text
route.Dst == dst
或 route.Net.Contains(dstIP)
```

从当前实现看没有显式执行 longest-prefix-match 排序。因此若多个 CIDR 同时匹配，配置顺序可能影响返回结果，阅读配置时需要关注。

### Linux 与其他平台

Linux：

```text
netlink.RouteReplace
```

把带合法 CIDR 和 gateway 的 route 写入系统路由表。

非 Linux：

```text
setSysRoutes 是 no-op
```

但内存中的 `GetRoute` 仍可工作。

### 为什么它与 chain.Router 不是一回事

```text
core/router.Router.GetRoute
    回答：这个目标对应哪个 gateway？

core/chain.Router.Dial
    执行：真正给我一条到目标的连接。
```

前者是路由表查询，后者是出站连接编排。

## 24. 调试 Router 的断点

主请求 Router：

```text
x/handler/http.(*httpHandler).dial
x/chain.(*Router).Dial
x/chain.(*Router).dial
x/internal/net.Resolve
x/chain.(*Chain).Route
x/chain.(*defaultRoute).Dial
x/chain.(*chainRoute).Dial
x/chain.(*chainRoute).connect
```

观察：

```text
network
address
ipAddr
r.options.Retries
r.options.Timeout
r.options.Chain
route.Nodes()
ifceName
conn.LocalAddr()
conn.RemoteAddr()
err
```

系统路由表 Router：

```text
x/router.(*localRouter).reload
x/router.(*localRouter).GetRoute
x/router.(*localRouter).setSysRoutes
```

## 25. Router 故障定位表

| 现象 | 优先检查 |
| --- | --- |
| 域名失败、IP 成功 | Resolver、HostMapper、Resolve 日志 |
| 所有请求固定约 15 秒失败 | 默认 Router timeout |
| 配置 retries 后等待很久 | 总尝试次数为 retries+1 |
| 没走预期代理 | Handler chain 注入、Chainer.Route、Node filter |
| 意外直连 | Chainer 返回 nil 后 Dial 回退 DefaultRoute |
| Bind 返回 empty route | 配置了 Chain 但没有可用 Node |
| 多网卡出口错误 | interface、auto、context DstAddr |
| Linux 策略路由不生效 | netns、SO_MARK、权限和系统规则 |
| UDP 上层类型断言失败 | Router 的 packetConn 适配路径 |
| route recorder 为空 | context buffer、recorders 配置、调用点 |
| TUN gateway 不对 | 检查 `core/router.Router`，不是 chain.Router |

## 26. Router 专题验收

- [ ] 能区分两个同名 Router。
- [ ] 能说明 Handler Router 在 ParseService 中如何注入。
- [ ] 能列出 RouterOptions 的主要字段。
- [ ] 能从 Dial 讲到 Resolve、Route 选择和 Route.Dial。
- [ ] 能解释没有 Route 时为什么会直连。
- [ ] 能计算 retries 与总尝试次数。
- [ ] 能解释原始域名与解析后 IP 为什么都要保留。
- [ ] 能说明 interface `auto` 的来源。
- [ ] 能说明 UDP `packetConn` adapter 的限制。
- [ ] 能解释 Dial 和 Bind 对空 Route 的不同处理。
- [ ] 能明确 Router 返回连接后，由 Handler 继续 RoundTrip 或 Pipe。
