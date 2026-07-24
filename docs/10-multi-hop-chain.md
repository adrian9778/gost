# 10. 阶段 4：多级代理链的执行过程

本篇聚焦：

> 配置多个 `-F` 后，一条到目标网站的连接如何依次穿过多个代理节点？

## Hop 和 Node 不是一回事

```text
Chain
├─ Hop 1
│   ├─ Node A
│   └─ Node B
└─ Hop 2
    ├─ Node C
    └─ Node D
```

- Chain 是按顺序经过的整条链；
- Hop 是链上的一个位置；
- 一个 Hop 可以包含多个候选 Node；
- selector 在每个 Hop 选择一个 Node；
- 一次 Route 可能得到 `Node B → Node C`。

因此多个 `-F` 默认形成多个 Hop，而同一 Hop 中的多个 Node 更偏向负载均衡/故障切换。

## Route 在每次 Dial 前生成

`x/chain.Router.dial`：

```text
解析目标地址
→ Chainer.Route(ctx, network, address)
→ Route.Dial(ctx, network, resolvedAddress)
```

`x/chain.Chain.Route` 遍历 Hop：

```go
node := h.Select(ctx,
    NetworkSelectOption(network),
    AddrSelectOption(address),
    HostSelectOption(originalHost),
)
```

选择可以考虑：

- 网络类型；
- 最终地址；
- 原始域名；
- selector 策略；
- Node 的失败标记；
- filter/matcher。

选出的 Node 被依次加入 `chainRoute.nodes`。

## 三个动作的精确定义

每个 Node 包含一个 `Transport`：

```text
Transport
├─ Dialer
└─ Connector
```

Transport 暴露：

```text
Dial(addr)                 连接当前代理节点
Handshake(conn)            建立当前节点的传输/协议会话
Connect(conn, nextAddr)     命令当前节点连接下一跳
```

关键点：

> 只有第一个节点是 GOST 直接 Dial 到的；后续节点由前一个节点的 Connector 帮忙连接。

## 两级代理的逐行级流程

假设：

```text
Node A = HTTP 代理 10.0.0.1:8080，传输 TCP
Node B = SOCKS5 代理 10.0.0.2:1080，传输 TLS
Target = example.com:443
```

### 第一步：连接 Node A

`chainRoute.connect`：

```go
cc, err := nodeA.Transport.Dial(ctx, nodeA.Addr)
cn, err := nodeA.Transport.Handshake(ctx, cc)
```

TCP Dialer：

```text
操作系统 TCP connect → 10.0.0.1:8080
```

TCP 没有额外 Handshake，因此 `cn` 仍是到 Node A 的 TCP 连接。

### 第二步：让 Node A 连接 Node B

循环处理后续节点：

```go
cc, err = nodeA.Transport.Connect(
    ctx, cn, "tcp", "10.0.0.2:1080",
)
```

Node A 使用 HTTP Connector，因此发送：

```http
CONNECT 10.0.0.2:1080 HTTP/1.1
Host: 10.0.0.2:1080
```

Node A 返回 200 后，`cc` 表面上仍是原连接，但现在它是一条：

```text
GOST → Node A → Node B
```

的透明字节通道。

### 第三步：与 Node B 做 TLS Handshake

```go
cc2, err := nodeB.Transport.Handshake(ctx, cc)
```

Node B 的 TLS Dialer 不会再次进行物理 Dial；此处在现有通道上执行 TLS Client Handshake：

```text
GOST
  → HTTP tunnel through A
    → TLS session with B
```

TLS wrapper 返回的仍实现 `net.Conn`，后续读写自动加解密。

### 第四步：让最后节点连接 Target

`chainRoute.Dial` 在 `connect` 返回后：

```go
last.Transport.Connect(
    ctx, conn, "tcp", "example.com:443",
)
```

Node B 使用 SOCKS5 Connector：

```text
发送 SOCKS5 CONNECT example.com:443
→ 读取 SOCKS5 reply
→ 成功后返回 conn
```

最终连接的逻辑路径：

```text
GOST
→ TCP to Node A
→ HTTP CONNECT Node B
→ TLS to Node B
→ SOCKS5 CONNECT Target
→ example.com:443
```

上层 HTTP Handler 只拿到一个 `net.Conn`，不需要知道这些细节。

