# 网关

[Gateway](https://github.com/rpcx-ecosystem/rpcx-gateway) 为 rpcx services 提供了 http 网关服务.

你可以使用你熟悉的编程语言， 比如Java、Python、C#、Node.js、Php、C\C++、Rust等等来调用 rpcx 服务。查看一些编程语言实现的[例子](https://github.com/rpcx-ecosystem/rpcx-gateway/blob/master/examples/README.md) 。

这意味着，你可以不用实现 rpcx的协议，而是使用熟悉的 http 访问方式调用 rpcx 服务， 设置用`curl`、`wget`等命令行工具。

## 部署模型

使用网关程序有两种部署模型: **Gateway** 和 **Agent**。

### Gateway

![](https://github.com/rpcx-ecosystem/rpcx-gateway/raw/master/doc/gateway.png)

你可以部署为网关模式。网关程序运行在独立的机器上，所有的client都将http请求发送给gateway, gateway负责将请求转换成rpcx的请求，并调用相应的rpcx 服务， 它将rpcx 的返回结果转换成http的response, 返回给client。

你可以部署多台gateway程序， 并可以利用nginx等进行负载均衡。


### Agent

![](https://github.com/rpcx-ecosystem/rpcx-gateway/raw/master/doc/agent.png)

你可以将网关程序和1你的client一起部署， agent作为一个后台服务部署在 client机器上。 如果你的机器有多个client, 你只需部署一个agent。

Client发送 http 请求到本地的agent, 本地的agent将请求转为 rpcx请求，然后转发到相应的 rpcx服务上， 然后将 rpcx的response转换为 http response返回给 client。

它类似 mesh service的 Sidecar模式， 但是比较好的是， 同一台机器上的client只需一个agent。

## http协议 

你可以使用任意的编程语言来发送http请求，但是需要额外设置一些header (但是不一定设置全部的header, 按需设置):

- X-RPCX-Version: rpcx 版本
- X-RPCX-MesssageType: 设置为0,代表请求
- X-RPCX-Heartbeat: 是否是heartbeat请求, 缺省false
- X-RPCX-Oneway: 是否是单向请求, 缺省false.
- X-RPCX-SerializeType: 0 as raw bytes, 1 as JSON, 2 as protobuf, 3 as msgpack
- X-RPCX-MessageID: 消息id, uint64 类型
- X-RPCX-ServicePath: service path
- X-RPCX-ServiceMethod: service method
- X-RPCX-Meta: 额外的元数据

对于 http response, 可能会包含一些header: 

- X-RPCX-Version: rpcx 版本
- X-RPCX-MesssageType: 1 ,代表response
- X-RPCX-Heartbeat: 是否是heartbeat请求
- X-RPCX-MessageStatusType:  Error 还是正常返回结果
- X-RPCX-SerializeType: 0 as raw bytes, 1 as JSON, 2 as protobuf, 3 as msgpack
- X-RPCX-MessageID: 消息id, uint64 类型
- X-RPCX-ServicePath: service path
- X-RPCX-ServiceMethod: service method
- X-RPCX-Meta: extra metadata
- X-RPCX-ErrorMessage: 错误信息,  如果 X-RPCX-MessageStatusType 是 Error的花


## 例子

下面是Go的一个例子:

```go
package main

import (
	"bytes"
	"io/ioutil"
	"log"
	"net/http"

	"github.com/rpcx-ecosystem/rpcx-gateway"

	"github.com/smallnest/rpcx/codec"
)

type Args struct {
	A int
	B int
}

type Reply struct {
	C int
}

func main() {
	cc := &codec.MsgpackCodec{}

    // request 
	args := &Args{
		A: 10,
		B: 20,
	}
	data, _ := cc.Encode(args)

	req, err := http.NewRequest("POST", "http://127.0.0.1:9981/", bytes.NewReader(data))
	if err != nil {
		log.Fatal("failed to create request: ", err)
		return
    }
    
    // set extra headers
	h := req.Header
	h.Set(gateway.XMessageID, "10000")
	h.Set(gateway.XMessageType, "0")
	h.Set(gateway.XSerializeType, "3")
	h.Set(gateway.XServicePath, "Arith")
	h.Set(gateway.XServiceMethod, "Mul")

    // send to gateway
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

    // parse reply
	reply := &Reply{}
	err = cc.Decode(replyData, reply)
	if err != nil {
		log.Fatal("failed to decode reply: ", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)
}
```