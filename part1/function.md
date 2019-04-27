# Client

**Example:** [function](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/function)

通常我们将方法注册为服务的方法，这些方法必须满足以下的要求：

- 必须是可导出类型的方法
- 接受3个参数，第一个是 `context.Context`类型，其他2个都是可导出（或内置）的类型。
- 第3个参数是一个指针
- 有一个 error 类型的返回值

Rpcx 也支持将纯函数注册为服务，函数必须满足以下的要求：

- 函数可以是可导出的或者不可导出的
- 接受3个参数，第一个是 `context.Context`类型，其他2个都是可导出（或内置）的类型。
- 第3个参数是一个指针
- 有一个 error 类型的返回值

下面有一个例子。

服务端必须使用 `RegisterFunction` 来注册一个函数并且提供一个服务名。

```go
// server.go
type Args struct {
	A int
	B int
}

type Reply struct {
	C int
}

func mul(ctx context.Context, args *Args, reply *Reply) error {
	reply.C = args.A * args.B
	return nil
}

func main() {
	flag.Parse()

	s := server.NewServer()
	s.RegisterFunction("a.fake.service", mul, "")
	s.Serve("tcp", *addr)
}
```

客户端可以通过服务名和函数名来调用服务：

```go
// client.go
d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")
xclient := client.NewXClient("a.fake.service", client.Failtry, client.RandomSelect, d, client.DefaultOption)
defer xclient.Close()

args := &example.Args{
    A: 10,
    B: 20,
}

reply := &example.Reply{}
err := xclient.Call(context.Background(), "mul", args, reply)
if err != nil {
    log.Fatalf("failed to call: %v", err)
}

log.Printf("%d * %d = %d", args.A, args.B, reply.C)
```
