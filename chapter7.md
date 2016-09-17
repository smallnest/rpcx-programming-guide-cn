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
 //Select returns a new client and it also update current client Select(clientCodecFunc ClientCodecFunc, options ...interface{}) (*rpc.Client, error)
 //SetClient set current client SetClient(*Client) SetSelectMode(SelectMode)
 //AllClients returns all Clients AllClients(clientCodecFunc ClientCodecFunc) []*rpc.Client }
```

`Select`从服务列表中根据路由算法选择一个服务来调用，它返回的是一个rpc.Client对象，这个对象建立了对实际选择的服务的连接。
`SetSelectMode`可以用来设置路由算法，路由算法根据一定的规则从服务列表中来选择服务。
`AllClients`返回对所有的服务的rpc.Client slice。
