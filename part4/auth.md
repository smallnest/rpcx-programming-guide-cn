# 身份认证

**示例:** [auth](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/auth)

出于安全的考虑， 很多场景下只有授权的客户端才可以调用服务。

客户端必须设置一个 `token`, 这个`token`可以从其他的 OAuth/OAuth2 服务器获得，或者由服务提供者分配。

服务接收到请求后，需要验证这个`token`。如果是`token`是从 OAuth2服务器中申请到，则服务需要到OAuth2服务中去验证， 如果是自己分配的，则需要和自己的记录进行对别。

因为rpcx提供的是一个身份验证的框架，所以具体的身份验证需要自己集成和验证。


```go
func main() {
	flag.Parse()

	s := server.NewServer()
	s.RegisterName("Arith", new(example.Arith), "")
	s.AuthFunc = auth
	s.Serve("reuseport", *addr)
}

func auth(ctx context.Context, req *protocol.Message, token string) error {

	if token == "bearer tGzv3JOkF0XG5Qx2TlKWIA" {
		return nil
	}

	return errors.New("invalid token")
}
```

服务器必须定义 `AuthFunc` 来验证token。在上面的例子中， 只有token为`bearer tGzv3JOkF0XG5Qx2TlKWIA` 才是合法的客户端。


客户端必须设置这个`toekn`:

```go

func main() {
	flag.Parse()

	d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")

	option := client.DefaultOption
	option.ReadTimeout = 10 * time.Second

	xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, option)
	defer xclient.Close()

	//xclient.Auth("bearer tGzv3JOkF0XG5Qx2TlKWIA")
	xclient.Auth("bearer abcdefg1234567")

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	ctx := context.WithValue(context.Background(), share.ReqMetaDataKey, make(map[string]string))
	err := xclient.Call(ctx, "Mul", args, reply)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)

}
```

注意: **你必须设置 `map[string]string` 为 `share.ReqMetaDataKey`的值，否则调用会出错**