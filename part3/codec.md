# 编解码

**Example:** [broadcast](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/codec/gob)

当前rpcx提供了四种内置的编解码器，你也可以定义你自己的编解码器， 如[Avro](https://github.com/linkedin/goavro)等:


```
// SerializeType defines serialization type of payload.
type SerializeType byte

const (
	// SerializeNone uses raw []byte and don't serialize/deserialize
	SerializeNone SerializeType = iota
	// JSON for payload.
	JSON
	// ProtoBuffer for payload.
	ProtoBuffer
	// MsgPack for payload
	MsgPack
)
```


服务会使用和客户端一样的编解码器，客户端使用JSON, 服务也返回 JSON 格式的数据。 rpcx 默认使用 msgpack 编解码器。

```go
var DefaultOption = Option{
	Retries:        3,
	RPCPath:        share.DefaultRPCPath,
	ConnectTimeout: 10 * time.Second,
	Breaker:        CircuitBreaker,
	SerializeType:  protocol.MsgPack,
	CompressType:   protocol.None,
}
```

你可以设置你的option, 选择你自己的编解码器:

```go
func NewXClient(servicePath string, failMode FailMode, selectMode SelectMode, discovery ServiceDiscovery, option Option) 
```

## SerializeNone

这种编解码器不会对数据进行编解码，并且要求数据是 `[]byte` 类型的数据。


## JSON

JSON是一个通用的数据交换的格式，可以应用在很多语言中。

对性能要求不是非常高的场景，可以使用这种编解码。

## Protobuf

[Protobuf](https://developers.google.com/protocol-buffers/) 是一个高性能的编解码器， 由google出品， 应用在很多项目中。

## MsgPack

[messagepack](https://msgpack.org/index.html) 是另外一种高性能的编解码器， 也是跨语言的编解码器。

## 定制编解码器

这个例子 [gob](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/codec/gob) 使用 gob作为编解码器。