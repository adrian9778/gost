# 09. 阶段 3：配置如何变成运行对象

本阶段回答：

> `-L`、`-F` 或 `gost.yml` 中的字符串，如何一步步变成正在监听端口的 Listener、处理请求的 Handler 和访问目标的 Router？

对应版本为 `github.com/go-gost/x v0.12.5`。

## 总流程

```text
命令行 flag
    │
    ├─ -C → 配置文件 / URL / stdin / 内联 JSON
    ├─ -L → service 短格式
    └─ -F → chain node 短格式
    ▼
parser.Init(Args)
    ▼
parser.Parse
    ├─ readConfig：读取各个 -C
    ├─ mergeConfig：从左到右合并
    ├─ cmd.BuildConfigFromCmd：转换 -L/-F
    └─ 环境变量和显式 CLI 开关覆盖部分字段
    ▼
*config.Config
    ▼
loader.Load
    ├─ 解析叶子依赖并注册
    ├─ ParseHop
    ├─ ParseChain
    └─ ParseService
         ├─ registry 查 Listener 构造器
         ├─ Listener.Init（此处绑定端口）
         ├─ registry 查 Handler 构造器
         ├─ Handler.Init
         └─ xservice.NewService
    ▼
ServiceRegistry
    ▼
program.run → Service.Serve
```

## 1. 输入如何进入 parser

当前仓库的第二个 `init()` 用标准库 `flag` 写入包级变量：

```text
-C → cfgFiles
-L → services
-F → nodes
-D → debug
-DD → trace
-api → apiAddr
-metrics → metricsAddr
```

`program.Init()` 将它们包装为：

```go
parser.Init(parser.Args{
    CfgFiles: cfgFiles,
    Services: services,
    Nodes: nodes,
    // ...
})
```

`parser.Init` 并不解析，只是替换包级 `defaultParser`：

```go
defaultParser = &parser{args: args}
```

这是“先配置对象，稍后执行”的模式。真正工作在 `program.Start → parser.Parse()`。

## 2. `-C` 支持哪些来源

文件：`x/config/parsing/parser/parser.go`

每个 `-C` 字符串由 `readConfig` 分类：

| 输入 | 识别方式 | 读取方式 |
| --- | --- | --- |
| `-` | 精确匹配 | stdin；首字节 `{` 为 JSON，否则 YAML |
| `{...}` | 首尾为大括号 | `json.Unmarshal` |
| `http://...` / `https://...` | URL scheme 与 host 合法 | HTTP GET |
| 其他 | 默认 | 本地配置文件 |

远程配置有明确边界：

- `http.Client.Timeout` 为 30 秒；
- 最大响应体为 10 MiB；
- 非 200 响应视为错误；
- 错误消息会移除 URL 中的用户名和密码；
- Content-Type 优先决定 JSON/YAML，其次看扩展名，最后默认 YAML。

这说明配置读取本身也是外部 I/O，可能在 `Start()` 中阻塞一段时间。`go-svc` 文档要求 Start 非阻塞，但这里的“不阻塞”更准确地理解为不能成为长期运行循环；配置文件和网络读取仍是同步完成的。

## 3. 多份配置如何合并

`parser.Parse` 对 `CfgFiles` 从左到右执行：

```go
cfg = mergeConfig(cfg, fileConfig)
```

合并规则分两类。

### 列表字段：追加

```text
Services、Chains、Hops、Authers、Bypasses……
cfg1 的元素 + cfg2 的元素
```

所以两份配置如果都定义同名对象，解析阶段不会去重。稍后 registry 注册时可能因重名得到 `ErrDup`。

### 单对象字段：后者非 nil 时覆盖

```text
TLS、Log、API、Metrics、Profiling
```

若右侧配置对应字段为 `nil`，保留左侧；非 nil 则整体替换，不是逐子字段深度合并。

## 4. `-L/-F` 如何转成结构化 Config

文件：`x/config/cmd/cmd.go`

入口：

```go
cmd.BuildConfigFromCmd(serviceList, nodeList)
```

### `-F` 形成默认 chain

只要存在至少一个 `-F`，就创建：

```text
chain-0
```

每个 `-F` 形成一个 Hop：

```text
-F 第 0 项 → hop-0
-F 第 1 项 → hop-1
```

每个 Hop 中可以因逗号地址形成多个 Node，并配置 selector。

### `-L` 形成 service

每个短格式 service 经过 URL 规范化、组件类型推断和 metadata 拆分，最终命名：

```text
service-0
service-1
...
```

若存在 `chain-0`，通常把它写入：

```text
service.Handler.Chain = "chain-0"
```

反向 listener（`rtcp`、`rudp`、`runix`）是例外，chain 写入 Listener。

