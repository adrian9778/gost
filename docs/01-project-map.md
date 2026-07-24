# 01. 项目地图与阅读方法

## 目录结构

```text
gost/
├── cmd/gost/
│   ├── main.go       # 进程入口、命令行参数、多 worker 模式
│   ├── program.go    # 初始化、启动、停止、热重载
│   ├── register.go   # 用空白导入注册所有内置组件
│   └── version.go    # 版本变量
├── tests/e2e/        # 基于 Docker 的端到端测试
├── gost.yml          # 配置示例
├── go.mod            # 当前模块、Go 版本、依赖版本
├── Makefile          # 构建和发布命令
└── Dockerfile        # 容器镜像构建
```

`package main` 表示这个包可以编译成可执行文件。一个可执行程序必须有 `main` 包，并在其中提供 `func main()`。

## 为什么源码看起来很少

`go.mod` 的第一行：

```go
module github.com/go-gost/gost
```

它声明当前目录是一整个 Go module。业务所依赖的两个关键 module 是：

```go
github.com/go-gost/core
github.com/go-gost/x
```

Go module 与 package 不是同一个概念：

- **module** 是版本发布和依赖管理单位，由一个 `go.mod` 定义。
- **package** 是代码编译和复用单位，通常对应一个目录。
- 一个 module 通常包含很多 package。
- import 路径指向 package，而 `require` 通常指向 module。

例如 `github.com/go-gost/x/config/loader` 是 `x` module 中的一个 package。

## 三层职责

### core：定义契约

`core` 主要提供接口。比如当前仓库依赖的 `service.Service` 可以理解为服务都要遵守的契约：服务需要能够报告地址、开始服务和关闭资源。

接口让启动层不必知道服务究竟是 HTTP、SOCKS5 还是其他协议。只要具体类型实现了接口要求的方法，就可以统一保存和调用。

### x：实现能力

`x` 包含配置结构、解析器、加载器、注册表以及协议实现。可以把它看成 GOST 的主要“零件库”。

### gost：装配并启动

当前仓库把零件注册起来，读取用户输入，构建服务并管理进程生命周期。它类似应用程序的 `composition root`（组合根）：负责决定使用哪些实现，却尽量不实现协议细节。

## 从一条命令反推代码

以此命令为例：

```bash
gost -L http://:8080
```

可以按下面的路径阅读：

1. `main.go` 中 `flag.Var(&services, "L", ...)` 接收 `-L`。
2. `program.Init()` 把 `services` 传给 `parser.Init()`。
3. `program.Start()` 调用 `parser.Parse()`，生成配置对象。
4. `loader.Load(cfg)` 根据配置查找已注册的 listener、handler 等组件并构建服务。
5. 构建好的服务进入 `registry.ServiceRegistry()`。
6. `program.run()` 遍历注册表，并在 goroutine 中执行 `Serve()`。

第 3～5 步的具体实现不在当前 module，需要继续阅读 `github.com/go-gost/x`。

## 建议的两轮阅读法

### 第一轮：只看控制流

先忽略协议和配置字段，只标记：

- 谁调用谁；
- 哪些函数可能阻塞；
- 哪些对象需要关闭；
- 错误向哪里返回；
- 哪些工作在 goroutine 中运行。

### 第二轮：选择一条具体业务链

建议从最简单的 HTTP 代理开始，只追：

```text
命令行 -L
→ parser
→ loader
→ service registry
→ listener
→ handler
```

一次只追一个协议。`register.go` 中有上百个导入，如果同时阅读，很容易失去主线。

## 配置文件与命令行的关系

程序支持三类主要输入：

- `-C`：配置文件、URL 或内联 JSON，可重复。
- `-L`：直接在命令行定义服务，可重复。
- `-F`：直接在命令行定义转发链节点，可重复。

它们先被保存到包级变量，随后统一交给 `parser`。这说明 `main` 包只负责采集输入，并不负责理解所有配置语义。

## E2E 测试提供了什么线索

`tests/e2e` 不直接测试某个小函数，而是：

1. 编译真实的 `gost` 二进制文件；
2. 创建 Docker 网络；
3. 启动回显服务器和多个 GOST 容器；
4. 让网络流量经过代理链；
5. 检查最终响应。

这种测试适合网络程序，因为单元测试通过不代表监听端口、容器网络和协议组合真的能工作。对于新手，先读 `tests/e2e/main_test.go` 和 `utils.go`，可以看到 `os/exec`、`TestMain`、临时文件与资源清理的实际用法。

## 第一阶段学习清单

- [ ] 能说明 module 与 package 的区别。
- [ ] 能从 `main()` 追到 `program.Start()`。
- [ ] 能解释 `parser.Parse`、`loader.Load` 和 `registry` 各自的角色。
- [ ] 能解释为什么 `register.go` 只有 import 也有作用。
- [ ] 能指出哪些函数运行在新 goroutine 中。
- [ ] 能指出正常退出时哪些资源会被关闭。
