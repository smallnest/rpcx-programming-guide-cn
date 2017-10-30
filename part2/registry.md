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

Apache ZooKeeper是Apache软件基金会的一个软件项目，他为大型分布式计算提供开源的分布式配置服务、同步服务和命名注册。 ZooKeeper曾经是Hadoop的一个子项目，但现在是一个独立的顶级项目。

ZooKeeper的架构通过冗余服务实现高可用性。因此，如果第一次无应答，客户端就可以询问另一台ZooKeeper主机。ZooKeeper节点将它们的数据存储于一个分层的命名空间，非常类似于一个文件系统或一个前缀树结构。客户端可以在节点读写，从而以这种方式拥有一个共享的配置服务。更新是全序的。

使用ZooKeeper的公司包括Rackspace、雅虎和eBay，以及类似于象Solr这样的开源企业级搜索系统。

ZooKeeper Atomic Broadcast (ZAB)协议是一个类似Paxos的协议，但也有所[不同](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos)。

Zookeeper一个应用场景就是服务发现，这在Java生态圈中得到了广泛的应用。Go也可以使用Zookeeper，尤其是在和Java项目混布的情况。

### 服务器

基于rpcx用户的反馈， rpcx 3.0进行了重构，目标之一就是对rpcx进行简化， 因为有些用户可能只需要zookeeper的特性，而不需要etcd、consul等特性。rpcx解决这个问题的方式就是使用`tag`，需要你在编译的时候指定所需的特性的`tag`。

比如下面这个例子， 需要加上`-tags zookeeper`这个参数， 如果需要多个特性，可以使用`-tags "tag1 tag2 tag3"`这样的参数。

服务端使用Zookeeper唯一的工作就是设置`ZooKeeperRegisterPlugin`这个插件。

它主要配置几个参数：

* ServiceAddress: 本机的监听地址， 这个对外暴露的监听地址， 格式为`tcp@ipaddress:port`
* ZooKeeperServers: Zookeeper集群的地址
* BasePath: 服务前缀。 如果有多个项目同时使用zookeeper，避免命名冲突，可以设置这个参数，为当前的服务设置命名空间
* Metrics: 用来更新服务的TPS
* UpdateInterval: 服务的刷新间隔， 如果在一定间隔内(当前设为2 * UpdateInterval)没有刷新,服务就会从Zookeeper中删除

**需要说明的是：插件必须在注册服务之前添加到Server中，否则插件没有办法获取注册的服务的信息。**

```go
// go run -tags zookeeper server.go
package main

import (
	"flag"
	"log"
	"time"

	metrics "github.com/rcrowley/go-metrics"
	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/server"
	"github.com/smallnest/rpcx/serverplugin"
)

var (
	addr     = flag.String("addr", "localhost:8972", "server address")
	zkAddr   = flag.String("zkAddr", "localhost:2181", "zookeeper address")
	basePath = flag.String("base", "/rpcx_test", "prefix path")
)

func main() {
	flag.Parse()

	s := server.NewServer(nil)
	addRegistryPlugin(s)

	s.RegisterName("Arith", new(example.Arith), "")
	s.Serve("tcp", *addr)
}

func addRegistryPlugin(s *server.Server) {

	r := &serverplugin.ZooKeeperRegisterPlugin{
		ServiceAddress:   "tcp@" + *addr,
		ZooKeeperServers: []string{*zkAddr},
		BasePath:         *basePath,
		Metrics:          metrics.NewRegistry(),
		UpdateInterval:   time.Minute,
	}
	err := r.Start()
	if err != nil {
		log.Fatal(err)
	}
	s.Plugins.Add(r)
}
```


### 客户端

客户端需要设置 `ZookeeperDiscovery`, 指定`basePath`和zookeeper集群的地址。
```go
// go run -tags zookeeper client.go
package main

import (
	"context"
	"flag"
	"log"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/client"
)

var (
	zkAddr   = flag.String("zkAddr", "localhost:2181", "zookeeper address")
	basePath = flag.String("base", "/rpcx_test/Arith", "prefix path")
)

func main() {
	flag.Parse()

	d := client.NewZookeeperDiscovery(*basePath, []string{*zkAddr}, nil)
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


## Etcd {#etcd}

etcd 是 CoreOS 团队于 2013 年 6 月发起的开源项目，它的目标是构建一个高可用的分布式键值（key-value）数据库，基于 Go 语言实现。我们知道，在分布式系统中，各种服务的配置信息的管理分享，服务的发现是一个很基本同时也是很重要的问题。CoreOS 项目就希望基于 etcd 来解决这一问题。

因为是用Go开发的，在Go的生态圈中得到广泛的应用。当然，因为etcd提供了RESTful的接口，其它语言也可以使用。

etcd registry使用和zookeeper非常相像。

编译的时候需要加上`etcd` tag。

### 服务器

服务器需要增加`EtcdRegisterPlugin`插件， 配置参数和Zookeeper的插件相同。

它主要配置几个参数：

* ServiceAddress: 本机的监听地址， 这个对外暴露的监听地址， 格式为`tcp@ipaddress:port`
* EtcdServers: etcd集群的地址
* BasePath: 服务前缀。 如果有多个项目同时使用zookeeper，避免命名冲突，可以设置这个参数，为当前的服务设置命名空间
* Metrics: 用来更新服务的TPS
* UpdateInterval: 服务的刷新间隔， 如果在一定间隔内(当前设为2 * UpdateInterval)没有刷新,服务就会从etcd中删除

**再说明一次：插件必须在注册服务之前添加到Server中，否则插件没有办法获取注册的服务的信息。以下的插件相同，就不赘述了**


```go
// go run -tags etcd server.go
package main

