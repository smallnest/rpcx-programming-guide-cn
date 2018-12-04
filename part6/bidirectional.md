# Bidirectional commnunication

**Example:** [bidirectional](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/bidirectional)

In normal case, clients send requests to services and services returns reponses to clients. It is the `request-response` rpc model.

But for some users, they want services can send commands or notifications to clients initiatively. It can be implemented by installing a service on the prior client and a client on the prior service but it is redundant and complicated.


Rpcx implements a simple notification model.

You should cache the connection and business user ID in order that yu know you want to send notifications to which client.

## Server

Server can use `SendMessage` to send messages to clients and data is `[]byte`. You should use `servicePath` and `serviceMethod` to indicate which notification the data is.

You can get the `net.Conn` by `ctx.Value(server.RemoteConnContextKey)` in service.

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

You must use `NewBidirectionalXClient` to create XClient by passing a channel. Then yuou can range the channel to get the message.

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