# HTTP 调用

大部分场景下， rpcx服务是通过 TCP 进行通讯的， 但是你也可以直接通过 http 进行访问， http请求需要设置一些 header, 这和 [gateway](https://github.com/rpcx-ecosystem/rpcx-gateway) 中的 header 是一样的。

很明显，通过http调用不可能取得和 TCP 一样的性能， 因为 http 调用是`一问一答`方式进行通讯的， 并不能并发的请求(除非你使用很多client)， 但是调用方式简单， 也可以应用在一些场景中。

可以http调用的服务必须使用 TCP 方式部署，而不能使用 UDP或者其他方式， 它和TCP共用同一个接口。 一个连接只能使用http调用或者TCP调用。

你可以使用你熟悉的语言进行调用，设置你使用的编解码器，最常用的是JSON编解码，注意一定要添加相应的header,否则rpcx服务不知道该如果处理这个htp请求。

下面是一个http调用的例子:

```go
func main() {
	cc := &codec.MsgpackCodec{}

	args := &Args{
		A: 10,
		B: 20,
	}

	data, _ := cc.Encode(args)

	req, err := http.NewRequest("POST", "http://127.0.0.1:8972/", bytes.NewReader(data))
	if err != nil {
		log.Fatal("failed to create request: ", err)
		return
	}

	h := req.Header
	h.Set(gateway.XMessageID, "10000")
	h.Set(gateway.XMessageType, "0")
	h.Set(gateway.XSerializeType, "3")
	h.Set(gateway.XServicePath, "Arith")
	h.Set(gateway.XServiceMethod, "Mul")

	res, err := http.DefaultClient.Do(req)
	if err != nil {
		log.Fatal("failed to call: ", err)
	}
	defer res.Body.Close()

	// handle http response
	replyData, err := ioutil.ReadAll(res.Body)
	if err != nil {
		log.Fatal("failed to read response: ", err)
	}

	reply := &Reply{}
	err = cc.Decode(replyData, reply)
	if err != nil {
		log.Fatal("failed to decode reply: ", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)
}
```

不能进行双向通讯， 也就是服务端不能主动发送请求给客户端。