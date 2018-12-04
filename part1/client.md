# Client

**Example:** [101basic](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/101basic)

客户端使用和服务同样的通信协议来发送请求和获取响应。


```go
type Client struct {
    Conn net.Conn

    Plugins PluginContainer
    // 包含过滤后的或者不可导出的字段
}
```

`Conn` 代表客户端与服务器之前的连接。 `Plugins` 包含了客户端启用的插件。


他有这些方法：

```go
    func (client *Client) Call(ctx context.Context, servicePath, serviceMethod string, args interface{}, reply interface{}) error
    func (client *Client) Close() error
    func (c *Client) Connect(network, address string) error
    func (client *Client) Go(ctx context.Context, servicePath, serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call
    func (client *Client) IsClosing() bool
    func (client *Client) IsShutdown() bool
```

`Call` 代表对服务同步调用。客户端在收到响应或错误前一直是阻塞的。 然而 `Go` 是异步调用。它返回一个指向 Call 的指针， 你可以检查 `*Call` 的值来获取返回的结果或错误。

`Close` 会关闭所有与服务的连接。他会立刻关闭连接，不会等待未完成的请求结束。

`IsClosing` 表示客户端是关闭着的并且不会接受新的调用。
`IsShutdown` 表示客户端不会接受服务返回的响应。


