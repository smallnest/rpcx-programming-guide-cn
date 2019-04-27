# 超时

**示例:** [timeout](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/timeout)

超时机制可以保护服务调用陷入无限的等待之中。超时定义了服务的最长等待时间，如果在给定的时间没有相应，服务调用就进入下一个状态，或者重试、或者立即返回错误。

## Server

你可以使用`OptionFn`设置服务器的 `readTimeout` 和 `writeTimeout`。

```go
type Server struct {
	……
	readTimeout  time.Duration
	writeTimeout time.Duration
	……
}
```

设置超时的`OptionFn`是 :

```go
func WithReadTimeout(readTimeout time.Duration) OptionFn
func WithWriteTimeout(writeTimeout time.Duration) OptionFn 
```

## Client

客户端有两种方式可是超时。

一种是设置连接的 read/write deadline, 一种是使用 `context.Context`.

### read/write deadline

Client的 `Option` 可以设置连接的超时值:

```go

type Option struct {
	……
	//ConnectTimeout sets timeout for dialing
	ConnectTimeout time.Duration
	// ReadTimeout sets readdeadline for underlying net.Conns
	ReadTimeout time.Duration
	// WriteTimeout sets writedeadline for underlying net.Conns
	WriteTimeout time.Duration
	……
}
```

`DefaultOption` 设置连接超时值为 10 秒，但是没有设置 ReadTimeout 和 WriteTimeout。 如果没有设置，则不会有超时限制。

由于多个服务可能共用同一个节点，有可能出现多个服务调用互相影响的状况。

### context.Context

`context.Context` 也可以用来控制超时。

你可以使用`context.WithTimeout` 来设置超时时间，这是推荐的设置超时的方式。

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```