# RPCX 起步

[rpcx]((https://github.com/smallnest/rpcx)是一个分布式的服务框架，致力于提供一个产品级的、高性能、透明化的RPC远程服务调用框架。它参考了目前在国内非常流行的Java生态圈的RPC框架Dubbo、Motan等，为Go生态圈提供一个全功能的RPC平台。

随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，分布式服务架构以及流动计算架构势在必行，亟需一个治理系统确保架构有条不紊的演进。

目前，随着网站的规模的扩大，一般会将单体程序逐渐演化为微服务的架构方式，这是目前流行的一种架构模式。

!\[\]\(ch4-microservices-architecture.png\)

即使不是微服务，也会将业务拆分成不同的服务的方式，服务之间互相调用。

那么，如何实现服务\(微服务\)之间的调用的？一般来说最常用的是两种方式: RESTful API和RPC。本书的第一章就介绍了这两种方式的特点和优缺点，那么本书重点介绍的是RPC的方式。

RPCX就是为Go生态圈提供的一个全功能的RPC框架,它参考了国内电商圈流行的RPC框架Dubbo的功能特性，实现了一个高性能的、可容错的，插件式的RPC框架。

它的特点包括：

* 开发简单，基本类似官方的RPC库开发
* 插件式设计，很容易扩展开发
* 可以基于TCP或者HTTP进行通讯，pipelining设计，性能更好R
* 支持纯的Go类型，无需特殊的IDL定义。但是也支持其它的编解码库，如gob、Json、MessagePack、gencode、ProtoBuf
* 支持JSON-RPC和JSON-RPC2，实现跨语言调用
* 支持服务发现，支持多种注册中心，如ZooKeeper、Etcd 和 Consul
* 容错，支持Failover、Failfast、Failtry、Broadcast
* 多种路由和负载均衡方式：Random,RoundRobin, WeightedRoundRobin, consistent hash等
* 支持授权验证方式
* 支持TLS
* 支持超时设计(Dial, Read, Write超时)
* 其它功能，比如限流、日志、监控\(metrics\)等
* 提供一个web管理界面

本章就让我们以两个无注册中心的例子，看看如何利用rpcx进行开发。

## 端对端的例子
首先让我们看一个简单的例子，一个服务器和一个客户端。
这并不是一个常用的应用场景，因为部署的规模太小，其实可以直接官方的库或者其它的RPC框架如gRPC、Thrift就可以实现，当然使用rpcx也很简单，我们就以这个例子，先熟悉一下rpcx的开发。

本书中常用的一个例子就是提供一个乘法的服务，客户端提供两个数，服务器计算这两个数的乘积返回。

### 服务器端开发

首先，和官方库的开发一样，定义相应的数据接口。这个例子中我们都使用默认的配置，包括序列化库Gob:

```go 
type Args struct { A int `msg:"a"` B int `msg:"b"`}

type Reply struct { C int `msg:"c"`}

type Arith int

func (t *Arith) Mul(args *Args, reply *Reply) error { 
    reply.C = args.A * args.B return nil
}
```

`Args`作为传入的参数，它的两个字段`A`、`B`代表两个乘数。
`Reply`的`C`代表返回的结果。
`Mul`就是业务方法，对乘数进行相乘，然后返回结果。


然后注册这个服务启动就可以了：
```go 
func main() { 
    server := rpcx.NewServer() 
    server.RegisterName("Arith", new(Arith)) 
    server.Serve("tcp", "127.0.0.1:8972")
}
```
和官方库rpc以及http一样， rpcx提供了一个缺省的Server,并且在rpcx包下提供了一些便利的方法。但是如果你想使用定制的服务器，你可以使用`rpcx.NewServer`生成一个新的服务器对象。

然后将我们的服务注册上，这样服务器就知道它所提供的服务的一些元数据信息。
`server.Serve`会启动一个TCP服务器，监听本地地址的8972端口。

如果使用缺省的服务器，你可以写如下的代码：
```go 
rpcx.RegisterName("Arith", new(Arith))
rpcx.Serve("tcp", "127.0.0.1:8972")
```

如果你有官方库的开发经验，或者你学习了本书第二章的内容，你会发现它和官方库的开发几乎完全一样，rpcx隐藏了内部处理的细节，让你依然可以很轻易的进行rpc的开发。

### 客户端同步调用
客户端代码也和官方库类似，但是首先它会配置一个`ClientSelector`,根据注册中心的不同，这个ClientSelector具体的实现不同。因为这个例子是简单的端对端的操作，所以我们使用直连的ClientSelector:

```go 
 s := &rpcx.DirectClientSelector{Network: "tcp", Address: "127.0.0.1:8972", DialTimeout: 10 * time.Second} client := rpcx.NewClient(s)
```

它所使用的数据结构和服务端一样。如果客户端和服务器端在一个工程里，所以你可以访问，它们可以共享一套数据结构。但是有些情况下服务方只提供一个说明文档，数据结构还得客户端自己定义：

```go 
type Args struct { A int `msg:"a"` B int `msg:"b"`}

type Reply struct { C int `msg:"c"`}
```

同步调用的代码如下：
```go
args := &Args{7, 8} 
var reply Reply 
err := client.Call("Arith.Mul", args, &reply) 
if err != nil {
 fmt.Printf("error for Arith: %d*%d, %v \n", args.A, args.B, err) } else {
 fmt.Printf("Arith: %d*%d=%d \n", args.A, args.B, reply.C)
}

client.Close()
```

这里的服务的名字必须是 服务器注册的服务名+ "." + 要调用的方法名。
`Call`调用是一个同步的调用。

输出结果为：
```sh
Arith: 7*8=56
```

### 客户端异步调用
客户端异步调用其实是使用`Go`方法，从返回的对象Done字段中可以得到返回的结果信息：

```go 
func main() {
 s := &rpcx.DirectClientSelector{Network: "tcp", Address: "127.0.0.1:8972", DialTimeout: 10 * time.Second}
 client := rpcx.NewClient(s)

 args := &Args{7, 8}
 var reply Reply divCall := client.Go("Arith.Mul", args, &reply, nil)
 replyCall := <-divCall.Done // will be equal to divCall
 if replyCall.Error != nil {
     fmt.Printf("error for Arith: %d*%d, %v \n", args.A, args.B, replyCall.Error)
 } else {
     fmt.Printf("Arith: %d*%d=%d \n", args.A, args.B, reply.C)
 }

 client.Close()}

```


开发起来是不是超简单？

## 多服务调用

我们再看一个复杂点的例子，这个例子中我们部署了两个相同的服务器，这两个服务注册的名字相同，都是计算两个数的乘积。为了区分客户端调用不同的服务，我们故意为一个服务器的乘积放大了十倍，便于我们观察调用的结果。

数据结构如下:
```go 
type Args struct { A int `msg:"a"` B int `msg:"b"`}

type Reply struct { C int `msg:"c"`}

type Arith int

func (t *Arith) Mul(args *Args, reply *Reply) error { reply.C = args.A * args.B return nil}

func (t *Arith) Error(args *Args, reply *Reply) error { panic("ERROR")}

type Arith2 int

func (t *Arith2) Mul(args *Args, reply *Reply) error { reply.C = args.A * args.B * 10 return nil}

func (t *Arith2) Error(args *Args, reply *Reply) error { panic("ERROR")}
```

`Arith2`的Mul方法中我们将计算结果放大了10倍，所以如果传入两个参数7和8,它返回的结果是560,而`Arith`返回56。

### 服务器端的代码
在服务器端我们启动两个服务器，每个服务器都注册了相同名称的一个服务`Arith`,它们分别监听本地的8972和8973端口:
```go 
func main() { 
    server1 := rpcx.NewServer() 
    server1.RegisterName("Arith", new(Arith)) 
    server1.Start("tcp", "127.0.0.1:8972")

    server2 := rpcx.NewServer() 
    server2.RegisterName("Arith", new(Arith2))
    server2.Serve("tcp", "127.0.0.1:8973")
}
```

### 客户端代码
因为我们没有使用注册中心，所以这里客户端需要定义这两个服务器的信息，然后定义路由方式是随机选取(RandomSelect)，调用十次看看服务器返回的结果：

```go
type Args struct { A int `msg:"a"` B int `msg:"b"`}

type Reply struct { C int `msg:"c"`}

func main() {
 server1 := &clientselector.ServerPeer{Network: "tcp", Address: "127.0.0.1:8972"}
 server2 := &clientselector.ServerPeer{Network: "tcp", Address: "127.0.0.1:8973"}

 servers := []*clientselector.ServerPeer{server1, server2}

 s := clientselector.NewMultiClientSelector(servers, rpcx.RandomSelect, 10*time.Second)

 for i := 0; i < 10; i++ {
   callServer(s) 
 }
}

func callServer(s rpcx.ClientSelector) {
 client := rpcx.NewClient(s)

 args := &Args{7, 8}
 var reply Reply err := client.Call("Arith.Mul", args, &reply)
 if err != nil {
   fmt.Printf("error for Arith: %d*%d, %v \n", args.A, args.B, err)
 } else {
   fmt.Printf("Arith: %d*%d=%d \n", args.A, args.B, reply.C)
 }

 client.Close()
}
```


输出结果,可以看到:
```sh
Arith: 7*8=560
Arith: 7*8=56
Arith: 7*8=560
Arith: 7*8=56
Arith: 7*8=56
Arith: 7*8=56
Arith: 7*8=560
Arith: 7*8=56
Arith: 7*8=56
Arith: 7*8=560
```
