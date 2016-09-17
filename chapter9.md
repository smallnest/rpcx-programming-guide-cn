# 统计与限流

通过一些额外的插件，我们可以为rpcx实现更多的功能和控制。本章就介绍两个有趣的插件。

## MetricsPlugin
Metrics是一个Java性能统计包，非常的流行。而[go-metrics](github.com/rcrowley/go-metrics)是这个库的实现，rpcx利用这个库进行各种的统计，包括:
* serviceCounter: 注册的服务的个数
* clientMeter: 吞吐率
* service_XXX_Read_Counter: 服务调用次数
* service_XXX_Write_Counter: 服务返回次数
* serice_XXX_CallTime: 服务调用时间

统计数据可以输出到控制台、syslog、graphite、influxdb、librato、stathat等


## RateLimitingPlugin
限流是高并发的情况下保证服务质量常用的一种手段。一个服务器的能力有限的，超过这个限度，我们可以拒绝新的连接，保证服务器还能够正常的运行。
