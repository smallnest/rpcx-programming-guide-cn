# 服务注册中心
服务注册中心用来实现服务发现和服务的元数据存储。

当前rpcx支持多种注册中心， 并且支持进程内的注册中心，方便开发测试。

![](registry.png)

rpcx会自动将服务的信息比如服务名，监听地址，监听协议，权重等注册到注册中心，同时还会定时的将服务的吞吐率更新到注册中心。

如果服务意外中断或者宕机，注册中心能够监测到这个事件，它会通知客户端这个服务目前不可用，在服务调用的时候不要再选择这个服务器。

客户端初始化的时候会从注册中心得到服务器的列表，然后根据不同的路由选择选择合适的服务器进行服务调用。 同时注册中心还会通知客户端某个服务暂时不可用。

通常客户端会选择一个服务器进行调用。

下面看看不同的注册中心的使用情况。

## Peer2Peer {#peer2peer}
点对点是最简单的一种注册中心的方式，事实上没有注册中心，客户端直接得到唯一的服务器的地址。

### 服务器

服务器并没有配置注册中心，而是直接启动。

```go
package main

import (
	"flag"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/server"
)

var (
	addr = flag.String("addr", "localhost:8972", "server address")
)

func main() {
	flag.Parse()

	s := server.Server{}
	s.RegisterName("Arith", new(example.Arith), "")
	s.Serve("tcp", *addr)
}
```


### 客户端

客户端直接配置了服务器的地址，格式是`network@ipaddress:port`的格式，并没有通过第三方组件来查找。

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
	xclient := client.NewXClient("Arith", "Mul", client.Failtry, client.RandomSelect, d, client.DefaultOption)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	err := xclient.Call(context.Background(), args, reply)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)

}
```

当然，如果服务器宕机，客户端访问就会报错。

## MultipleServers {#multiple}

上面的方式只能访问一台服务器，假设我们有固定的几台服务器提供相同的服务，我们可以采用这种方式。

### 服务器

服务器还是和上面的代码一样，只需启动自己的服务，不需要做额外的配置。下面这个例子启动了两个服务。

```go
package main

import (
	"flag"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/server"
)

var (
	addr1 = flag.String("addr1", "localhost:8972", "server1 address")
	addr2 = flag.String("addr2", "localhost:9981", "server2 address")
)

func main() {
	flag.Parse()

	go createServer(*addr1)
	go createServer(*addr2)

	select {}
}

func createServer(addr string) {
	s := server.NewServer(nil)
	s.RegisterName("Arith", new(example.Arith), "")
	s.Serve("tcp", addr)
}
```

### 客户端

客户度需要使用`MultipleServersDiscovery`来配置同一个服务的多个服务器地址，这样客户端就能基于规则从中选择一个进行调用。

可以看到，除了初始化`XClient`有所不同外，实际调用服务是一样的， 后面介绍的注册中心也是一样，只有初始化客户端有所不同，后续的调用都一样。

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
	addr1 = flag.String("addr1", "tcp@localhost:8972", "server1 address")
	addr2 = flag.String("addr2", "tcp@localhost:9981", "server2 address")
)

func main() {
	flag.Parse()

	d := client.NewMultipleServersDiscovery([]*client.KVPair{{Key: *addr1}, {Key: *addr2}})
	xclient := client.NewXClient("Arith", "Mul", client.Failtry, client.RandomSelect, d, client.DefaultOption)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	err := xclient.Call(context.Background(), args, reply)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)

}
```

## ZooKeeper {#zookeeper}

### 服务器


### 客户端

## Etcd {#etcd}

### 服务器


### 客户端

## Consul {#consul}

### 服务器


### 客户端

## mDNS {#mdns}

### 服务器


### 客户端


## Inprocess {#inpreocess}

### 服务器


### 客户端


