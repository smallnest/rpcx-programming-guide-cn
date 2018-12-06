# 别名

**示例:** [alias](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/alias)

这个插件可以为一个服务方法设置一个别名。

下面的代码使用 `Arith`的别名`a.b.c.d`， 为`Mul`设置别名`Times`。

```go
func main() {
	flag.Parse()

	a := serverplugin.NewAliasPlugin()
	a.Alias("a.b.c.d", "Times", "Arith", "Mul")
	s := server.NewServer()
	s.Plugins.Add(a)
	s.RegisterName("Arith", new(example.Arith), "")
	err := s.Serve("reuseport", *addr)
	if err != nil {
		panic(err)
	}
}
```


客户端可以使用别名来调用服务：

```go
func main() {
	flag.Parse()

	d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")

	option := client.DefaultOption
	option.ReadTimeout = 10 * time.Second

	xclient := client.NewXClient("a.b.c.d", client.Failtry, client.RandomSelect, d, option)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	err := xclient.Call(context.Background(), "Times", args, reply)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)

}
```