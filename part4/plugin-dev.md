# 插件开发

rpcx为服务器和客户端定义了几个插件接口，在一些处理点上可以调用插件。

## Server Plugin

**示例:** [trace](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/trace)


[go doc](https://godoc.org/github.com/smallnest/rpcx/server#PostConnAcceptPlugin)

```go
type PostConnAcceptPlugin

type PostConnAcceptPlugin interface {
    HandleConnAccept(net.Conn) (net.Conn, bool)
}


type PostReadRequestPlugin

type PostReadRequestPlugin interface {
    PostReadRequest(ctx context.Context, r *protocol.Message, e error) error
}

PostReadRequestPlugin represents .
type PostWriteResponsePlugin

type PostWriteResponsePlugin interface {
    PostWriteResponse(context.Context, *protocol.Message, *protocol.Message, error) error
}

PostWriteResponsePlugin represents .
type PreReadRequestPlugin

type PreReadRequestPlugin interface {
    PreReadRequest(ctx context.Context) error
}

PreReadRequestPlugin represents .
type PreWriteResponsePlugin

type PreWriteResponsePlugin interface {
    PreWriteResponse(context.Context, *protocol.Message) error
}

PreWriteResponsePlugin represents . 
```



## Client Plugin

[go doc](https://godoc.org/github.com/smallnest/rpcx/client#Plugin)

```go
type PluginContainer

type PluginContainer interface {
    Add(plugin Plugin)
    Remove(plugin Plugin)
    All() []Plugin

    DoPreCall(ctx context.Context, servicePath, serviceMethod string, args interface{}) error
    DoPostCall(ctx context.Context, servicePath, serviceMethod string, args interface{}, reply interface{}, err error) error
}

type PostCallPlugin

type PostCallPlugin interface {
    DoPostCall(ctx context.Context, servicePath, serviceMethod string, args interface{}, reply interface{}, err error) error
}

PostCallPlugin is invoked after the client calls a server.
type PreCallPlugin

type PreCallPlugin interface {
    DoPreCall(ctx context.Context, servicePath, serviceMethod string, args interface{}) error
}

PreCallPlugin is invoked before the client calls a server. 
```