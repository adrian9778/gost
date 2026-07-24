# 12. 阶段 5：并发、关闭与可靠性

网络代理的难点往往不是解析一个请求，而是：

- 同时处理大量连接；
- 一个方向关闭时正确处理另一个方向；
- 超时、取消和热重载时不泄漏 goroutine；
- 局部故障不会让整个服务退出。

本篇从 goroutine 所有权和资源生命周期阅读 GOST。

## 1. goroutine 层级

```text
main goroutine
└─ svc.Run
   ├─ 每个业务 Service.Serve goroutine
   │  ├─ service stats observer goroutine（可选）
   │  └─ 每个 client connection 一个 Handler goroutine
   │     ├─ HTTP transport 内部 goroutine
   │     ├─ Pipe client→target goroutine
   │     └─ Pipe target→client goroutine
   ├─ API Serve goroutine
   ├─ Metrics Serve goroutine
   ├─ Profiling Serve goroutine
   └─ reload goroutine
```

若启用 WebSocket keepalive、observer、动态 loader 等功能，还会增加后台 goroutine。

读并发代码时始终问四个问题：

1. 谁创建它？
2. 谁通知它退出？
3. 谁等待它结束？
4. 它阻塞的 I/O 如何被唤醒？

## 2. Service 的 accept 并发模型

`x/service.defaultService.Serve` 在一个 goroutine 中串行 Accept：

```text
Accept conn1 → 启动 Handler goroutine 1
Accept conn2 → 启动 Handler goroutine 2
Accept conn3 → 启动 Handler goroutine 3
```

Accept 本身串行没有问题，因为每次只负责取出一个已建立连接；耗时的协议处理放到独立 goroutine。

Service 用 `sync.WaitGroup` 记录 Handler：

```go
wg.Add(1)
go func() {
    defer wg.Done()
    handler.Handle(ctx, conn)
}()
```

`Serve()` 返回时执行 `wg.Wait()`，意图是不在已有 Handler 尚未结束时完全退出。

## 3. context 中传递什么

每个连接建立 session context，包含：

- SID；
- labels；
- source/destination address；
- client IP hash；
- recorder/log 信息；
- client identity。

context 有两种用途：

1. 取消和 deadline；
2. request-scoped value。

Go 社区通常建议 context value 只放跨 API 边界的请求级信息，不放普通可选参数。GOST 的 SID、地址、日志上下文符合请求级特征。

## 4. HTTP Handler 的连接所有权

`httpHandler.Handle` 开头：

```go
defer conn.Close()
```

这明确表示 Handler 接管入站连接，Handle 返回时关闭它。

普通 HTTP：

- `http.Transport` 管理出站连接池；
- 每次 `resp.Body.Close()` 允许连接被回收复用；
- client keep-alive 循环继续读取下一 request。

CONNECT：

- `handleConnect` 建立出站 `cc`；
- 默认由 defer 关闭；
- Pipe 在 Handle 返回前一直运行；
- Handle 最终关闭 client conn。

接口注释也约定 Handler 返回时连接被关闭。

## 5. `xnet.Pipe` 如何双向复制

文件：`x/internal/net/pipe.go`

```text
goroutine A: Read(client) → Write(target)
goroutine B: Read(target) → Write(client)
```

为什么不能在一个 goroutine 中轮流读两边？

任何一次 `Read` 都可能长期阻塞。如果先阻塞读取客户端，就无法及时转发目标网站已经返回的数据。两个方向必须独立等待。

### 错误 channel

```go
errCh := make(chan error, 2)
```

容量为 2，两个 pipeHalf 即使 Pipe 主协程暂时不接收，也都能报告结果并结束，避免发送错误本身阻塞。

Pipe 等待两个方向完成，保留第一个非 nil error。EOF 被 pipeHalf 转成 nil，代表正常结束。

### Buffer Pool

每个方向：

```go
buf := bufpool.Get(bufferSize / 2)
defer bufpool.Put(buf)
```

