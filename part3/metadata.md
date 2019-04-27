# 元数据

**示例:** [metadata](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/metadata)

客户端和服务器端可以互相传递元数据。

元数据不是服务请求和服务响应的业务数据，而是一些辅助性的数据。

元数据是一个键值队的列表，键和值都是字符串， 类似 `http.Header`。


## Client

如果你想在客户端传给服务器元数据， 你 **必须** 在上下文中设置 `share.ReqMetaDataKey`。

如果你想在客户端读取客户端的数据， 你 **必须** 在上下文中设置 `share.ResMetaDataKey`。

```go
// client.go
reply := &example.Reply{}
ctx := context.WithValue(context.Background(), share.ReqMetaDataKey, map[string]string{"aaa": "from client"})
ctx = context.WithValue(ctx, share.ResMetaDataKey, make(map[string]string))
err := xclient.Call(ctx, "Mul", args, reply)
```

## Server

服务器可以从上下文读取`share.ReqMetaDataKey` 和 `share.ResMetaDataKey`:

```go
// server.go
reqMeta := ctx.Value(share.ReqMetaDataKey).(map[string]string)
resMeta := ctx.Value(share.ResMetaDataKey).(map[string]string)
```