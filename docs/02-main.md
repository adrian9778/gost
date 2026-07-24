# 02. 程序入口：`main.go`

`main.go` 做四件事：

1. 定义可重复使用的字符串参数类型；
2. 在第一个 `init()` 中实现多 worker 模式；
3. 在第二个 `init()` 中解析命令行参数；
4. 在 `main()` 中设置日志并进入服务生命周期。

## `stringList`：让一个 flag 出现多次

```go
type stringList []string
```

这是一个以 `[]string` 为底层类型的自定义类型。定义新类型后，可以给它添加方法。

标准库 `flag` 的 `Value` 接口要求实现：

```go
type Value interface {
    String() string
    Set(string) error
}
```

本项目用指针接收者实现：

```go
func (l *stringList) Set(value string) error {
    *l = append(*l, value)
    return nil
}
```

逐步理解这行代码：

- `l` 是指向 `stringList` 的指针；
- `*l` 取出指针指向的切片值；
- `append(*l, value)` 返回追加后的新切片；
- 再赋值给 `*l`，才能更新原变量的切片头。

必须使用指针接收者，因为 `Set` 要修改原值。于是：

```bash
gost -L http://:8080 -L socks5://:1080
```

会让 `services` 保存两个字符串。

## 包级变量

`cfgFiles`、`services`、`nodes` 等变量定义在函数外，所以同一个 `main` 包里的 `program.go` 也能直接访问它们。

这很方便，但也形成全局状态。阅读时要记住：

```text
main.go 的 init() 写入变量
       ↓
program.go 的 Init() 读取变量
```

## Go 程序的初始化顺序

简化后的顺序是：

1. 初始化被导入的依赖包；
2. 初始化当前包的包级变量；
3. 按编译器确定的文件顺序执行当前包的各个 `init()`；
4. 最后执行 `main()`。

不要把“同一个包内文件的字母顺序”当作语言规范保证。当前代码的两个 `init()` 位于同一文件，并按源码顺序出现：先处理 worker 分隔符，再解析 flag。

`init()` 不能由普通代码显式调用，也不能带参数和返回值。

## 第一个 `init()`：多 worker 模式

程序把参数连接起来并寻找两侧带空格的 `--`：

```go
args := strings.Join(os.Args[1:], "  ")
if strings.Contains(args, " -- ") {
    // ...
}
```

例如：

```bash
gost -L http://:8080 -- -L socks5://:1080
```

父进程把它分成两组参数，每组启动一个相同的 `gost` 子进程。子进程分别只收到属于自己的那组参数。

这里使用两个空格连接原始参数，是一种自定义编码方式：组内参数用两个空格重新拆分，组间用 ` -- ` 拆分。阅读此类代码时应意识到，它不是通用 shell 解析器，核心前提是输入由 `os.Args` 提供，shell 已经完成第一轮参数切分。

### `WaitGroup`

```go
var wg sync.WaitGroup
wg.Go(func() {
    worker(...)
})
wg.Wait()
```

`WaitGroup` 用来等待一组并发任务结束：

- `wg.Go` 启动任务并把它计入等待组；
- `wg.Wait` 阻塞，直到所有任务返回。

如果阅读旧版 Go 项目，更常见的是 `Add(1)`、`go func()` 和 `defer Done()` 的组合。这里使用的是较新标准库提供的 `WaitGroup.Go`。

### 循环变量与闭包

goroutine 的函数闭包使用了循环变量 `wid` 和 `wargs`：

```go
for wid, wargs := range wargsList {
    wg.Go(func() {
        worker(wid, ..., ctx, &ret)
    })
}
```

现代 Go 的 `range` 每轮会产生对应迭代值，因此闭包能取得该轮的值。阅读旧代码或旧版 Go 教程时，可能会看到在循环体内重新声明变量以规避经典的闭包捕获问题。

### `context` 的作用

```go
ctx, cancel := context.WithCancel(context.Background())
```

