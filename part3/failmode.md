# 失败模式

在分布式架构中， 如SOA或者微服务架构，你不能担保服务调用如你所预想的一样好。有时候服务会宕机、网络被挖断、网络变慢等，所以你需要容忍这些状况。

rpcx支持四种调用失败模式，用来处理服务调用失败后的处理逻辑， 你可以在创建`XClient`的时候设置它。

`FailMode`的设置仅仅对同步调用有效(`XClient.Call`), 异步调用用，这个参数是无意义的。


## Failfast {#failfast}

**示例:** [failfast](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/failmode/failfast)

在这种模式下， 一旦调用一个节点失败， rpcx立即会返回错误。 注意这个错误不是业务上的 `Error`, 业务上服务端返回的`Error`应该正常返回给客户端，这里的错误可能是网络错误或者服务异常。

## Failover {#failover}

**示例:** [failover](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/failmode/failover)

在这种模式下,  rpcx如果遇到错误，它会尝试调用另外一个节点， 直到服务节点能正常返回信息，或者达到最大的重试次数。 
重试测试`Retries`在参数`Option`中设置， 缺省设置为3。

## Failtry {#failtry}

**示例:** [failtry](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/failmode/failtry)

在这种模式下， rpcx如果调用一个节点的服务出现错误， 它也会尝试，但是还是选择这个节点进行重试， 直到节点正常返回数据或者达到最大重试次数。

## Failbackup {#failbackup}

**示例:** [failbackup](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/failmode/failbackup)

在这种模式下， 如果服务节点在一定的时间内不返回结果， rpcx客户端会发送相同的请求到另外一个节点， 只要这两个节点有一个返回， rpcx就算调用成功。

这个设定的时间配置在 `Option.BackupLatency` 参数中。

这种通过资源换取延迟的方式可以参看 Jeff Dean的文章 [Achieving Rapid Response Times in Large Online Services](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44875.pdf)

> From http://highscalability.com/blog/2012/6/18/google-on-latency-tolerant-systems-making-a-predictable-whol.html
> Backup requests are the idea of sending requests out to multiple replicas, but in a particular way. Here’s the example for a read operation for a distributed file system client:
> 
> send request to first replica
> wait 2 ms, and send to second replica
> servers cancel request on other replica when starting read
> A request could wait in a queue stuck behind an expensive query or a packet could be dropped, so if a reply is not returned quickly other replicas are tried. Responses come back faster if requests sit in multiple queues.
