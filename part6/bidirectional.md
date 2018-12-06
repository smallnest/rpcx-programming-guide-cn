# 双向通讯

**示例:** [bidirectional](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/bidirectional)

在正常情况下， 客户端发送请求，服务器返回结果，这样一问一答的方式就是`request-response` rpc 模型。

但是对于一些用户， 比如 `IoT` 的开发者， 可能需要在某些时候发送通知给客户端。 如果客户端和服务端都配两套代码就显得多余和臃肿了。

rpcx实现了一个简单的通知机制。

首先你需要缓存客户端的连接，可能还需要将用户的ID和连接进行关联， 以便服务器知道将通知发送给哪个客户端。


## Server

服务器使用`SendMessage`方法发送通知， 数据是`[]byte`类型。 你可以设置 `servicePath` 和 `serviceMethod`以便提供给客户端更多的信息，用来区分不同的通知。

 `net.Conn` 对象可以在客户端调用服务的时候从`ctx.Value(server.RemoteConnContextKey)`中获取。

```go
func (s *Server) SendMessage(conn net.Conn, servicePath, serviceMethod string, metadata map[string]string, data []byte) error
```


```go server.go
func main() {
	flag.Parse()

	ln, _ := net.Listen("tcp", ":9981")
	go http.Serve(ln, nil)

	s := server.NewServer()
	//s.RegisterName("Arith", new(example.Arith), "")
	s.Register(new(Arith), "")
	go s.Serve("tcp", *addr)

	for !connected {
		time.Sleep(time.Second)
	}

	fmt.Printf("start to send messages to %s\n", clientConn.RemoteAddr().String())
	for {
		if clientConn != nil {
			err := s.SendMessage(clientConn, "test_service_path", "test_service_method", nil, []byte("abcde"))
			if err != nil {
				fmt.Printf("failed to send messsage to %s: %v\n", clientConn.RemoteAddr().String(), err)
				clientConn = nil
			}
		}
		time.Sleep(time.Second)
	}
}
```

## Client

你必须使用 `NewBidirectionalXClient` 创建 XClient 客户端， 你需要传如一个channel， 这样你就可以从channel中读取通知了。

```go client.go
func main() {
	flag.Parse()

	ch := make(chan *protocol.Message)

	d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")
	xclient := client.NewBidirectionalXClient("Arith", client.Failtry, client.RandomSelect, d, client.DefaultOption, ch)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	err := xclient.Call(context.Background(), "Mul", args, reply)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)

	for msg := range ch {
		fmt.Printf("receive msg from server: %s\n", msg.Payload)
	}
}
```