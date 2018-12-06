# 广播模式

**示例:** [broadcast](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/broadcast)


`Broadcast` 是 `XClient` 的一个方法， 你可以将一个请求发送到这个服务的所有节点。
如果所有的节点都正常返回，没有错误的话， `Broadcast`将返回其中的一个节点的返回结果。 如果有节点返回错误的话，`Broadcast`将返回这些错误信息中的一个。

```go

func main() {
    ……

	xclient := client.NewXClient("Arith", client.Failover, client.RoundRobin, d, client.DefaultOption)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	for {
		reply := &example.Reply{}
		err := xclient.Broadcast(context.Background(), "Mul", args, reply)
		if err != nil {
			log.Fatalf("failed to call: %v", err)
		}

		log.Printf("%d * %d = %d", args.A, args.B, reply.C)
		time.Sleep(1e9)
	}

}
```