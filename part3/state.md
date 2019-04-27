# 服务状态

**示例:** [state](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/state)

`state` 是另外一个元数据。 如果你在元数据中设置了`state=inactive`, 客户端将不能访问这些服务，即使这些服务是"活"着的。

你可以使用临时禁用一些服务，而不是杀掉它们， 这样就实现了服务的降级。 server.

你可以通过 [rpcx-ui](https://github.com/smallnest/rpcx-ui))来时实现禁用和启用的功能。

```go
// server.go
func main() {
	flag.Parse()

	go createServer1(*addr1, "")
	go createServer2(*addr2, "state=inactive")

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


```go
// client.go
xclient := client.NewXClient("Arith", client.Failover, client.RoundRobin, d, client.DefaultOption)
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
