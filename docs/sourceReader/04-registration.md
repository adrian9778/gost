# 04. 组件注册与依赖关系：`register.go`

`register.go` 几乎全是这种 import：

```go
_ "github.com/go-gost/x/listener/tcp"
_ "github.com/go-gost/x/handler/http"
```

它没有普通函数，却是可执行程序拥有各种协议能力的关键。

## 空白导入有什么用

Go 要求普通 import 必须被使用。将包名写成 `_` 表示：

> 我不在当前文件直接调用这个包的导出成员，但我需要执行这个包的初始化副作用。

被导入包通常会在自己的 `init()` 中完成类似动作：

```go
func init() {
    registry.ListenerRegistry().Register("tcp", NewListener)
}
```

上面是帮助理解的示意代码，不代表依赖中的精确函数签名。

## 注册表模式

可以把 registry 想成从名字到构造器的 map：

```text
"tcp"    → TCP listener 构造器
"tls"    → TLS listener 构造器
"http"   → HTTP handler 构造器
"socks5" → SOCKS5 handler 构造器
```

配置解析得到字符串名字后，loader 不需要写一个巨大的 `switch`，而是从 registry 查找相应构造器。

完整链条是：

```text
register.go 空白导入
        ↓
具体组件包的 init()
        ↓
构造器注册到 registry
        ↓
parser 生成配置
        ↓
loader 按配置名称查 registry
        ↓
创建 listener / handler / dialer / connector
        ↓
组装为 service 并放入 ServiceRegistry
        ↓
program.run 调用 Serve()
```

## 四类核心组件

以下解释是阅读入口，不是所有协议的完整定义。

### Listener

Listener 负责“怎么接收进入程序的连接或数据”。例子包括 TCP、TLS、WebSocket、QUIC、UDP、Unix socket。

可以类比餐厅的入口：客人通过哪扇门、以什么方式进入。

### Handler

Handler 负责“收到连接后怎么理解并处理”。HTTP 代理、SOCKS、端口转发、DNS、隧道等属于 handler。

它更像接待流程：识别请求、认证、决定目标和转发方式。

### Dialer

Dialer 负责转发链上“怎样建立到下一跳的底层连接”。例如用 TCP、TLS、WebSocket、QUIC 或 SSH 连接下一节点。

### Connector

Connector 负责在已经建立的传输之上“怎样与代理节点协商并连接最终目标”。例如 SOCKS5 connector 会进行 SOCKS5 协议握手。

初学时最容易混淆 dialer 和 connector。可先记：

```text
Dialer：通道怎么建
Connector：通道建好后，怎么通过代理协议请求目标
```

之后应以 `core` 中接口定义和具体实现校正这个心智模型。

## 为什么采用初始化副作用

优点：

- 主程序只需导入某组件，就能把它加入最终二进制；
- loader 面向统一 registry，不依赖所有具体实现；
- 新协议通常只需实现接口、注册构造器、在组装层增加导入。

代价：

- 依赖关系不够显式，搜索普通函数调用找不到注册过程；
- 初始化顺序和全局状态需要谨慎管理；
- 漏掉空白导入时，代码可能编译成功，但运行时提示组件未注册；
- 全量导入会增加编译依赖和最终二进制所含能力。

## 如何继续追一个组件

以 TCP listener 为例：

```bash
# 获取 x module 源码目录
go list -m -f '{{.Dir}}' github.com/go-gost/x

# 假设上一命令输出 $X_DIR，寻找注册入口
rg 'func init|Register' "$X_DIR/listener/tcp"
```

然后按此顺序阅读：

1. 包内 `init()`：以什么名字注册；
2. registry 接收的构造器类型；
3. `New...`：有哪些 option 和默认值；
4. `Init`/`Accept`/`Close` 等接口方法；
5. loader 在哪里取出这个构造器；
6. 配置字段如何变成构造 option。

不要先读所有辅助函数。先沿着接口方法走通一次生命周期。

## 编译期接口检查

在具体实现中，可能会看到：

```go
var _ SomeInterface = (*SomeType)(nil)
```

这不创建可用对象。它利用赋值规则让编译器检查 `*SomeType` 是否实现 `SomeInterface`。如果缺少方法，编译立即失败。

## 接口的隐式实现

Go 类型不需要写 `implements`。只要方法集满足接口，就自动实现。例如：

```go
type Runner interface {
    Run() error
}

type Server struct{}

func (s *Server) Run() error { return nil }
```

那么 `*Server` 就实现了 `Runner`。这让 `core` 可以只定义接口，`x` 在另一个 module 中提供实现，两者不必形成双向依赖。

## `register.go` 也是“功能清单”

文件按 connectors、dialers、handlers、listeners 分组。阅读时可以用它回答：

- 当前发行版内置了哪些能力？
- 新增的实现是否已经进入最终二进制？
- 同一种协议在哪些层出现？

例如 `http2` 可能同时有 connector、dialer、handler、listener。名字相同不表示职责相同，要结合所在分组判断。

## 推荐的下一阶段阅读路线

完成当前仓库后，按以下顺序进入依赖：

1. `core/service`：理解 `Service` 接口。
2. `x/registry`：理解注册表的泛型或具体实现。
3. `x/config/parsing/parser`：理解 CLI/文件如何合并成 `Config`。
4. `x/config/loader`：理解配置如何组装成运行对象。
5. `x/listener/tcp`：选择最简单的 listener。
6. `x/handler/http`：选择一个常见 handler。
7. 再追一条带转发链的 dialer + connector。

## 动手练习

1. 在依赖源码中找到 TCP listener 注册时使用的名字。
2. 找到 service registry 的 `GetAll()` 返回类型。
3. 画出 `-L http://:8080` 最终选择 listener 和 handler 的过程。
4. 临时移除一个空白导入并构建，观察是编译失败还是运行时失败。实验后恢复代码，不要提交这项改动。

完成这些练习后，注册表模式就不再是“魔法”，而是一条可以被搜索和验证的普通调用链。
