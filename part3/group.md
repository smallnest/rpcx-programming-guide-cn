# 分组

**分组:** [group](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/group)

当你在服务器端注册服务的时候，你可能注意到第三个参数我们一般设置它为空的字符串，事实上你可以为服务增加一些元数据。

你可以通过UI管理器查看服务的元数据 [rpcx-ui](https://github.com/smallnest/rpcx-ui))，或者增删一些元数据。


`group` 就是一个元数据。如果你为服务设置了设置`group`， 只有在这个`group`的客户端才能访问这些服务(这个限制是在路由的时候限制的， 当然你在客户端绕过这个限制)。


```go
// server.go

func main() {
	flag.Parse()

	go createServer1(*addr1, "")
	go createServer2(*addr2, "group=test")

	select {}
}

func createServer1(addr, meta string) {
	s := server.NewServer()
	s.RegisterName("Arith", new(example.Arith), meta)
	s.Serve("tcp", addr)
}

func createServer2(addr, meta string) {
	s := server.NewServer()
	s.RegisterName("Arith", new(Arith), meta)
	s.Serve("tcp", addr)
}
```

客户端通过 `option.Group` 设置组。

如果在客户端你没有设置 `option.Group`, 客户端可以访问这些服务， 无论服务是否设置了组还是没设置。

```go
// client.go
option := client.DefaultOption
option.Group = "test"
xclient := client.NewXClient("Arith", client.Failover, client.RoundRobin, d, option)
defer xclient.Close()

args := &example.Args{
    A: 10,
    B: 20,
}

for {
    reply := &example.Reply{}
    err := xclient.Call(context.Background(), "Mul", args, reply)
    if err != nil {
        log.Fatalf("failed to call: %v", err)
    }

    log.Printf("%d * %d = %d", args.A, args.B, reply.C)
    time.Sleep(1e9)
}
```