`Client` uses the default [CircuitBreaker (circuit.NewRateBreaker(0.95, 100))](https://godoc.org/github.com/rubyist/circuitbreaker#NewRateBreaker) to handle errors. This is a poplular rpc error handling style. When the error rate hits the threshold, this service is marked unavailable in 10 second window. You can implement your customzied CircuitBreaker.
`Client` 使用默认的 [CircuitBreaker (circuit.NewRateBreaker(0.95, 100))](https://godoc.org/github.com/rubyist/circuitbreaker#NewRateBreaker) 来处理错误。这是rpc处理错误的普遍做法。当出错率达到阈值， 这个服务就会在接下来的10秒内被标记为不可用。你也可以实现你自己的 CircuitBreaker。

下面是客户端的例子：

```go
	client := &Client{
		option: DefaultOption,
	}

	err := client.Connect("tcp", addr)
	if err != nil {
		t.Fatalf("failed to connect: %v", err)
	}
	defer client.Close()

	args := &Args{
		A: 10,
		B: 20,
	}

	reply := &Reply{}
	err = client.Call(context.Background(), "Arith", "Mul", args, reply)
	if err != nil {
		t.Fatalf("failed to call: %v", err)
	}

	if reply.C != 200 {
		t.Fatalf("expect 200 but got %d", reply.C)
	}
```


## XClient

`XClient` 是对客户端的封装，增加了一些服务发现和服务治理的特性。

```go
type XClient interface {
    SetPlugins(plugins PluginContainer)
    ConfigGeoSelector(latitude, longitude float64)
    Auth(auth string)

    Go(ctx context.Context, serviceMethod string, args interface{}, reply interface{}, done chan *Call) (*Call, error)
    Call(ctx context.Context, serviceMethod string, args interface{}, reply interface{}) error
    Broadcast(ctx context.Context, serviceMethod string, args interface{}, reply interface{}) error
    Fork(ctx context.Context, serviceMethod string, args interface{}, reply interface{}) error
    Close() error
}
```

`SetPlugins` 方法可以用来设置 Plugin 容器， `Auth` 可以用来设置鉴权token。

`ConfigGeoSelector` 是一个可以通过地址位置选择器来设置客户端的经纬度的特别方法。

一个XCLinet只对一个服务负责，它可以通过`serviceMethod`参数来调用这个服务的所有方法。如果你想调用多个服务，你必须为每个服务创建一个XClient。

一个应用中，一个服务只需要一个共享的XClient。它可以被通过goroutine共享，并且是协程安全的。

`Go` 代表异步调用， `Call` 代表同步调用。

XClient对于一个服务节点使用单一的连接，并且它会缓存这个连接直到失效或异常。

### 服务发现

rpcx 支持许多服务发现机制，你也可以实现自己的服务发现。

- * [Peer to Peer](../part2/registry.md#peer2peer): 客户端直连每个服务节点。 the client connects the single service directly. It acts like the `client` type.
  * [Peer to Multiple](../part2/registry.md#multiple): 客户端可以连接多个服务。服务可以被编程式配置。
  * [Zookeeper](../part2/registry.md#zookeeper): 通过 zookeeper 寻找服务。
  * [Etcd](../part2/registry.md#etcd): 通过 etcd 寻找服务。
  * [Consul](../part2/registry.md#consul): 通过 consul 寻找服务。
  * [mDNS](../part2/registry.md#mdns): 通过 mDNS 寻找服务（支持本地服务发现）。
  * [In process](../part2/registry.md#inprocess): 在同一进程寻找服务。客户端通过进程调用服务，不走TCP或UDP，方便调试使用。


下面是一个同步的 rpcx 例子：

```go
package main

import (
	"context"
	"flag"
	"log"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/client"
)

var (
	addr = flag.String("addr", "localhost:8972", "server address")
)

func main() {
	flag.Parse()

	d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")
	xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, client.DefaultOption)
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

}
```

### 服务治理 (失败模式与负载均衡)

在一个大规模的rpc系统中，有许多服务节点提供同一个服务。客户端如何选择最合适的节点来调用呢？如果调用失败，客户端应该选择另一个节点或者立即返回错误？这里就有了故障模式和负载均衡的问题。

rpcx 支持 故障模式：

- [Failfast](../part3/failmode.md#failfast)：如果调用失败，立即返回错误
- [Failover](../part3/failmode.md#failover)：选择其他节点，直到达到最大重试次数
- [Failtry](../part3/failmode.md#failtry)：选择相同节点并重试，直到达到最大重试次数


对于负载均衡，rpcx 提供了许多选择器：

 * [Random](../part3/selector.md#random_selector)： 随机选择节点
  * [Roundrobin](../part3/selector.md#roundrobin_selector)： 使用 [roundrobin](https://zh.wikipedia.org/wiki/%E5%BE%AA%E7%92%B0%E5%88%B6) 算法选择节点
  * [Consistent hashing](../part3/selector.md#hash_selector): 如果服务路径、方法和参数一致，就选择同一个节点。使用了非常快的[jump consistent hash](https://arxiv.org/abs/1406.2294)算法。
  * [Weighted](../part3/selector.md#weighted_selector): 根据元数据里配置好的权重(`weight=xxx`)来选择节点。类似于nginx里的实现(smooth weighted algorithm)
  * [Network quality](../part3/selector.md#ping_selector): 根据`ping`的结果来选择节点。网络质量越好，该节点被选择的几率越大。
  * [Geography](../part3/selector.md#geo_selector): 如果有多个数据中心，客户端趋向于连接同一个数据机房的节点。
  * [Customized Selector](../part3/selector.md#user_selector): 如果以上的选择器都不适合你，你可以自己定制选择器。例如一个rpcx用户写过它自己的选择器，他有2个数据中心，但是这些数据中心彼此有限制，不能使用 `Network quality` 来检测连接质量。


下面是一个异步的 rpcx 例子：

```go
package main

import (
	"context"
	"flag"
	"log"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/client"
)

var (
	addr2 = flag.String("addr", "localhost:8972", "server address")
)

func main() {
	flag.Parse()

	d := client.NewPeer2PeerDiscovery("tcp@"+*addr2, "")
	xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, client.DefaultOption)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	call, err := xclient.Go(context.Background(), "Mul", args, reply, nil)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	replyCall := <-call.Done
	if replyCall.Error != nil {
		log.Fatalf("failed to call: %v", replyCall.Error)
	} else {
		log.Printf("%d * %d = %d", args.A, args.B, reply.C)
	}
}
```

客户端使用了 `Failtry` 模式并且随机选择节点。

### 广播与群发

特殊情况下，你可以使用 XClient 的 `Broadcast` 和 `Fork` 方法。

```go
    Broadcast(ctx context.Context, serviceMethod string, args interface{}, reply interface{}) error
	Fork(ctx context.Context, serviceMethod string, args interface{}, reply interface{}) error
```

`Broadcast` 表示向所有服务器发送请求，只有所有服务器正确返回时才会成功。此时FailMode 和 SelectMode的设置是无效的。请设置超时来避免阻塞。

`Fork` 表示向所有服务器发送请求，只要任意一台服务器正确返回就成功。此时FailMode 和 SelectMode的设置是无效的。

你可以使用 `NewXClient` 来获取一个 XClient 实例。

```go
func NewXClient(servicePath string, failMode FailMode, selectMode SelectMode, discovery ServiceDiscovery, option Option) XClient
```

`NewXClient` 必须使用服务名称作为第一个参数， 然后是 failmode、 selector、 discovery等其他选项。
