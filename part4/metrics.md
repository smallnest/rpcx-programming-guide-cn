# Metrics 插件

**示例:** [metrics](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/metrics)


Metrics 插件使用流行的[go-metrics](github.com/rcrowley/go-metrics) 来计算服务的指标。

它包含多个统计指标:

1. serviceCounter
2. clientMeter
3. "service_"+servicePath+"."+serviceMethod+"_Read_Qps"
4. "service_"+servicePath+"."+serviceMethod+"_Write_Qps"
5. "service_"+servicePath+"."+serviceMethod+"_CallTime"


你可以将metrics输出到graphite中，通过grafana来监控。