# 03. 生命周期与热重载：`program.go`

`program` 是整个进程的运行状态容器：

```go
type program struct {
    srvApi       service.Service
    srvMetrics   service.Service
    srvProfiling *http.Server
    cancel       context.CancelFunc
}
```

字段保存需要跨函数关闭或替换的资源。若只用局部变量，`Stop()` 就无法找到已经启动的服务器。

## 生命周期总览

```text
svc.Run
  ├─ Init：把命令行输入交给 parser
  ├─ Start
  │   ├─ Parse：输入 → Config
  │   ├─ config.Set：保存当前全局配置
  │   ├─ loader.Load：Config → Service Registry
  │   ├─ run：启动业务/API/metrics/pprof 服务
  │   └─ reload：等待 SIGHUP、定时器或取消
  └─ Stop：取消 reload，关闭全部服务
```

## `Init()`：传递输入，不做重活

```go
func (p *program) Init(env svc.Environment) error {
    parser.Init(parser.Args{...})
    return nil
}
```

`env` 没被使用在 Go 中是允许的，因为未使用参数不会造成编译错误；未使用的局部变量和 import 才会报错。

这一阶段只设置解析器输入，没有监听端口。生命周期函数保持职责单一：`Init` 准备，`Start` 真正启动。

## `Start()`：从文本到运行服务

### 1. 解析配置

```go
cfg, err := parser.Parse()
if err != nil {
    return err
}
```

这是 Go 最常见的错误处理方式。错误是普通返回值，调用方必须决定是处理、包装还是继续向上传递。这里选择直接返回，最终由 `svc.Run` 返回给 `main()`。

### 2. 只输出配置

如果指定 `-O yaml` 或 `-O json`，程序把解析、合并后的配置写到标准输出，然后退出。这非常适合验证命令行最终如何变成结构化配置。

### 3. 保存当前配置并启用 metrics

```go
config.Set(cfg)
```

它把配置保存到 `x/config` 管理的全局位置，供其他包读取。

metrics 必须在 `loader.Load` 之前启用，因为 listener wrapper 在初始化时就会检查 metrics 状态。这说明执行顺序不仅影响性能，也影响最终构造出的对象。

### 4. 加载服务

```go
if err := loader.Load(cfg); err != nil {
    return err
}
```

`loader` 读取配置，通过各类 registry 查找具体实现，构建业务服务，并把结果放进 service registry。当前仓库并不知道每一种协议的构造细节。

### 5. 启动服务

```go
if err := p.run(cfg); err != nil {
    return err
}
```

### 6. 启动重载 goroutine

```go
ctx, cancel := context.WithCancel(context.Background())
p.cancel = cancel
go p.reload(ctx)
```

`Start()` 不能被永久阻塞，所以长期等待信号的工作放到 goroutine。`cancel` 存入结构体，供 `Stop()` 通知 goroutine 退出。

## `run()`：并发启动四类服务

### 业务服务

```go
for _, svc := range registry.ServiceRegistry().GetAll() {
    svc := svc
    go func() {
        svc.Serve()
    }()
}
```

`Serve()` 通常会持续接受连接，是阻塞操作，所以每个服务都在独立 goroutine 运行。

`svc := svc` 是常见的“为本轮迭代创建局部副本”的写法。现代 Go 已改善 range 变量的闭包语义，但保留这行仍明确表达“每个 goroutine 使用自己的服务值”。

这里没有检查 `Serve()` 的返回错误，与 API 和 metrics 的处理不同。读源码时要区分：

- 函数是否返回错误；
- 调用方是否使用了这个错误；
- 错误是否可能已经由服务内部记录。

不能仅凭当前文件断言错误一定丢失，需要继续查看 `service.Service` 契约与实现。

### API 服务

热重载时先关闭旧服务：

```go
if p.srvApi != nil {
    p.srvApi.Close()
    p.srvApi = nil
}
```

再按新配置构建并启动。goroutine 内：

```go
defer s.Close()
if err := s.Serve(); !errors.Is(err, http.ErrServerClosed) {
    log.Error(err)
}
```

`defer` 在当前 goroutine 的函数返回时执行。服务器被正常关闭时常返回 `http.ErrServerClosed`，这是预期状态，不应作为故障打印。

`errors.Is` 比 `err == target` 更稳健，因为它能识别经过包装的错误链。

### Metrics 服务

结构与 API 服务近似，这种重复让各服务的生命周期一目了然。构建逻辑在 `buildMetricsService` 中。

### Profiling 服务

它直接使用标准库 `http.Server`：

```go
s := &http.Server{Addr: addr}
go s.ListenAndServe()
```

由于 `main.go` 空白导入了 `net/http/pprof`，默认 mux 已带有 pprof 路由。

安全提示：源码默认空地址时监听 `:6060`。在真实部署中，性能分析接口可能暴露运行信息，应结合网络边界和配置控制访问范围。

## `Stop()`：集中释放资源

停止顺序是：

