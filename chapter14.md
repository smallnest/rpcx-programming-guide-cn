# 插件开发
rpcx提供了插件式的开发，你可以在某个或者某些插入点上加入你自己的业务逻辑来扩展RPC框架，事实上注册中心就是一个插件。


## 服务器插入点
服务端提供了以下的插入点：
```go
func (p *ServerPluginContainer) DoPostConnAccept(conn net.Conn) bool

func (p *ServerPluginContainer) DoPostReadRequestBody(body interface{}) error

func (p *ServerPluginContainer) DoPostReadRequestHeader(r *rpc.Request) error

func (p *ServerPluginContainer) DoPostWriteResponse(resp *rpc.Response, body interface{}) error

func (p *ServerPluginContainer) DoPreReadRequestBody(body interface{}) error

func (p *ServerPluginContainer) DoPreReadRequestHeader(r *rpc.Request) error

func (p *ServerPluginContainer) DoPreWriteResponse(resp *rpc.Response, body interface{}) error

func (p *ServerPluginContainer) DoRegister(name string, rcvr interface{}, metadata ...string) error

```