# 性能比较

通过和Dubbo、Motan、Thrift、gRPC的性能比较，rpcx目前是性能最好的rpc框架。

本文通过一个统一的服务，测试这四种框架实现的完整的服务器端和客户端的性能。这个服务传递的消息体有一个protobuf文件定义：

```protosyntax = "proto2";

package main;

option optimize_for = SPEED;

message BenchmarkMessage { required string field1 = 1; optional string field9 = 9; optional string field18 = 18; optional bool field80 = 80 [default=false]; optional bool field81 = 81 [default=true]; required int32 field2 = 2; required int32 field3 = 3; optional int32 field280 = 280; optional int32 field6 = 6 [default=0]; optional int64 field22 = 22; optional string field4 = 4; repeated fixed64 field5 = 5; optional bool field59 = 59 [default=false]; optional string field7 = 7; optional int32 field16 = 16; optional int32 field130 = 130 [default=0]; optional bool field12 = 12 [default=true]; optional bool field17 = 17 [default=true]; optional bool field13 = 13 [default=true]; optional bool field14 = 14 [default=true]; optional int32 field104 = 104 [default=0]; optional int32 field100 = 100 [default=0]; optional int32 field101 = 101 [default=0]; optional string field102 = 102; optional string field103 = 103; optional int32 field29 = 29 [default=0]; optional bool field30 = 30 [default=false]; optional int32 field60 = 60 [default=-1]; optional int32 field271 = 271 [default=-1]; optional int32 field272 = 272 [default=-1]; optional int32 field150 = 150; optional int32 field23 = 23 [default=0]; optional bool field24 = 24 [default=false]; optional int32 field25 = 25 [default=0]; optional bool field78 = 78; optional int32 field67 = 67 [default=0]; optional int32 field68 = 68; optional int32 field128 = 128 [default=0]; optional string field129 = 129 [default="xxxxxxxxxxxxxxxxxxxxx"]; optional int32 field131 = 131 [default=0];}```

相应的Thrift定义文件为```thriftnamespace java com.colobu.thrift

struct BenchmarkMessage{ 1: string field1, 2: i32 field2, 3: i32 field3, 4: string field4, 5: i64 field5, 6: i32 field6, 7: string field7, 9: string field9, 12: bool field12, 13: bool field13, 14: bool field14, 16: i32 field16, 17: bool field17, 18: string field18, 22: i64 field22, 23: i32 field23, 24: bool field24, 25: i32 field25, 29: i32 field29, 30: bool field30, 59: bool field59, 60: i32 field60, 67: i32 field67, 68: i32 field68, 78: bool field78, 80: bool field80, 81: bool field81, 100: i32 field100, 101: i32 field101, 102: string field102, 103: string field103, 104: i32 field104, 128: i32 field128, 129: string field129, 130: i32 field130, 131: i32 field131, 150: i32 field150, 271: i32 field271, 272: i32 field272, 280: i32 field280,}

service Greeter {

 BenchmarkMessage say(1:BenchmarkMessage name);

}```

事实上这个文件摘自gRPC项目的测试用例，使用反射为每个字段赋值，使用protobuf序列化后的大小为 581 个字节左右。因为Dubbo和Motan缺省不支持Protobuf,所以序列化和反序列化是在代码中手工实现的。

服务很简单：

```protoservice Hello { // Sends a greeting rpc Say (BenchmarkMessage) returns (BenchmarkMessage) {}}```

接收一个BenchmarkMessage，更改它前两个字段的值为"OK" 和 100，这样客户端得到服务的结果后能够根据结果判断服务是否正常的执行。Dubbo的测试代码改自 [dubo-demo](https://github.com/alibaba/dubbo/tree/master/dubbo-demo),Motan的测试代码改自 [motan-demo](https://github.com/weibocom/motan/tree/master/motan-demo)。rpcx和gRPC的测试代码在 [rpcx benchmark](https://github.com/smallnest/rpcx/tree/master/_benchmarks)。Thrift使用Java进行测试。

正如左耳朵耗子对Dubbo批评一样，Dubbo官方的测试不正规 ([性能测试应该怎么做？](http://coolshell.cn/articles/17381.html))。本文测试将用吞吐率、相应时间平均值、响应时间中位数、响应时间最大值进行比较(响应时间最小值都为0，不必比较)，当然最好以Top Percentile的指标进行比较，但是我没有找到Go语言的很好的统计这个库，所以暂时比较中位数。另外测试中服务的成功率都是100%。

测试是在两台机器上执行的，一台机器做服务器，一台机器做客户端。

两台机器的配置都是一样的，比较老的服务器：

- **CPU**: Intel(R) Xeon(R) CPU E5-2620 v2 @ 2.10GHz, 24 cores- **Memory**: 16G- **OS**: Linux 2.6.32-358.el6.x86_64, CentOS 6.4- **Go**: 1.7- **Java**: 1.8- **Dubbo**: 2.5.4-SNAPSHOT (2016-09-05)- **Motan**: 0.2.2-SNAPSHOT (2016-09-05)- **gRPC**: 1.0.0- **rpcx**: 2016-09-05- **thrift**: 0.9.3 (java)

分别在client并发数为100、500、1000、2000 和 5000的情况下测试，记录吞吐率(每秒调用次数, Throughput)、响应时间(Latency) 、成功率。(更精确的测试还应该记录CPU使用率、内存使用、网络带宽、IO等，本文中未做比较)

首先看在四种并发下各RPC框架的吞吐率：![吞吐率](throughput.png)rpcx的性能遥遥领先，并且其它三种框架在并发client很大的情况下吞吐率会下降。thrift比rpcx性能差一点，但是还不错，远好于gRPC,dubbo和motan,但是随着client的增多，性能也下降的很厉害，在client较少的情况下吞吐率挺好。

在这四种并发的情况下平均响应：![平均响应时间](mean-latency.png)这个和吞吐率的表现是一致的，还是rpcx最好，平均响应时间小于30ms, Dubbo在并发client多的情况下响应时间很长。我们知道，在微服务流行的今天，一个单一的RPC的服务可能会被不同系统所调用，这些不同的系统会创建不同的client。如果调用的系统很多，就有可能创建很多的client。这里统计的是这些client总的吞吐率和总的平均时间。

平均响应时间可能掩盖一些真相，尤其是当响应时间的分布不是那么平均，所以我们还可以关注另外一个指标，就是中位数。这里的中位数指小于这个数值的测试数和大于这个数值的测试数相等。![响应时间中位数](median-latency.png)gRPC框架的表现最好。

另外一个就是比较一下最长的响应时间，看看极端情况下各框架的表现：![最大响应时间](max-latency.png)rpcx的最大响应时间都小于1秒，Motan的表现也不错，都小于2秒，其它两个框架表现不是太好。

本文以一个相同的测试case测试了四种RPC框架的性能，得到了这四种框架在不同的并发条件下的性能表现。期望读者能提出宝贵的意见，以便完善这个测试，并能增加更多的RPC框架的测试。
