# 路由

在大型的微服务系统中，我们会为同一个服务部署多个节点， 以便服务可以支持大并发的访问。它们可能部署在同一个数据中心的多个节点，或者多个数据中心中。

那么， 客户端该如何选择一个节点呢？ rpcx通过 `Selector`来实现路由选择， 它就像一个负载均衡器，帮助你选择出一个合适的节点。

rpcx提供了多个路由策略算法，你可以在创建`XClient`来指定。


注意，这里的路由是针对 `ServicePath` 和 `ServiceMethod`的路由。

## 随机 {#random_selector}

**示例:** [random](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/random)

从配置的节点中随机选择一个节点。

最简单，但是有时候单个节点的负载比较重。这是因为随机数只能保证在大量的请求下路由的比较均匀，并不能保证在很短的时间内负载是均匀的。


## 轮询 {#roundrobin_selector}

**示例:** [roundrobin](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/roundrobin)


使用轮询的方式，依次调用节点，能保证每个节点都均匀的被访问。在节点的服务能力都差不多的时候适用。

## WeightedRoundRobin {#weighted_selector}

**示例:** [weighted](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/weighted)


使用Nginx [平滑的基于权重的轮询算法](https://github.com/phusion/nginx/commit/27e94984486058d73157038f7950a0a36ecc6e35)。


比如如果三个节点`a`、`b`、`c`的权重是`{ 5, 1, 1 }`, 这个算法的调用顺序是 `{ a, a, b, a, c, a, a }`,
相比较 `{ c, b, a, a, a, a, a }`, 虽然权重都一样，但是前者更好，不至于在一段时间内将请求都发送给`a`。


## 网络质量优先 {#ping_selector}

**示例:** [ping](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/ping)

首先客户端会基于`ping(ICMP)`探测各个节点的网络质量，越短的ping时间，这个节点的权重也就越高。但是，我们也会保证网络较差的节点也有被调用的机会。


假定`t`是ping的返回时间， 如果超过1秒基本就没有调用机会了:

-  weight=191       if t <= 10
-  weight=201 -t    if 10 < t <=200
-  weight=1         if 200 < t < 1000
-  weight=0         if t >= 1000


## 一致性哈希 {#hash_selector}

**示例:** [hash](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/hash)


使用 [JumpConsistentHash](https://arxiv.org/abs/1406.2294) 选择节点， 相同的servicePath, serviceMethod 和 参数会路由到同一个节点上。 JumpConsistentHash 是一个快速计算一致性哈希的算法，但是有一个缺陷是它不能删除节点，如果删除节点，路由就不准确了，所以在节点有变动的时候它会重新计算一致性哈希。

## 地理位置优先 {#geo_selector}

**示例:** [geo](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/geo)

如果我们希望的是客户端会优先选择离它最新的节点， 比如在同一个机房。
如果客户端在北京， 服务在上海和美国硅谷，那么我们优先选择上海的机房。

它要求服务在注册的时候要设置它所在的地理经纬度。

如果两个服务的节点的经纬度是一样的， rpcx会随机选择一个。

比必须使用下面的方法配置客户端的经纬度信息：
```go
func (c *xClient) ConfigGeoSelector(latitude, longitude float64)
```

## 定制路由规则 {#user_selector}

**示例:** [customized](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/selector/customized)

如果上面内置的路由规则不满足你的需求，你可以参考上面的路由器自定义你自己的路由规则。

曾经有一个网友提到， 如果调用参数的某个字段的值是特殊的值的话，他们会把请求路由到一个指定的机房。这样的需求就要求你自己定义一个路由器，只需实现实现下面的接口：

```go
type Selector interface {
	Select(ctx context.Context, servicePath, serviceMethod string, args interface{}) string
	UpdateServer(servers map[string]string)
}
```


-`Select`: defines the select algorithm.
-`UpdateServer`: clients init the nodes and update if nodes change.
