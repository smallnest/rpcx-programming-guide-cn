# Summary

## Part Ⅰ 开发起步
* [快速起步](part1/quickstart.md)
* [服务端开发示例](part1/server.md)
* [客户端开发示例](part1/client.md)
* [传输](part1/transport.md)
  * [TCP](part1/transport.md#tcp)
  * [HTTP](part1/transport.md#HTTP)
  * [UnixDomain](part1/transport.md#unixdomain)
  * [QUIC](part1/transport.md#quic)
  * [KCP](part1/transport.md#kcp)
* [函数为服务](part1/function.md)

## Part Ⅱ 注册中心
* [注册中心](part2/registry.md)
  * [点对点](part2/registry.md#peer2peer)
  * [点对多](part2/registry.md#multiple)
  * [Zookeeper](part2/registry.md#zookeeper)
  * [Etcd](part2/registry.md#etcd)
  * [Consul](part2/registry.md#consul)
  * [mDNS](part2/registry.md#mdns)
  * [进程内调用](part2/registry.md#inprocess)


## Part Ⅲ 特性
* [编解码](part3/codec.md)
* [失败模式](part3/failmode.md)
  * [Failfast](part3/failmode.md#failfast)
  * [Failover](part3/failmode.md#failover)
  * [Failtry](part3/failmode.md#failtry)
  * [Failbackup](part3/failmode.md#failbackup)
* [Fork模式](part3/fork.md)
* [广播模式](part3/broadcast.md)
* [路由](part3/selector.md)
  * [随机选择](part3/selector.md#random_selector)
  * [轮训](part3/selector.md#roundrobin_selector)
  * [一致性哈希](part3/selector.md#hash_selector)
  * [权重](part3/selector.md#weighted_selector)
  * [网络质量优先](part3/selector.md#ping_selector)
  * [地理位置优先](part3/selector.md#geo_selector)
  * [定制路由规则](part3/selector.md#user_selector)
* [超时](part3/timeout.md)
* [元数据](part3/metadata.md)
* [心跳](part3/heartbeat.md)
* [分组](part3/group.md)
* [服务状态](part3/state.md)
* [断路器](part3/circuit_breaker.md)


## Part Ⅳ 插件
* [Metrics](part4/metrics.md)
* [限流](part4/rate-limiting.md)
* [别名](part4/alias.md)
* [身份认证](part4/auth.md)
* [插件开发](part4/plugin-dev.md)

## Part Ⅴ 其它
* [Benchmark](part5/benchmark.md)
* [UI管理工具](part5/ui.md)
* [协议详解](part5/protocol.md)

## Part Ⅵ 网关
* [Gateway](part6/gateway.md)
* [HTTP 方式调用](part6/http_invoke.md)
* [双向通讯](part6/bidirectional.md)