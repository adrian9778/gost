# GOST 源码阅读笔记

这组笔记面向刚开始读 Go 项目源码的读者。目标不只是说明“每个函数做什么”，还会解释代码中出现的 Go 语法、设计习惯、并发模型，以及继续追踪源码的方法。

> 当前笔记基于本仓库 `go.mod` 所声明的 GOST 版本和当前工作区源码。代码升级后，行号可能变化，应以函数名和调用关系为准。

## 先认识这个仓库

当前仓库是 GOST 可执行程序的**组装层和启动层**，不是所有实现的集合：

| 模块 | 作用 | 在哪里 |
| --- | --- | --- |
| `github.com/go-gost/gost` | 命令行入口、生命周期、组件装配 | 当前仓库 |
| `github.com/go-gost/core` | 核心接口，例如 `service.Service`、日志和认证接口 | Go 依赖模块 |
| `github.com/go-gost/x` | 配置解析、注册表，以及各种协议的具体实现 | Go 依赖模块 |

因此，当前仓库只有少量 Go 文件并不异常。它采用了常见的“主程序很薄、能力由依赖包提供”的组织方式。

## 推荐阅读顺序

1. [项目地图与阅读方法](01-project-map.md)
2. [程序入口：main.go](02-main.md)
3. [生命周期与热重载：program.go](03-program.md)
4. [组件注册与依赖关系：register.go](04-registration.md)
5. [core：核心接口与抽象](05-core.md)
6. [x：主要实现与组装过程](06-x.md)
7. [go-svc：进程生命周期框架](07-go-svc.md)
8. [专题：从目标 URL 到响应返回](08-http-proxy-request-flow.md)
9. [阶段 3：配置如何变成运行对象](09-config-to-runtime.md)
10. [阶段 4：多级代理链的执行过程](10-multi-hop-chain.md)
11. [阶段 4：HTTP、SOCKS5、TLS 与 WebSocket](11-protocol-stack.md)
12. [阶段 5：并发、关闭与可靠性](12-concurrency-reliability.md)
13. [阶段 5：调试、测试与源码实验](13-debugging-testing.md)
14. [专题补充：Router 的功能与完整调用链](14-router.md)

第一轮建议只回答三个问题：

1. 操作系统启动 `gost` 后，代码按什么顺序执行？
2. 命令行和配置文件如何变成正在运行的服务？
3. HTTP、SOCKS5、TLS 等实现为什么没有在当前仓库中直接被调用？

第二轮重点阅读第 5～8 篇。其中第 8 篇是主线专题，先读它可以建立整体认识，再回头用第 5～7 篇补齐各层职责。

完整路线分为五个阶段：

| 阶段 | 主题 | 文档 |
| --- | --- | --- |
| 1 | 启动入口与组件注册 | 01～04 |
| 2 | 核心依赖与 HTTP 请求链 | 05～08 |
| 3 | 配置到运行对象 | 09 |
| 4 | 代理链与具体协议 | 10～11 |
| 5 | 并发、可靠性、调试与测试 | 12～13 |

## 一张总调用图

```text
操作系统启动进程
    │
    ├─ 加载所有 import 的包
    │    └─ register.go 的空白导入触发各组件 init()，写入 registry
    │
    ├─ main.go 第一个 init()
    │    └─ 检测 ` -- `，必要时派生多个 worker 子进程
    │
    ├─ main.go 第二个 init()
    │    └─ 注册并解析命令行参数
    │
    └─ main()
         ├─ 设置全局日志器
         └─ svc.Run(&program{})
              ├─ program.Init()
              │    └─ 保存配置解析参数
              ├─ program.Start()
              │    ├─ parser.Parse()：得到 config.Config
              │    ├─ loader.Load()：构建服务并写入 registry
              │    ├─ program.run()：启动所有服务
              │    └─ 启动 reload goroutine
              └─ program.Stop()
                   └─ 取消 reload，并关闭所有服务
```

## 阅读时可使用的命令

```bash
# 查看当前模块及 Go 版本
go env GOMOD GOVERSION

# 查看直接依赖
go list -m all

# 找到某个依赖在本机模块缓存中的目录
go list -m -f '{{.Dir}}' github.com/go-gost/x
go list -m -f '{{.Dir}}' github.com/go-gost/core

# 查看当前仓库的包
go list ./...

# 编译入口包（不运行）
go build ./cmd/gost
```

## 做笔记的统一模板

以后增加一篇源码笔记时，可以按以下问题组织：

```markdown
# 文件或包名

## 它解决什么问题
## 它在整个调用链中的位置
## 核心类型和接口
## 主流程（按执行顺序）
## Go 语法知识
## 并发、资源释放和错误处理
## 我还没有弄懂的问题
## 可以动手验证的实验
```

不要一开始逐行翻译代码。先确定输入、输出、依赖和调用者，再阅读函数内部，效率会高很多。
