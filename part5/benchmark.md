# Benchmark

 Benchmark 的测试代码可以在这里找到：[rpcx-ecosystem/rpcx-benchmark](https://github.com/rpcx-ecosystem/rpcx-benchmark)。

测试使用相同的测试环境，相同的测试数据，相同的测试参数，分别测试了 grpc, rpcx, dubbo, motan, thrift 和 go-micro rpc框架。

基于以前的测试结果, dubbo, motan 和 go-micro 的测试结果不乐观，所以它们最新的测试并没有列在这里，你可以运行测试代码来测试它们。

> 注意: 这个测试是基于io敏感的场景进行测试的， 也就是会让服务`sleep` n秒钟， 对于cpu敏感的测试场景并没有实现。
>      测试假定网络条件很好， 如果是跨数据中心的服务调用，尤其是中美之间这种跨洋调用，服务的主要耗时并不在服务实现上，而是耗在了网络传输上,
> 这不是本测试要测试的场景,并且这个场景已经不太好区分出rpc框架的性能了。


 ## 测试逻辑

使用`protobuf`作为编解码器。proto文件位于 [benchmark.proto](https://github.com/rpcx-ecosystem/rpcx-benchmark/blob/master/rpcx/benchmark.proto):

 ```proto
 syntax = "proto2";
 
 package main;
 
 option optimize_for = SPEED;
 
 
 message BenchmarkMessage {
   required string field1 = 1;
   optional string field9 = 9;
   optional string field18 = 18;
   optional bool field80 = 80 [default=false];
   optional bool field81 = 81 [default=true];
   required int32 field2 = 2;
   required int32 field3 = 3;
   
   ......
 }
 ```

Client 会创建一个 request ,并且对相应的字段进行赋值，最终的request序列化后的大小为 **518** 字节。

Server 接收这个请求，并且将第一个字段的值设置为`OK`,第二个字段设置为`100`,然后把这个对象返回给Client。


测试工具中下面两个参数用来控制并发数和总的请求数。
```
var concurrency = flag.Int("c", 1, "concurrency")
var total = flag.Int("n", 1, "total requests for all clients")
```

## 测试环境

- CPU: Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz, 32 cores
- Memory: 32G
- Go: 1.9.2
- OS: CentOS 7 / 3.10.0-229.el7.x86_64


Client 和 Server 安装在同一台机器上 (忽略网络传输的影响)


## 测试结果

### TPS

5000 并发client的情况下, rpcx 可以达到 176,894 transations/second 吞吐率, 但是 grpc-go 只能达到 105,219 transations/second 的吞吐率

|_concurrency_| RPCX | GRPC-GO|
|----|----|------|
|5000|176894|105219|
|2000|161660|108245|
|1000|148227|111351|
|100|145479|93447|


### 延迟: 平均时间

|_concurrency_| RPCX | GRPC-GO|
|----|----|------|
|5000|27|47|
|2000|12|18|
|1000|6|8|
|100|0|1|


### 延迟: 中位数时间

|_concurrency_| RPCX | GRPC-GO|
|----|----|------|
|5000|3|42|
|2000|7|15|
|1000|5|7|
|100|0|0|