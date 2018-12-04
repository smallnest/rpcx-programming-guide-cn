# Summary

## Part Ⅰ 开发起步
* [快速起步](part1/quickstart.md)
* [服务端开发示例](part1/server.md)
* [客户端开发示例](part1/client.md)
* [传输](part1/transport.md)
  * [TCP](part1/tcp.md)
  * [HTTP](part1/http.md)
  * [UnixDomain](part1/unixdomain.md)
  * [QUIC](part1/quic.md)
  * [KCP](part1/kcp.md)
* [函数为服务](part1/function.md)
## Part Ⅱ 注册中心
* [注册中心](part2/registry.md)
  * [点对点](part2/peer2peer.md)
  * [点对多](part2/multiple.md)
  * [Zookeeper](part2/zookeeper.md)
  * [Etcd](part2/etcd.md)
  * [Consul](part2/consul.md)
  * [mDNS](part2/mdns.md)
  * [进程内调用](part2/inprocess.md)


## Part Ⅲ 特性
* [编解码](part3/codec.md)
* [失败模式](part3/failmode.md)
  * [Failfast](part3/failfast.md)
  * [Failover](part3/failover.md)
  * [Failtry](part3/failtry.md)
  * [Failbackup](part3/failbackup.md)
* [Fork模式](part3/fork.md)
* [广播模式](part3/broadcast.md)
* [路由](part3/selector.md)
  * [随机选择](part3/random_selector.md)
  * [轮训](part3/roundrobin_selector.md)
  * [一致性哈希](part3/hash_selector.md)
  * [权重](part3/weighted_selector.md)
  * [网络质量优先](part3/ping_selector.md)
  * [地理位置优先](part3/geo_selector.md)
  * [定制路由规则](part3/user_selector.md)
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