复用缓冲区能减少高连接量下的分配和 GC 压力。借出的 buffer 不能在 Put 后继续保存或使用。

## 6. TCP half-close

TCP 是全双工的。客户端不再发送数据，并不代表它不能继续接收响应。

pipeHalf 完成一个方向时尝试：

```text
src.CloseRead()
dst.CloseWrite()
```

`CloseWrite` 发送 FIN，只关闭该方向：

```text
client → target 已结束
client ← target 仍可继续接收
```

若 wrapper 不支持 half-close，就回退到完整 `Close()`。成功 half-close 后设置 10 秒 read deadline，避免另一半永久停留。

wrapper 是否正确透传 `CloseRead/CloseWrite` 会影响行为。只实现 `net.Conn` 的简单 wrapper 可能让 half-close 退化为全关闭。

## 7. context 取消怎样唤醒阻塞 Read

仅仅这样检查还不够：

```go
select {
case <-ctx.Done():
default:
}
return conn.Read(buf)
```

因为取消可能发生在进入阻塞 `Read` 之后。

`Pipe` 的主循环同时监听 `ctx.Done()`，取消后执行：

```text
forceClose(clientConn, targetConn)
```

关闭 socket 会让两个阻塞 Read 返回错误，pipeHalf 才能退出。可靠取消通常需要：

```text
context 信号 + 关闭底层 I/O
```

## 8. Deadline 与 idle timeout

`WithReadTimeout(d)` 让每次 Read 前执行：

```go
SetReadDeadline(now + d)
```

这是空闲超时，而不是整条连接的总寿命：

- 持续有数据：每次 Read 都刷新 deadline；
- 长时间无数据：Read 超时，Pipe 返回。

`d == 0` 时禁用，由 context、TCP keepalive 或对端关闭发现死连接。

三种机制不同：

| 机制 | 解决问题 |
| --- | --- |
| Context timeout | 一次逻辑操作的总时间 |
| Socket deadline | 阻塞 Read/Write/Handshake 的截止时间 |
| TCP keepalive | 内核探测长时间失活或断网的连接 |

## 9. Accept 错误退避

Service 对实现 `net.Error` 且 `Temporary()` 为 true 的错误重试：

```text
1s → 2s → 4s → 5s → 5s...
```

作用：

- 防止临时资源耗尽时 busy loop；
- 给系统恢复时间；
- 状态暂时切为 Failed，随后恢复 Ready。

非临时错误使 Serve 返回。listener 被 Close 时通常得到 `net.ErrClosed`，不作为异常日志。

## 10. Router 超时与重试

每一次尝试创建：

```go
context.WithTimeout(parent, routerTimeout)
```

并用 `defer cancel()` 回收 timer。

总最坏时间不是简单等于 timeout，而接近：

```text
(Retries + 1) × 每次 timeout
```

再加解析和调度开销。配置重试时应考虑客户端本身的超时，否则客户端早已断开，Router 仍可能继续尝试，除非 parent context 同步取消。

## 11. 关闭顺序

最外层：

```text
SIGINT/SIGTERM
→ go-svc 调用 program.Stop
→ 取消 reload
→ 关闭每个业务 Service
→ 关闭 API/Metrics/Profiling
```

`defaultService.Close`：

```text
关闭 Handler（若实现 io.Closer）
→ 关闭 Observer（若实现 io.Closer）
→ 关闭 Listener
```

关闭 Listener 会唤醒 Accept，使 Serve 返回。

## 12. 源码中值得警惕的关闭边界

### Service 的 defer 顺序

`Serve` 先注册：

```go
defer cancel()
```

后注册：

```go
defer wg.Wait()
```

defer 按后进先出执行，因此函数返回时实际顺序是：

```text
wg.Wait()
→ cancel()
```

这表示 Serve 会先等待现有 Handler 完成，之后才取消 `gctx`。而 `Service.Close()` 只关闭 Handler 自身后台任务、Observer 和 Listener，没有保存并逐个关闭当前 client conn。

由当前代码可推断：