## `chainRoute.connect` 的资源清理

任一步失败都会尽量关闭已经建立的连接：

- 首节点 Dial 失败：直接返回；
- Handshake 失败：关闭原连接；
- 连接下一 Node 失败：关闭当前连接；
- 下一 Node Handshake 失败：关闭新旧连接；
- 最终目标 Connect 失败：`chainRoute.Dial` 关闭 chain conn。

同时 Node 的 marker 被 `Mark()`，成功则 `Reset()`。Selector 可以利用失败标记暂时避开故障节点。

## Router 的重试边界

Router：

```text
尝试次数 = Retries + 1
每次尝试：
  → 新建 timeout context
  → 重新解析
  → 重新选择 Route
  → Route.Dial
```

这意味着重试时可能选择同一 Node，也可能由 selector 选到其他 Node，取决于策略和 marker。

超时 context 只有在下层网络操作正确响应 context 时才能及时中止。代码仍为部分握手设置 socket deadline，构成第二层保护。

## DNS 在哪里解析

存在两类地址：

1. 最终目标：Router 使用 service resolver/hosts 解析；
2. 每个 Node：`chainRoute.connect` 使用 Node 自己的 resolver/hosts 解析。

若希望代理节点远程解析目标域名，需要结合协议和配置确认传给 Connector 的地址是否仍含域名。当前 Router 在选路后将解析得到的 `ipAddr` 交给 Route，因此默认链路可能已经在本地完成目标解析。

不要仅用“SOCKS5 支持域名”推断当前配置一定远端 DNS；必须看 Router 传入 Connector 的实际 address。

## Multiplex 路由切分

某些 Dialer 实现 `Multiplexer`，能在一个物理连接上承载多个逻辑流。

`Chain.Route` 遇到 `Transport.Multiplex() == true` 时：

1. Copy 当前 Transport；
2. 把之前已形成的 Route 放进 `Transport.Options().Route`；
3. Copy Node 并换入新的 Transport；
4. 新建 Route，从这个 multiplex Node 重新开始。

随后 `Transport.Dial` 构造内部 net dialer 时，如果自身带前置 Route：

```go
netd.DialFunc = previousRoute.Dial
```

含义是 multiplex 节点的底层连接仍可经过前置代理链，但建立后可以复用，避免每个逻辑请求重复搭建全部物理通道。

这是本阶段最难的部分。初学时先掌握非 multiplex 的顺序链，再回来理解路由切分。

## Chain Group

一个 Service 可以配置多个 Chain。`chainGroup` 用 selector 先选一条 Chain：

```text
Handler Router
→ chainGroup.Route
→ selector 选择 Chain
→ Chain.Route 在每个 Hop 选择 Node
```

两级选择不要混淆：

- Chain selector：多条完整路线中选一条；
- Hop selector：某个位置的多个 Node 中选一个。

## 推荐日志解读方法

Router 会把 Route 写成：

```text
nodeA@10.0.0.1:8080 > nodeB@10.0.0.2:1080 > targetIP:443
```

`httpHandler.dial` 用 context buffer 收集这条字符串，并放进 recorder object。

阅读日志时：

- `dial target/network`：Handler 想访问的最终目标；
- `route(retry=N)`：本次尝试的实际节点路径；
- `connect address/network`：某个 Connector 正请求其代理节点访问下一地址。

## 阶段练习

```bash
go run ./cmd/gost \
  -F 'http://127.0.0.1:18080' \
  -F 'socks5+tls://127.0.0.1:11080' \
  -L 'http://127.0.0.1:8080' \
  -O yaml
```

先只看展开配置，回答：

1. 有几个 Hop？
2. 每个 Node 的 Dialer 和 Connector 是什么？
3. 哪个 Connector 负责连接第二节点？
4. 哪个 Connector 负责连接最终目标？

## 阶段完成标准

- [ ] 能区分 Chain、Hop、Node。
- [ ] 能解释为什么只有第一 Node 由 Dialer 直接连接。
- [ ] 能按顺序讲出 `Dial → Handshake → Connect`。
- [ ] 能画出两级代理的嵌套协议。
- [ ] 能说明重试会重新进行 Route 选择。
- [ ] 能区分 Chain selector 和 Hop selector。
- [ ] 对 multiplex 的前置 Route 注入有基本认识。