import (
	"flag"
	"log"
	"time"

	metrics "github.com/rcrowley/go-metrics"
	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/server"
	"github.com/smallnest/rpcx/serverplugin"
)

var (
	addr     = flag.String("addr", "localhost:8972", "server address")
	etcdAddr = flag.String("etcdAddr", "localhost:2379", "etcd address")
	basePath = flag.String("base", "/rpcx_test", "prefix path")
)

func main() {
	flag.Parse()

	s := server.NewServer(nil)
	addRegistryPlugin(s)

	s.RegisterName("Arith", new(example.Arith), "")
	s.Serve("tcp", *addr)
}

func addRegistryPlugin(s *server.Server) {

	r := &serverplugin.EtcdRegisterPlugin{
		ServiceAddress: "tcp@" + *addr,
		EtcdServers:    []string{*etcdAddr},
		BasePath:       *basePath,
		Metrics:        metrics.NewRegistry(),
		UpdateInterval: time.Minute,
	}
	err := r.Start()
	if err != nil {
		log.Fatal(err)
	}
	s.Plugins.Add(r)
}
```


### 客户端

客户端需要设置`EtcdDiscovery`插件，设置`basepath`和etcd集群的地址。


```
// go run -tags etcd client.go
package main

import (
	"context"
	"flag"
	"log"

	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/client"
)

var (
	etcdAddr = flag.String("etcdAddr", "localhost:2379", "etcd address")
	basePath = flag.String("base", "/rpcx_test/Arith", "prefix path")
)

func main() {
	flag.Parse()

	d := client.NewEtcdDiscovery(*basePath, []string{*etcdAddr}, nil)
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


## Consul {#consul}

Consul是HashiCorp公司推出的开源工具，用于实现分布式系统的服务发现与配置。Consul是分布式的、高可用的、 可横向扩展的。它具备以下特性:

- 服务发现: Consul提供了通过DNS或者HTTP接口的方式来注册服务和发现服务。一些外部的服务通过Consul很容易的找到它所依赖的服务。
- 健康检测: Consul的Client提供了健康检查的机制，可以通过用来避免流量被转发到有故障的服务上。
- Key/Value存储: 应用程序可以根据自己的需要使用Consul提供的Key/Value存储。 Consul提供了简单易用的HTTP接口，结合其他工具可以实现动态配置、功能标记、领袖选举等等功能。
- 多数据中心: Consul支持开箱即用的多数据中心. 这意味着用户不需要担心需要建立额外的抽象层让业务扩展到多个区域。

Consul也是使用Go开发的，在Go生态圈也被广泛应用。

使用`consul`需要添加`consul` tag。

### 服务器

服务器端的开发和zookeeper、consul类似。

需要配置`ConsulRegisterPlugin`插件。

它主要配置几个参数：

* ServiceAddress: 本机的监听地址， 这个对外暴露的监听地址， 格式为`tcp@ipaddress:port`
* ConsulServers: consul集群的地址
* BasePath: 服务前缀。 如果有多个项目同时使用zookeeper，避免命名冲突，可以设置这个参数，为当前的服务设置命名空间
* Metrics: 用来更新服务的TPS
* UpdateInterval: 服务的刷新间隔， 如果在一定间隔内(当前设为2 * UpdateInterval)没有刷新,服务就会从consul中删除


```go
// go run -tags consul server.go
package main

import (
	"flag"
	"log"
	"time"

	metrics "github.com/rcrowley/go-metrics"
	example "github.com/rpcx-ecosystem/rpcx-examples3"
	"github.com/smallnest/rpcx/server"
	"github.com/smallnest/rpcx/serverplugin"
)

var (
	addr       = flag.String("addr", "localhost:8972", "server address")
	consulAddr = flag.String("consulAddr", "localhost:8500", "consul address")
	basePath   = flag.String("base", "/rpcx_test", "prefix path")
)

func main() {
	flag.Parse()

	s := server.NewServer(nil)
	addRegistryPlugin(s)

	s.RegisterName("Arith", new(example.Arith), "")
	s.Serve("tcp", *addr)
}

func addRegistryPlugin(s *server.Server) {

	r := &serverplugin.ConsulRegisterPlugin{
		ServiceAddress: "tcp@" + *addr,
		ConsulServers:  []string{*consulAddr},
		BasePath:       *basePath,
		Metrics:        metrics.NewRegistry(),
		UpdateInterval: time.Minute,
	}
	err := r.Start()
	if err != nil {
		log.Fatal(err)
	}
	s.Plugins.Add(r)
}

```

### 客户端

配置`ConsulDiscovery`，使用`basepath`和consul的地址。

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
	consulAddr = flag.String("consulAddr", "localhost:8500", "consul address")
	basePath   = flag.String("base", "/rpcx_test/Arith", "prefix path")
)

func main() {
	flag.Parse()

	d := client.NewConsulDiscovery(*basePath, []string{*consulAddr}, nil)
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

## mDNS {#mdns}

### 服务器


### 客户端


## Inprocess {#inpreocess}

### 服务器


### 客户端