- 旧 Listener 会停止接收新连接；
- 已存在的长连接可能继续运行；
- Serve 可能等待长连接自然结束；
- gctx 的 cancel 不能提前帮助这些 Handler 退出。

`program` 启动 `Serve` 时也没有保存其 goroutine 的 WaitGroup，因此 `program.Stop` 不会等待所有 Service.Serve goroutine 完成。

这不是在此直接判定为 bug，而是需要在“优雅关闭”和热重载测试中重点验证的行为边界。

### 多 worker 的取消时机

`main.go` 多 worker 父进程：

```text
启动全部 worker
→ wg.Wait()
→ cancel()
```

因此一个 worker 返回时不会立即执行 cancel；父进程要等所有 worker 返回后才取消 context。源码注释中的“one worker exits early”意图与当前控制流需要进一步核对。

### `signal.Notify` 清理

`program.reload` 调用 `signal.Notify(c, SIGHUP)`，返回前没有 `signal.Stop(c)`。当前程序通常只启动一次 reload goroutine，影响有限；若生命周期允许反复 Start/Stop，则应测试订阅是否积累。

## 13. 热重载的并发过程

```text
reload goroutine 收到 SIGHUP
→ parser.Parse
→ loader.Load
   → 关闭旧 Service/Listener
   → 构造新 Service/Listener
→ program.run 启动新 Service.Serve
```

旧连接与新连接可能短暂并存：

- 新连接进入新 Service；
- 旧 Service 不再 Accept；
- 已存在旧 Handler 可能继续处理。

这可能是期望的连接排空，也可能导致旧配置长期残留，取决于业务要求。可靠性测试应专门覆盖长连接跨 reload 的语义。

## 14. `Transport` 与 `Pipe` 的错误处理差异

`x/internal/net` 同时存在较旧的 `Transport` 和功能更完整的 `Pipe`。

`Transport` 启动两个复制 goroutine 后只接收一次：

```go
if err := <-errc; err != nil && err != io.EOF {
    return err
}
return nil
```

如果正常结束的方向先写入 channel，它立即返回 nil，另一个方向稍后发生的错误不会被调用方看到。

本阶段实际运行当前依赖测试时：

```text
TestTransport_ReadError
expected error, got nil
```

这与上述调度竞争一致。结果可能依赖哪个复制方向先返回。

`Pipe` 则等待两个 pipeHalf 都结束，并保存第一个非 nil error，错误语义更完整。阅读调用点时要确认使用的是 `Transport` 还是 `Pipe`，不能因两者都是双向复制就认为行为完全相同。

这是源码学习中很有价值的例子：

- 并发单测失败不一定是测试环境问题；
- channel 容量安全不等于结果聚合逻辑正确；
- “等待第一个完成”与“等待全部完成并保留第一个错误”是不同需求。

## 15. 错误传播层次

```text
启动期错误
parser/loader/run error
→ program.Start error
→ svc.Run error
→ main Fatal

单连接错误
Handler.Handle error
→ Service goroutine 记录日志/指标
→ 不影响 Accept 循环

目标连接错误
Router/Route/Connector error
→ Handler 转为协议错误响应（HTTP 常为 503）
→ 当前请求结束

Accept 致命错误
→ Service.Serve 返回
→ program.run 的 goroutine 当前未消费其 error
```

最后一点值得关注：`program.run` 对业务 Service 使用 `go svc.Serve()`，没有记录返回错误；API/Metrics 则明确检查并记录。

## 阶段完成标准

- [ ] 能画出 goroutine 层级和所有者。
- [ ] 能解释每连接一个 Handler goroutine。
- [ ] 能解释 Pipe 为什么需要两个 goroutine。
- [ ] 能说明 half-close 与 full close 的区别。
- [ ] 能说明 context 取消为何还要关闭 socket。
- [ ] 能区分 context timeout、deadline、keepalive。
- [ ] 能解释 Accept 的指数退避。
- [ ] 能指出热重载中新旧长连接的并存边界。
- [ ] 能从 defer 注册顺序推导真实关闭顺序。
