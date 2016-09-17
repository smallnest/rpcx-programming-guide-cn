# 客户端开发

不同的注册中心有不同的ClientSelector, rpcx利用ClientSelector配置到注册中心的连接以及客户端的连接，然后根据ClientSelector生成rpcx.Client。


注册中心那一章我们已经介绍了三种ClientSeletor,目前rpcx支持五种ClientSelector:

```go
type ConsulClientSelector

func NewConsulClientSelector(consulAddress string, serviceName string, sessionTimeout time.Duration, sm rpcx.SelectMode, dailTimeout time.Duration) *ConsulClientSelector

func NewEtcdClientSelector(etcdServers []string, basePath string, sessionTimeout time.Duration, sm rpcx.SelectMode, dailTimeout time.Duration) *EtcdClientSelector

func NewMultiClientSelector(servers []*ServerPeer, sm rpcx.SelectMode, dailTimeout time.Duration) *MultiClientSelector

func NewZooKeeperClientSelector(zkServers []string, basePath string, sessionTimeout time.Duration, sm rpcx.SelectMode, dailTimeout time.Duration) *ZooKeeperClientSelector

type DirectClientSelector struct { Network, Address string DialTimeout time.Duration Client *Client }
```

它们都实现了`ClientSelector`接口。
```go 
type ClientSelector interface {
 //Select returns a new client and it also update current client 
 Select(clientCodecFunc ClientCodecFunc, options ...interface{}) (*rpc.Client, error)
 //SetClient set current client
 SetClient(*Client)
 SetSelectMode(SelectMode)
 //AllClients returns all Clients
 AllClients(clientCodecFunc ClientCodecFunc) []*rpc.Client }
```

`Select`从服务列表中根据路由算法选择一个服务来调用，它返回的是一个rpc.Client对象，这个对象建立了对实际选择的服务的连接。
`SetClient`用来建立对当前选择的Client的引用，它用来关联一个rpcx.Client。
`SetSelectMode`可以用来设置路由算法，路由算法根据一定的规则从服务列表中来选择服务。
`AllClients`返回对所有的服务的rpc.Client slice。

所以你可以看到，底层rpcx还是利用官方库`net/rpc`进行通讯的。因此通过`NewClient`得到的rpcx.Client调用方法和官方库类似，`Call`是同步调用，`Go`是异步调用，调用完毕后可以`Close`释放连接。
`Auth`方法可以设置授权验证的信息。

```go 
type Client

func NewClient(s ClientSelector) *Client

func (c *Client) Auth(authorization, tag string) error

func (c *Client) Call(serviceMethod string, args interface{}, reply interface{}) (err error)

func (c *Client) Close() error

func (c *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *rpc.Call) *rpc.Call
```

rpcx.Client的定义如下：
```go
type Client struct {
 ClientSelector ClientSelector
 ClientCodecFunc ClientCodecFunc
 PluginContainer IClientPluginContainer
 FailMode FailMode
 TLSConfig *tls.Config
 Retries int
 //Timeout sets deadline for underlying net.Conns
 Timeout time.Duration
 //Timeout sets readdeadline for underlying net.Conns
 ReadTimeout time.Duration
 //Timeout sets writedeadline for underlying net.Conns 
 WriteTimeout time.Duration
 // contains filtered or unexported fields
 }
```