`context` 用于传递取消信号。它不是强制终止任意函数的魔法；被调用方必须主动观察它。这里 `exec.CommandContext` 会把取消与子进程生命周期关联。

当前流程在 `wg.Wait()` 之后才执行 `cancel()`，所以正常路径是等待所有 worker 完成后取消 context。注释所说的“一个 worker 提前退出时取消其他 worker”需要结合实际控制流审视：仅从当前代码看，`cancel` 并没有在单个 worker 返回时立即调用。

这是很好的源码阅读练习：注释表达设计意图，但最终行为必须以控制流为准。

### `atomic.Int32`

多个 worker 可能并发写退出码：

```go
ret.Store(int32(cmd.ProcessState.ExitCode()))
```

普通变量的并发读写可能产生数据竞争。`atomic.Int32` 提供原子 `Store` 和 `Load`。不过它只保证单次读写安全，不定义“哪个 worker 的退出码应该获胜”；最后一次写入会成为父进程退出码。

## `worker()`：启动子进程

```go
cmd := exec.CommandContext(ctx, os.Args[0], args...)
```

- `os.Args[0]` 是当前可执行文件路径；
- `args...` 使用展开语法，把 `[]string` 作为多个可变参数传入；
- `CommandContext` 创建命令对象，此时还没有启动进程；
- `cmd.Run()` 才启动并等待子进程结束。

标准输出和错误输出被连接到父进程：

```go
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
```

每个子进程还会获得 `_GOST_ID`：

```go
cmd.Env = append(os.Environ(), fmt.Sprintf("_GOST_ID=%d", id))
```

`os.Environ()` 保留父进程已有环境变量，`append` 增加 worker ID。

## 第二个 `init()`：flag 解析

这部分使用 Go 标准库 `flag`：

```go
flag.BoolVar(&debug, "D", false, "debug mode")
flag.DurationVar(&reload, "R", 0, "auto reload period")
flag.Parse()
```

函数接收变量地址，解析后直接修改对应变量。`DurationVar` 可解析 `30s`、`1m` 这样的字符串并生成 `time.Duration`。

`-V` 和 `-O` 都可能让程序提前退出：

- `-V` 在 `init()` 中打印版本并调用 `os.Exit(0)`；
- `-O` 在 `program.Start()` 中输出合并后的配置并退出。

`os.Exit` 会立即终止进程，已注册的 `defer` 不会执行。因此一般只在明确不需要清理，或清理已完成时使用。

## 空白导入 `net/http/pprof`

```go
_ "net/http/pprof"
```

下划线表示导入包但不直接引用其导出标识符。这个包的 `init()` 会向默认 HTTP mux 注册性能分析路由。之后 `program.go` 启动 HTTP profiling server 时，这些路由已经存在。

## `main()`：把控制权交给生命周期框架

```go
log := xlogger.NewLogger()
logger.SetDefault(log)

p := &program{}
if err := svc.Run(p); err != nil {
    logger.Default().Fatal(err)
}
```

这里体现“面向接口编程”：

- `program` 实现了框架期望的 `Init`、`Start`、`Stop` 方法；
- `svc.Run` 负责操作系统服务和信号等通用生命周期；
- 当前项目只实现自己的初始化、启动和清理逻辑。

`&program{}` 创建一个字段为零值的 `program` 并取得指针。Go 的零值设计让 `nil` service、`nil` cancel 函数在后续通过判空安全处理。

## 动手实验

```bash
# 查看版本分支，不会进入 main()
go run ./cmd/gost -V

# 查看 flag 自动生成的帮助
go run ./cmd/gost -h

# 观察可重复参数如何合并成配置
go run ./cmd/gost -L http://:8080 -L socks5://:1080 -O yaml
```

思考题：

1. 为什么 `Set` 使用 `*stringList` 而 `String` 也选择指针接收者？
2. 如果 `cmd.Stdout` 不赋值，会在哪里看到子进程输出？
3. 多 worker 中一个子进程永久运行、另一个退出时，父进程会怎样？
