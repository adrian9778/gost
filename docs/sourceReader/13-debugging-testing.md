# 13. 阶段 5：调试、测试与源码实验

本篇给出一条适合 Go 新手的验证路线：

```text
静态阅读
→ 配置展开
→ 本地最小实验
→ 日志
→ 单元测试
→ race detector
→ Delve
→ E2E
→ pprof
```

不要一开始就运行复杂 Docker 多级代理。先用能观察的最小系统验证一个猜想。

## 1. 建立可重复的本地实验

### 终端 1：目标 HTTP 服务

```bash
python3 -m http.server 9000 --bind 127.0.0.1
```

### 终端 2：GOST HTTP 代理

```bash
go run ./cmd/gost -DD -L http://127.0.0.1:8080
```

### 终端 3：客户端

```bash
curl -v \
  -x http://127.0.0.1:8080 \
  http://127.0.0.1:9000/
```

这三个角色分别对应：

```text
curl clientConn → GOST → targetConn Python server
```

本地实验不依赖 DNS、互联网和第三方网站，更适合定位控制流。

## 2. 先用 `-O` 验证配置

遇到“不知道短格式生成了什么”时：

```bash
go run ./cmd/gost \
  -L 'http://:8080' \
  -F 'socks5+tls://proxy:1080' \
  -O yaml
```

`-O` 在 loader 之前退出：

- 能验证 parser/cmd；
- 不会绑定端口；
- 不会真的连接代理节点；
- 不能验证 registry 构造和协议运行。

这是静态配置问题与运行时网络问题的分界工具。

## 3. 日志级别

```text
-D  → debug
-DD → trace
```

Debug 适合观察：

- service 监听地址；
- Router dial；
- route 路径；
- Connector connect；
- reload。

Trace 可能打印 HTTP request/response Header。注意其中可能包含认证信息、Cookie 或业务数据，不应在生产环境随意长期启用或公开日志。

使用 SID 关联同一客户端连接的多层日志：

```text
Service SID
→ Handler log
→ Router log
→ Connector log
```

## 4. 用系统工具观察连接

macOS/Linux 可根据环境使用：

```bash
lsof -nP -iTCP:8080
lsof -nP -iTCP:9000
```

你应看到：

- GOST LISTEN `127.0.0.1:8080`；
- curl 到 GOST 的 ESTABLISHED；
- GOST 到 Python server 的 ESTABLISHED。

这能验证“代理至少有入站、出站两条连接”。

抓取明文测试流量可使用 Wireshark/tcpdump。涉及管理员权限的命令应由学习者在明确网卡和过滤条件后自行执行，避免无边界抓取其他流量。

## 5. 单元测试的层次

当前入口仓库主要是 E2E，核心单元测试位于依赖 module。

可以从当前 module 运行指定依赖包：

```bash
go test github.com/go-gost/x/handler/http -v
go test github.com/go-gost/x/chain -v
go test github.com/go-gost/x/service -v
go test github.com/go-gost/x/internal/net -v
```

重点测试映射：

| 想验证的行为 | 包/测试 |
| --- | --- |
| HTTP RoundTrip 与 Header 清理 | `x/handler/http` |
| Route 和节点选择 | `x/chain` |
| Accept/Handler 生命周期 | `x/service` |
| Pipe、取消和 half-close | `x/internal/net` |
| 配置来源和合并 | `x/config/parsing/parser` |
| loader 热重载注册 | `x/config/loader` |

## 6. `net.Pipe` 适合协议单元测试

标准库 `net.Pipe()` 返回一对内存中的 `net.Conn`：

```text
connA.Write → connB.Read
connB.Write → connA.Read
```

它无需真实端口，适合测试：

- HTTP CONNECT request/reply；
- SOCKS5 handshake；
- 双向复制；
- deadline；
- close。

但内存 Pipe 与真实 TCP 在地址、buffer、half-close 和内核行为上可能不同，所以不能替代集成测试。

## 7. Race detector

并发改动后：

```bash
go test -race github.com/go-gost/x/chain
go test -race github.com/go-gost/x/service
go test -race github.com/go-gost/x/internal/net
```

Race detector 查找无同步的共享内存访问，不负责发现：

- goroutine 泄漏；
- deadlock 的所有形式；
- 协议逻辑错误；
- 网络超时配置错误。

它也会明显降低运行速度，因此测试 timeout 要适当放宽。

## 8. Delve 调试主请求链

安装并确认：

```bash
dlv version
```

启动：

```bash
dlv debug ./cmd/gost -- \
  -L http://127.0.0.1:8080
```

建议断点顺序：

```text
github.com/go-gost/x/service.(*defaultService).Serve
github.com/go-gost/x/handler/http.(*httpHandler).Handle
github.com/go-gost/x/handler/http.(*httpHandler).handleRequest
github.com/go-gost/x/handler/http.(*httpHandler).proxyRoundTrip
github.com/go-gost/x/handler/http.(*httpHandler).dial
github.com/go-gost/x/chain.(*Router).Dial
github.com/go-gost/x/chain.(*defaultRoute).Dial
```

Delve 对未导出方法的符号写法可能随版本和编译优化略有差异，可先：

```text
funcs httpHandler
funcs defaultService
```

再复制实际符号。

观察变量：

```text
req.Method
req.URL
req.Host
addr
network
resp.StatusCode
conn.LocalAddr()
conn.RemoteAddr()
```

依赖源码位于 module cache，调试器仍能定位；若要修改依赖，使用本地 clone + replace。

