# 心跳

**示例:** [heartbeat](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/heartbeat)

你可以设置自动的心跳来保持连接不断掉。
rpcx会自动处理心跳(事实上它直接就丢弃了心跳)。

客户端需要启用心跳选项，并且设置心跳间隔：

```go
option := client.DefaultOption
option.Heartbeat = true
option.HeartbeatInterval = time.Second
```