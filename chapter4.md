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
* Other: metrics、log.
* Authorization.

