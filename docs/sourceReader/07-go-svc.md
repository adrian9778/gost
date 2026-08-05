# 07. `github.com/judwhite/go-svc`：进程生命周期框架

本文对应版本：`github.com/judwhite/go-svc v1.2.1`。

`go-svc` 不参与 HTTP 请求转发。它处于更外层，负责让 GOST 进程在控制台和 Windows Service 环境中以统一方式启动、等待退出信号并停止。

## 找到源码

```bash
go list -m -f '{{.Dir}}' github.com/judwhite/go-svc
```

关键文件：

- `svc.go`：接口定义；
- `svc_common.go`：非 Windows 的 `Run`；
- `svc_windows.go`：Windows Service 与控制台逻辑。

## `svc.Service` 接口

```go
type Service interface {
    Init(Environment) error
    Start() error
    Stop() error
}
```

GOST 的 `program` 实现了这个接口：

```text
program.Init  → 把 CLI 输入交给 parser
program.Start → 解析配置、构建并启动代理服务
program.Stop  → 关闭所有代理服务
```

`Init` 和 `Start` 必须非阻塞。GOST 因此在 `program.run()` 中用 goroutine 启动各个会阻塞的 `Service.Serve()`。

## Unix/非 Windows 上的精确流程

`svc.Run(p)` 的顺序是：

```text
p.Init(environment{})
    │ error → 立即返回，不调用 Start/Stop
    ▼
p.Start()
    │ error → 立即返回，不调用 Stop
    ▼
准备默认信号 SIGINT、SIGTERM
注册 signal.Notify
    ▼
等待：
  ├─ 收到 SIGINT/SIGTERM
  └─ 可选 svc.Context.Done()
    ▼
p.Stop()
    ▼
返回 Stop 的 error
```

当前 `program` 没实现 `svc.Context` 接口，所以 `Run` 使用 `context.Background()`，正常情况下依靠信号结束等待。

### 为什么 `svc.Run` 会一直阻塞

阻塞点是：

```go
select {
case <-signalChan:
case <-ctx.Done():
}
```

`program.Start()` 返回不代表进程退出；它只是表示启动工作已完成。`svc.Run()` 随后停在 `select`，而各代理服务在 goroutine 中工作。

## `Environment`

```go
type Environment interface {
    IsWindowsService() bool
}
```

它让应用在 `Init` 时判断自己是否作为 Windows Service 运行。GOST 当前不使用参数：

```go
func (p *program) Init(env svc.Environment) error
```

保留参数是为了满足接口。

## `Context` 是可选接口

```go
type Context interface {
    Context() context.Context
}
```

`Run` 使用类型断言检查传入对象是否还实现这个接口。若实现，除了系统信号外，context 被取消也会触发 `Stop()`。

这是 Go 常见的“最小必需接口 + 可选扩展接口”设计。

## 与 `program.reload` 的信号分工

两个地方都监听系统信号，但用途不同：

| 信号 | 监听者 | 行为 |
| --- | --- | --- |
| `SIGINT` | `go-svc.Run` | 调用 `program.Stop()` 并退出 |
| `SIGTERM` | `go-svc.Run` | 调用 `program.Stop()` 并退出 |
| `SIGHUP` | `program.reload` | 重新加载配置，不退出 |

这是一种很清晰的 Unix 服务约定。

## Windows 分支应该怎样读

Go 文件名后缀参与条件编译：

- `svc_common.go` 带 `// +build !windows`，用于非 Windows；
- `svc_windows.go` 用于 Windows。

在 macOS/Linux 学习主请求链时，只需先掌握 common 分支。Windows 分支会判断当前是否作为 Windows Service 运行，并把 SCM 的 Start/Stop 控制转换为同一个 `Service` 接口调用。

## 错误与清理方面的细节

非 Windows `Run` 中：

- `Init` 返回错误：不调用 `Start`，也不调用 `Stop`；
- `Start` 返回错误：不调用 `Stop`；
- 等待结束后：返回 `Stop` 的错误。

这意味着 `Init` 或 `Start` 在返回错误前，应自行清理已经创建但尚未交给正常 Stop 流程的资源。检查 GOST 时，需要特别关注 `program.Start()` 中途失败是否可能遗留 listener。

## 它不负责什么

`go-svc` 不负责：

- 接收客户端网络连接；
- 解析 HTTP URL；
- DNS 解析；
- 建立目标网站连接；
- 搬运响应数据；
- 管理单个 GOST service 的连接 goroutine。

这些都在 `core` 契约和 `x` 实现中。阅读请求链时，`go-svc` 只覆盖最外层：

```text
启动进程 → 启动 GOST services → 等待停止信号 → 关闭 services
```