## 9. 修改依赖进行实验

不要直接修改 `$GOMODCACHE`。建议：

```bash
git clone https://github.com/go-gost/x /path/to/local/x
go mod edit -replace github.com/go-gost/x=/path/to/local/x
go mod tidy
```

实验结束后：

```bash
go mod edit -dropreplace github.com/go-gost/x
go mod tidy
```

注意 `go mod tidy` 会修改 `go.mod/go.sum`。在学习分支中进行，并检查 diff 后再决定是否提交。

## 10. 当前 E2E 架构

`tests/e2e/TestMain`：

```text
解析 test flags
→ 若未指定 -gost-bin，则编译 ../../cmd/gost
→ 创建共享 Docker network
→ m.Run()
→ 删除 network 和临时 binary
```

每个 Suite：

```text
SetupSuite：启动 TCP/UDP echo container
→ 启动 GOST server/client containers
→ 发送真实网络流量
→ 验证 hello-gost
→ 失败时 DumpLogs
→ defer Terminate containers
→ TearDownSuite
```

这是真正的黑盒测试：使用编译后的二进制、配置文件、容器网络和 curl。

## 11. 如何运行 E2E

前提：

- Docker daemon 正常；
- 当前用户能访问 Docker；
- 首次构建镜像可能需要网络和较长时间。

```bash
go test ./tests/e2e/ -v -timeout 10m
```

只运行一组：

```bash
go test ./tests/e2e/ \
  -v \
  -run TestParallelSelectorSuite \
  -timeout 5m
```

复用已编译 binary：

```bash
CGO_ENABLED=0 go build -o /tmp/gost-study ./cmd/gost

go test ./tests/e2e/ \
  -v \
  -gost-bin /tmp/gost-study \
  -timeout 10m
```

测试自定义 flag 必须放在包参数之后；`go test` 会把它传给测试 binary。

## 12. 阅读 E2E 的资源清理

常见模式：

```go
container, err := ...
Require().NoError(err)
defer container.Terminate(ctx)
```

只有创建成功后才注册 defer。

临时配置：

```go
rendered, err := RenderConfig(...)
defer os.Remove(rendered)
```

`TestMain` 必须显式 `os.Exit(code)`，因此它不能依赖自身 defer 做最终清理；源码先 Remove network/binary，再 Exit。

## 13. 为 HTTP 专题补一条 E2E 的建议

现有测试通过 curl 检查响应正文，适合验证端到端结果。若新增专门 HTTP 请求链测试，至少覆盖：

1. 普通 GET 经 HTTP proxy 返回 body；
2. POST body 原样到达目标；
3. 目标返回 404/500 时状态码透传；
4. keep-alive 上连续两个请求；
5. CONNECT 后 TLS 请求成功；
6. 目标不可达时返回 503；
7. Proxy-Authorization 不泄露到目标；
8. reload 后新连接使用新配置；
9. reload 时已有长连接的预期语义；
10. 取消/关闭后 goroutine 数量最终回落。

## 14. pprof

`main.go` 空白导入 `net/http/pprof`，配置 profiling 后启动 HTTP server。

常见观察：

```text
/debug/pprof/goroutine
/debug/pprof/heap
/debug/pprof/profile
```

示例配置：

```yaml
profiling:
  addr: 127.0.0.1:6060
```

只绑定 loopback 更适合本地学习。pprof 可能暴露函数、goroutine、内存和运行状态，不应无认证直接暴露公网。

排查 goroutine 泄漏时：

1. 记录空闲基线；
2. 建立并关闭一批代理连接；
3. 等待合理清理时间；
4. 再取 goroutine profile；
5. 比较是否大量停在 `Accept`、`Read`、`Pipe`、ticker 或 observer。

## 15. 故障定位决策表

| 现象 | 第一检查点 |
| --- | --- |
| 启动即报 unknown handler/listener | registry 空白导入与配置 type |
| 端口无法绑定 | 旧进程、旧 Service、listener.Init |
| 能连接代理但目标失败 | Handler target、Router route、Connector |
| HTTP 返回 503 | RoundTrip/dial 错误日志 |
| HTTPS CONNECT 成功但随后卡住 | Pipe、TLS handshake、目标 443 |
| 只有域名失败、IP 成功 | Resolver/hosts/本地与远程 DNS |
| 一段时间后连接断开 | 各层 deadline、idle timeout、keepalive |
| reload 后行为混合 | 新旧 Service/长连接并存 |
| CPU 高 | Accept error busy loop、复制循环、pprof CPU |
| 内存持续增长 | connection/Body 未 Close、buffer、goroutine profile |

## 16. 五阶段最终验收项目

选择下面命令：

```bash
gost \
  -L http://127.0.0.1:8080 \
  -F socks5+tls://proxy.example:1080
```

不用看笔记，独立完成：

1. 用 `-O yaml` 画出 Config；
2. 说明 loader 注册顺序；
3. 画出 Service/Listener/Handler/Router/Chain；
4. 分解 Node 的 Dialer 与 Connector；
5. 跟踪普通 HTTP GET；
6. 跟踪 HTTPS CONNECT；
7. 说明 response 回客户端的两种方式；
8. 标出所有主要 goroutine；
9. 标出每条连接的关闭者；
10. 设计一个成功测试和三个故障测试。

能完整讲清这十项，就已经不只是“看懂几行 Go”，而是建立了阅读真实网络服务源码的系统方法。
