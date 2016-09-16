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
var addr = flag.String("s", "127.0.0.1:8972", "service address")var zk = flag.String("zk", "127.0.0.1:2181", "zookeeper URL")var n = flag.String("n", "127.0.0.1:2181", "Arith")

func main() { flag.Parse()

 server := rpcx.NewServer() rplugin := &plugin.ZooKeeperRegisterPlugin{ ServiceAddress: "tcp@" + *addr, ZooKeeperServers: []string{*zk}, BasePath: "/rpcx", Metrics: metrics.NewRegistry(), Services: make([]string, 1), UpdateInterval: 10 * time.Second, } rplugin.Start() server.PluginContainer.Add(rplugin) server.RegisterName(*n, new(Arith), "weight=5&state=active") server.Serve("tcp", *addr)}
```
 