### 查询参数如何分层

短格式 URL 的 query 最初是一个 map，然后按前缀分发：

```text
handler.xxx   → Handler.Metadata
listener.xxx  → Listener.Metadata
connector.xxx → Connector.Metadata
dialer.xxx    → Dialer.Metadata
hop.xxx       → Hop.Metadata
node.xxx      → Node.Metadata
无前缀字段    → 按 cmd 解析规则归属或继承
```

因此 query 参数不是最终业务对象的字段，而是先成为 `map[string]any`，随后在各组件 `Init(metadata.Metadata)` 中解释。

## 5. Config 是中间表示

文件：`x/config/config.go`

顶层 `Config` 主要包含：

```go
type Config struct {
    Services []*ServiceConfig
    Chains   []*ChainConfig
    Hops     []*HopConfig
    Authers  []*AutherConfig
    // ...
}
```

这些结构体只有数据，没有 accept 循环，也没有网络连接。它们相当于编译器中的中间表示：

```text
用户友好的 YAML/URL
→ 统一 Config 数据结构
→ 运行时接口对象
```

`-O yaml` 的输出点恰好在 Config 已生成、loader 尚未运行之时。因此它非常适合观察短格式的展开结果：

```bash
go run ./cmd/gost \
  -L 'http://user:pass@:8080' \
  -F 'socks5://proxy.example:1080' \
  -O yaml
```

## 6. loader 为什么必须按依赖顺序运行

`loader.register` 的顺序是：

```text
第一层：logger、auth、bypass、resolver、hosts、limiter……
第二层：hop
第三层：chain
第四层：service
```

依赖方向：

```text
Service
├─ 引用 Chain
│   └─ 引用 Hop
│       ├─ 引用 Resolver / Hosts / Bypass
│       └─ 包含 Node
└─ 引用 Auth / Limiter / Recorder / Observer
```

解析器大量使用：

```go
registry.SomeRegistry().Get(name)
```

若先解析 Service，按名字就取不到后面的依赖。

## 7. Hop 如何构建 Node

文件：`x/config/parsing/hop/parse.go`

`ParseHop` 对每个 Node：

1. 复制 Node metadata，避免继承过程直接污染原 map；
2. 从 Hop 继承 resolver、hosts、interface、so_mark、netns；
3. Node 自己的设置优先；
4. 设置默认 connector 为 `http`；
5. 设置默认 dialer 为 `tcp`；
6. 调用 `node_parser.ParseNode`；
7. 将所有 Node 与 selector 组装成 `xhop.Hop`。

这里出现暂时替换：

```text
保存原 Node.Metadata
→ 换成继承合并后的 metadata
→ ParseNode
→ 恢复原 metadata
```

目的是让 ParseNode 看见最终值，同时尽量不让 Config 长期携带解析副作用。

## 8. Node 如何构建 Transport

文件：`x/config/parsing/node/parse.go`

`ParseNode` 的关键步骤：

```text
ConnectorRegistry.Get(connectorType)
→ 调用 Connector 构造器
→ Connector.Init

DialerRegistry.Get(dialerType)
→ 调用 Dialer 构造器
→ Dialer.Init

xchain.NewTransport(dialer, connector)
→ chain.NewNode(name, addr, TransportNodeOption)
```

默认值：

```text
Connector = http
Dialer    = tcp
```

若类型没有在 `register.go` 中空白导入并通过 `init()` 注册，解析会返回：

```text
unregistered connector: ...
unregistered dialer: ...
```

这就是“空白导入 → registry → 配置解析”的闭环。

## 9. Chain 如何引用 Hop

文件：`x/config/parsing/chain/parse.go`

两种写法：

- Chain 中含 inline `Nodes` 或 Plugin：直接 `ParseHop`；
- 只有 Hop 名字：从 `HopRegistry` 取得已注册 Hop。

然后：

```go
c.AddHop(hop)
```

得到实现 `core/chain.Chainer` 的 `xchain.Chain`。

## 10. Service 的构造是最关键的一步

文件：`x/config/parsing/service/parse.go`

默认类型：

```text
Listener = tcp
Handler  = auto
```

### 构造 Listener Router

Parser 先从 registry 解析 resolver、hosts 和 chain，并建立：

```go
xchain.NewRouter(routerOptions...)
```

这个 Router 通过 `listener.RouterOption` 传给 Listener，供某些需要主动连接或反向通信的 Listener 使用。

### 构造并初始化 Listener

```text
ListenerRegistry.Get(type)
→ 调用工厂函数(listenOpts...)
→ ln.Init(listenerMetadata)
```

对 TCP Listener，`Init` 内会调用 Listen 并绑定端口。这意味着：

