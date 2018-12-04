# Server

**Example:** [102basic](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/102basic)


你可以在服务端实现Service。

Service的类型并不重要。你可以使用自定义类型来保持状态，或者直接使用 `struct{}`、 `int`。

你需要启动一个TCP或UDP服务器来暴露Service。

你也可以添加一些plugin来为服务器增加新特性。

## Service

作为服务提供者，首先你需要定义服务。
当前rpcx仅支持 可导出的 `methods` （方法） 作为服务的函数。 （see [可导出](https://tour.golang.org/basics/3)）
并且这个可导出的方法必须满足以下的要求：

- 必须是可导出类型的方法
- 接受3个参数，第一个是 `context.Context`类型，其他2个都是可导出（或内置）的类型。
- 第3个参数是一个指针
- 有一个 error 类型的返回值

你可以使用 `RegisterName` 来注册 `rcvr` 的方法，这里这个服务的名字叫做 `name`。
如果你使用 `Register`， 生成的服务的名字就是 `rcvr`的类型名。
你可以在注册中心添加一些元数据供客户端或者服务管理者使用。例如 `weight`、`geolocation`、`metrics`。

```
func (s *Server) Register(rcvr interface{}, metadata string) error
func (s *Server) RegisterName(name string, rcvr interface{}, metadata string) error
```

这里是一个实现了 `Mul` 方法的例子：

```go
import "context"

type Args struct {
	A int
	B int
}

type Reply struct {
	C int
}

type Arith int

func (t *Arith) Mul(ctx context.Context, args *Args, reply *Reply) error {
	reply.C = args.A * args.B
	return nil
}
```

在这个例子中，你可以定义 `Arith` 为 `struct{}` 类型， 它不会影响到这个服务。
你也可以定义 `args` 为 `Args`， 也不会产生影响。


## Server

在你定义完服务后，你会想将它暴露出去来使用。你应该通过启动一个TCP或UDP服务器来监听请求。

服务器支持以如下这些方式启动，监听请求和关闭：

```go
    func NewServer(options ...OptionFn) *Server
    func (s *Server) Close() error
    func (s *Server) RegisterOnShutdown(f func())
    func (s *Server) Serve(network, address string) (err error)
    func (s *Server) ServeHTTP(w http.ResponseWriter, req *http.Request)
```

首先你应使用 `NewServer` 来创建一个服务器实例。其次你可以调用 `Serve` 或者 `ServeHTTP` 来监听请求。

服务器包含一些字段（有一些是不可导出的）：
```go
type Server struct {
    Plugins PluginContainer
    // AuthFunc 可以用来鉴权
    AuthFunc func(ctx context.Context, req *protocol.Message, token string) error
    // 包含过滤后或者不可导出的字段
}
```

`Plugins` 包含了服务器上所有的插件。我们会在之后的章节介绍它。

`AuthFunc` 是一个可以检查客户端是否被授权了的鉴权函数。我们也会在之后的章节介绍它。

rpcx 提供了 3个 `OptionFn` 来设置启动选项：

```go

    func WithReadTimeout(readTimeout time.Duration) OptionFn
    func WithTLSConfig(cfg *tls.Config) OptionFn
    func WithWriteTimeout(writeTimeout time.Duration) OptionFn
```

可以设置 读超时、写超时和tls证书。

`ServeHTTP` 将服务通过HTTP暴露出去。

`Serve` 通过TCP或UDP协议与客户端通信。

rpcx 支持如下的网络类型：

- tcp: 推荐使用
- http: 通过劫持http连接实现
- unix: unix domain sockets
- reuseport: 要求 `SO_REUSEPORT` socket 选项, 仅支持 Linux kernel 3.9+
- quic: support [quic protocol](https://en.wikipedia.org/wiki/QUIC)
- kcp: sopport [kcp protocol](https://github.com/skywind3000/kcp)


下面是一个服务器的示例代码：

```go
package main

import (
	"flag"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/server"
)

var (
	addr = flag.String("addr", "localhost:8972", "server address")
)

func main() {
	flag.Parse()

	s := server.NewServer()
	//s.RegisterName("Arith", new(example.Arith), "")
	s.Register(new(example.Arith), "")
	s.Serve("tcp", *addr)
}
```