# 客户端路由选择

rpcx面向的是大规模的集群服务，所以同一个服务可能会部署多个节点，这些节点可能在同一个数据中心，也可能在不同的数据中心。对于客户端来说，它的一次调用必然要选择一个节点建立连接并调用，这个选择算法就是路由选择。

rpcx支持多种路由选择算法：

* RandomSelect SelectMode = iota
* RoundRobin
* WeightedRoundRobin
* LeastActive
* ConsistentHash