> 端口不是在 `Service.Serve()` 才绑定，而是在配置加载的 ParseService 阶段已经绑定。

这也解释了热重载为何必须先关闭旧 Service。

### 构造 Handler Router

Handler 有自己的一套 Router options：

- retries；
- dial timeout；
- 出站 interface/netns/socket mark；
- resolver/hosts；
- recorders；
- Handler 使用的 chain。

最终：

```go
handler.RouterOption(xchain.NewRouter(routerOpts...))
```

第 8 篇专题中的 `h.options.Router.Dial`，就是在这里注入的。

### 构造并初始化 Handler

```text
HandlerRegistry.Get(type)
→ 调用工厂函数(handler options...)
→ h.Init(handlerMetadata)
```

HTTP Handler 在 `Init` 中创建带 `DialContext: h.dial` 的 `http.Transport`。

### 组合 Service

最后：

```go
xservice.NewService(name, listener, handler, ...)
```

这时对象关系已完整：

```text
defaultService
├─ tcpListener
└─ httpHandler
    └─ Router
        └─ Chain
            └─ Hop
                └─ Node
                    └─ Transport(Dialer + Connector)
```

## 11. 具体例子：`-L` 加 `-F`

命令：

```bash
gost \
  -L 'http://:8080' \
  -F 'socks5://127.0.0.1:1080'
```

概念展开：

```text
Config
├─ Chains
│   └─ chain-0
│       └─ hop-0
│           └─ node-0
│               ├─ addr: 127.0.0.1:1080
│               ├─ connector: socks5
│               └─ dialer: tcp
└─ Services
    └─ service-0
        ├─ addr: :8080
        ├─ listener: tcp
        └─ handler: http
            └─ chain: chain-0
```

运行对象：

```text
defaultService
├─ tcpListener(:8080)
└─ httpHandler
    └─ Router
        └─ Chain(chain-0)
            └─ Hop(hop-0)
                └─ Node(127.0.0.1:1080)
                    └─ Transport
                        ├─ tcpDialer
                        └─ socks5Connector
```

请求到来后：

```text
HTTP Handler 得到 example.com:443
→ Router 选 chain-0
→ tcpDialer 连接 127.0.0.1:1080
→ socks5Connector 请求 CONNECT example.com:443
```

## 12. 配置优先级

从代码执行顺序可以得到：

1. 多个 `-C` 从左到右合并；
2. `-L/-F` 生成的列表追加到文件配置；
3. `GOST_LOGGER_LEVEL`、`GOST_API`、`GOST_METRICS`、`GOST_PROFILING` 覆盖相应字段；
4. `-D/-DD` 再覆盖日志级别，`-DD` 优先；
5. 显式 `-api/-metrics` 最后重建对应配置。

注意：列表通常是追加，不宜简单说“CLI 一定覆盖配置文件”。只有代码中明确重新赋值的单对象字段才是覆盖。

## 13. 热重载的非原子边界

loader 对 service 的策略是：

```text
先注销并关闭全部旧 Service
→ 逐个 ParseService 并注册新 Service
```

优点是释放端口，避免 `address already in use`。

代价是如果第 N 个新 Service 构造失败：

- 旧服务已经关闭；
- 前 N-1 个新服务已经注册；
- 后面的服务尚未构造；
- 当前状态是部分更新，不是事务式回滚。

这是生产系统设计中很重要的可靠性边界。

## 14. 阶段验收实验

### 查看短格式展开

```bash
go run ./cmd/gost \
  -L 'http://:8080?handler.keepalive=true' \
  -F 'socks5://127.0.0.1:1080' \
  -O yaml
```

### 对比多配置合并

```bash
go run ./cmd/gost -C first.yml -C second.yml -O yaml
```

重点观察：

- services/chains 是否追加；
- log/api 是否被后一个非 nil 对象整体替换；
- 同名对象在 loader 阶段是否产生注册错误。

### 建议断点

```text
parser.(*parser).Parse
cmd.BuildConfigFromCmd
loader.Load
loader.register
hop.ParseHop
node.ParseNode
service.ParseService
tcp.(*tcpListener).Init
http.(*httpHandler).Init
```

## 阶段 3 完成标准

- [ ] 能解释四种 `-C` 来源。
- [ ] 能说明列表追加与单对象覆盖的区别。
- [ ] 能用 `-O yaml` 查看 `-L/-F` 的结构化结果。
- [ ] 能解释 loader 的依赖注册顺序。
- [ ] 能指出 Dialer 和 Connector 在哪里由 registry 构造。
- [ ] 能指出 Handler 的 Router 在哪里注入。
- [ ] 能解释为何 TCP 端口在 ParseService 阶段已经绑定。
- [ ] 能说明热重载为什么不是完全原子操作。
