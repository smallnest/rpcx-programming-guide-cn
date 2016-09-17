# 客户端路由选择

rpcx面向的是大规模的集群服务，所以同一个服务可能会部署多个节点，这些节点可能在同一个数据中心，也可能在不同的数据中心。对于客户端来说，它的一次调用必然要选择一个节点建立连接并调用，这个选择算法就是路由选择。

rpcx支持多种路由选择算法：

* RandomSelect： 随机选择
* RoundRobin: 轮转的方式
* WeightedRoundRobin: 基于权重的平滑的选择
* ConsistentHash： 快速一致哈希


WeightedRoundRobin是参考Nginx实现的基于权重的轮转的算法，既可以实现权重，也会将请求较为平均的发送给各个服务器。

ConsistentHash选择算法需要用户定义一个Hash算法，客户端根据这个Hash算法计算应该选择哪个服务器, rpcx提供了JumpConsistentHash算法，它可以根据请求参数选择服务器，相同的参数会选择相同的服务器，当然你也可以定制自己的Hash算法以实现不同的业务逻辑。



