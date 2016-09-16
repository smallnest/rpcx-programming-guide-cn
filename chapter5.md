# 服务注册中心
服务注册中心用来实现服务发现和服务的元数据存储。

当前rpcx支持ZooKeeper、Etcd 和 Consul三种注册中心， 其中consul注册中心是实验性的，可能一些特性比如web管理程序不支持。

![](ch5-registry.png)

rpcx会自动将服务的信息比如服务名，监听地址，监听协议，权重等注册到注册中心，同时还会定时的将服务的吞吐率更新到注册中心。

如果服务意外中断或者宕机，注册中心能够监测到这个事件，它会通知客户端这个服务目前不可用，在服务调用的时候不要再选择这个服务器。

客户端初始化的时候会从注册中心得到服务器的列表，然后根据不同的路由选择选择合适的服务器进行服务调用。 同时注册中心还会通知客户端某个服务暂时不可用。

通常客户端会选择一个服务器进行调用，但是在FailMode为broadcast或者forking的时候会进行群发。

下面看看使用不同的注册中心时服务器端和客户端的代码都有什么样的变化。


## ZooKeeper注册中心
### 服务端
```go
var addr = flag.String("s", "127.0.0.1:8972", "service address")
var zk = flag.String("zk", "127.0.0.1:2181", "zookeeper URL")
var n = flag.String("n", "127.0.0.1:2181", "Arith")

func main() { 
 flag.Parse()

 server := rpcx.NewServer()
 rplugin := &plugin.ZooKeeperRegisterPlugin{
  ServiceAddress: "tcp@" + *addr,
  ZooKeeperServers: []string{*zk},
  BasePath: "/rpcx",
  Metrics: metrics.NewRegistry(),
  Services: make([]string, 1),
  UpdateInterval: 10 * time.Second,
 }
 rplugin.Start()
 server.PluginContainer.Add(rplugin)
 server.RegisterName(*n, new(Arith), "weight=5&state=active")
 server.Serve("tcp", *addr)}
```
 
首先，我们必须创建一个`ZooKeeperRegisterPlugin`插件，用来设置zookeeper和服务的基本信息，这里我们还定义了更新metrics的功能，然后启动这个插件。

这个插件开始连接zookeeper，并且检查响应的节点是否存在，如果不存在的话会自动创建这个节点。

这个插件还定义了这个服务要暴露的监听地址和端口。

这里我们使用的协议是`tcp`,你也可以换用`http`。

最后我们需要把这个插件加入到插件容器中。只有加入到插件容器中，我们才能进行下一步的操作，比如注册服务。

剩下的动作和原来的无注册中心的操作一样，这是我们还初始化了这个服务的权重和状态。

看以看到，使用注册中心的时候，**必须首先创建一个响应的注册中心的插件，并加入到注册容器中**。

### 客户端
```go
var zk = flag.String("zk", "127.0.0.1:2181", "zookeeper URL")
var n = flag.String("n", "127.0.0.1:2181", "Arith")

func main() { flag.Parse()

 //basePath = "/rpcx/" + serviceName 
s := clientselector.NewZooKeeperClientSelector([]string{*zk}, "/rpcx/"+*n, 2*time.Minute, rpcx.WeightedRoundRobin, time.Minute) 
client := rpcx.NewClient(s)

 args := &Args{7, 8} var reply Reply

 for i := 0; i < 10; i++ {
  err := client.Call(*n+".Mul", args, &reply)
  if err != nil {
    fmt.Printf("error for "+*n+": %d*%d, %v \n", args.A, args.B, err)
  } else {
   fmt.Printf(*n+": %d*%d=%d \n", args.A, args.B, reply.C)
  }
 }

 client.Close()
}
```

客户端的变化不大，只是将直连的ClientSelector换成了ZooKeeperClientSelector,当然你需要在将zookeeper和服务的基本信息设置到这个对象上，rpcx根据这个信息生成一个rpcx.Client对象。


## Etcd
zookeeper是Java生态圈常用的一个服务发现的软件，而Go生态圈更常用的是etcd。

### 服务器
```go
var addr = flag.String("s", "127.0.0.1:8972", "service address")
var e = flag.String("e", "http://127.0.0.1:2379", "etcd URL")
var n = flag.String("n", "Arith", "Service Name")

func main() {
 flag.Parse()

 server := rpcx.NewServer()
 rplugin := &plugin.EtcdRegisterPlugin{
  ServiceAddress: "tcp@" + *addr,
  EtcdServers: []string{*e},
  BasePath: "/rpcx",
  Metrics: metrics.NewRegistry(),
  Services: make([]string, 1),
  UpdateInterval: time.Minute,
 }
 rplugin.Start() server.PluginContainer.Add(rplugin) server.PluginContainer.Add(plugin.NewMetricsPlugin()) server.RegisterName(*n, new(Arith), "weight=1&m=devops") server.Serve("tcp", *addr)}
```