1. 调用 `cancel()`，让 reload goroutine 退出；
2. 关闭 registry 中的所有业务服务；
3. 关闭 API；
4. 关闭 metrics；
5. 关闭 profiling。

每次调用前都检查 `nil`，这使得部分初始化失败或某些服务未配置时也能安全清理。

`Close()` 是否等待所有连接完成，取决于具体实现。标准库 `http.Server.Close()` 与 `Shutdown(ctx)` 语义不同：前者直接关闭监听器和活跃连接，后者提供优雅关闭流程。当前 profiling 使用的是 `Close()`。

## `reload()`：channel、ticker 与 select

### 信号 channel

```go
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGHUP)
```

channel 容量为 1，可以暂存一个信号，降低发送方因接收者暂时未就绪而丢失通知的风险。

`signal.Notify` 让系统的 `SIGHUP` 被发送到 `c`。Unix 服务常用 `SIGHUP` 表示重新加载配置。

### 可选 ticker 的技巧

```go
var ticker <-chan time.Time
if reload > 0 {
    t := time.NewTicker(reload)
    defer t.Stop()
    ticker = t.C
}
```

未启用自动重载时，`ticker` 是 `nil channel`。在 Go 中：

- 从 `nil channel` 接收会永久阻塞；
- `select` 会忽略当前不可能完成的 nil channel case。

因此不需要写两套 `select`，这是很实用的 Go 并发技巧。

### `select` 的三个出口

```go
select {
case <-c:
    // 收到 SIGHUP
case <-ticker:
    // 到达自动重载时间
case <-ctx.Done():
    return
}
```

`for` 让它持续等待。若多个 case 同时就绪，Go 会选择其中一个可执行 case，而不是保证从上到下优先。

`ctx.Done()` 在 `cancel()` 后关闭，接收立即完成，使 goroutine 返回。

值得进一步检查的一点是：函数注册了 `signal.Notify`，但返回前没有调用 `signal.Stop(c)`。对于进程生命周期内只启动一次的监听器通常影响有限；如果未来允许反复启动和停止同一个 `program`，就应重新审视信号订阅的清理。

## `reloadConfig()`：重新构建

```text
重新 Parse
→ 更新全局 config
→ loader.Load
→ p.run
```

它不是简单修改一个字段，而是重新解析和装载，再让 `run()` 替换管理类服务。

阅读热重载代码时必须继续确认 `loader.Load` 如何处理 registry 中的旧业务服务：是先关闭、覆盖、做差量更新，还是保留。仅阅读当前文件还不能得出结论。

## Functional Options 模式

构建 API 服务时：

```go
api_service.NewService(
    network,
    addr,
    api_service.PathPrefixOption(cfg.PathPrefix),
    api_service.AccessLogOption(cfg.AccessLog),
    api_service.AutherOption(auther),
)
```

前两个是必填参数，后面是一组 option。Functional Options 常用于构造参数较多且大部分可选的对象：

- 调用处有字段名般的可读性；
- 增加可选项时通常不用修改所有调用方；
- 可以在 option 函数内统一校验或设置默认值。

## API 与 metrics 的地址处理

默认使用 TCP：

```go
network := "tcp"
addr := cfg.Addr
```

若地址以 `unix://` 开头，就改用 Unix domain socket，并去掉协议前缀。这里的 `network` 和 `addr` 最终交给具体 service 构造器。

API 的认证器可以组合：

1. 从内联认证配置解析一个认证器；
2. 从 auther registry 按名字取另一个认证器；
3. 用 `AuthenticatorGroup(authers...)` 组合。

metrics 的实现则是后者覆盖前者。两者相似但并不完全相同，阅读时不要因代码外形相近就默认语义一致。

## 资源所有权表

| 资源 | 创建位置 | 保存位置 | 关闭位置 |
| --- | --- | --- | --- |
| 业务服务 | `loader.Load` | service registry | `Stop()`，重载细节需看 loader |
| API service | `buildApiService` | `p.srvApi` | `run()` 替换时、goroutine defer、`Stop()` |
| Metrics service | `buildMetricsService` | `p.srvMetrics` | `run()` 替换时、goroutine defer、`Stop()` |
| Profiling server | `run()` | `p.srvProfiling` | `run()` 替换时、goroutine defer、`Stop()` |
| reload ticker | `reload()` | 局部变量 | `defer t.Stop()` |
| reload goroutine | `Start()` | 通过 context 管理 | `Stop()` 调用 `cancel()` |

## 动手实验

```bash
# 输出合并配置，观察 Start 的提前退出分支
go run ./cmd/gost -L http://:8080 -O yaml

# 每 30 秒重载一次（另开终端观察日志）
go run ./cmd/gost -C gost.yml -R 30s

# Unix 系统中手动发送 SIGHUP
kill -HUP <gost-pid>
```

建议为 `Start`、`run`、`reload` 各画一次“输入—处理—输出/副作用”表。网络程序的关键往往不在返回值，而在监听端口、启动 goroutine、更新注册表等副作用。
