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

你可以将实现插入点方法的插件加入到服务器的容器中,或者移除容器。
```go
func (p *ServerPluginContainer) Add(plugin IPlugin) error

func (p *ServerPluginContainer) GetAll() []IPlugin

func (p *ServerPluginContainer) GetByName(pluginName string) IPlugin

func (p *ServerPluginContainer) GetDescription(plugin IPlugin) string

func (p *ServerPluginContainer) GetName(plugin IPlugin) string

func (p *ServerPluginContainer) Remove(pluginName string) error
```

事实上，服务器端的插件点定义了几个接口，你只需要实现一个或者几个接口即可。
```go
//IRegisterPlugin represents register plugin. IRegisterPlugin interface { Register(name string, rcvr interface{}, metadata ...string) error }

 //IPostConnAcceptPlugin represents connection accept plugin. // if returns false, it means subsequent IPostConnAcceptPlugins should not contiune to handle this conn // and this conn has been closed. IPostConnAcceptPlugin interface { HandleConnAccept(net.Conn) bool }

 //IServerCodecPlugin represents . IServerCodecPlugin interface { IPreReadRequestHeaderPlugin IPostReadRequestHeaderPlugin IPreReadRequestBodyPlugin IPostReadRequestBodyPlugin IPreWriteResponsePlugin IPostWriteResponsePlugin }

 //IPreReadRequestHeaderPlugin represents . IPreReadRequestHeaderPlugin interface { PreReadRequestHeader(r *rpc.Request) error }

 //IPostReadRequestHeaderPlugin represents . IPostReadRequestHeaderPlugin interface { PostReadRequestHeader(r *rpc.Request) error }

 //IPreReadRequestBodyPlugin represents . IPreReadRequestBodyPlugin interface { PreReadRequestBody(body interface{}) error }

 //IPostReadRequestBodyPlugin represents . IPostReadRequestBodyPlugin interface { PostReadRequestBody(body interface{}) error }

 //IPreWriteResponsePlugin represents . IPreWriteResponsePlugin interface { PreWriteResponse(*rpc.Response, interface{}) error }

 //IPostWriteResponsePlugin represents . IPostWriteResponsePlugin interface { PostWriteResponse(*rpc.Response, interface{}) error }
```

## 客户端插入点

客户端也提供了一些插入点：
```go
func (p *ClientPluginContainer) DoPostReadResponseBody(body interface{}) error

func (p *ClientPluginContainer) DoPostReadResponseHeader(r *rpc.Response) error

func (p *ClientPluginContainer) DoPostWriteRequest(r *rpc.Request, body interface{}) error

func (p *ClientPluginContainer) DoPreReadResponseBody(body interface{}) error

func (p *ClientPluginContainer) DoPreReadResponseHeader(r *rpc.Response) error

func (p *ClientPluginContainer) DoPreWriteRequest(r *rpc.Request, body interface{}) error
```

你可以将插件加入的插件容器中：
```go
func (p *ClientPluginContainer) Add(plugin IPlugin) error

func (p *ClientPluginContainer) GetAll() []IPlugin

func (p *ClientPluginContainer) GetByName(pluginName string) IPlugin

func (p *ClientPluginContainer) GetDescription(plugin IPlugin) string

func (p *ClientPluginContainer) GetName(plugin IPlugin) string

func (p *ClientPluginContainer) Remove(pluginName string) error
```

它也定义了一些插入点的接口，你只需实现这些接口即可。
```go


```
