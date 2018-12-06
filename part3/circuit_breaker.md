# 断路器模式

在一个节点失败的情况下，断路器可以避免这个错误影响其他服务，以免出现雪崩的情况。查看断路器的详细介绍: [Pattern: Circuit Breaker](http://microservices.io/patterns/reliability/circuit-breaker.html).

客户端通过断路器调用服务， 一旦连续的错误达到一个阈值，断路器就会断开进行保护，这个时候如果还调用这个节点的话，直接就返回错误。等一定的时间，断路器会处于半开的状态，允许一定数量的请求尝试发送这个节点，如果正常访问，断路器就处于全开的状态，否则又进入短路的状态。


Rpcx 定义了 `Breaker` 接口， 你可以自己实现复杂情况的断路器。

```go
type Breaker interface {
	Call(func() error, time.Duration) error
}
```

Rpcx 提供了一个简单的断路器 `ConsecCircuitBreaker`， 它在连续失败一定次数后就会断开，再经过一段时间后打开。
你可以将你的断路器设置到 `Option.Breaker`中。