# 服务器端开发

前面两章已经看到了简单的服务器的开发，接下来的两章我们将了解的服务器和客户端更详细的设置和开发。

服务器提供了几种启动服务器的方法：

```go 
func (s *Server) Serve(network, address string)
func (s *Server) ServeByHTTP(ln net.Listener, rpcPath string)
func (s *Server) ServeHTTP(w http.ResponseWriter, req *http.Request)
func (s *Server) ServeListener(ln net.Listener)
func (s *Server) ServeTLS(network, address string, config *tls.Config)
func (s *Server) Start(network, address string)
func (s *Server) StartTLS(network, address string, config *tls.Config)
```

`ServeXXX`方法的方法会阻塞当前的goroutine，如果不想阻塞当前的goroutine，可以调用`StartXXX`方法。

一些例子：
```go
ln, _ := net.Listen("tcp", "127.0.0.1:0")
go s.ServeByHTTP(ln, "foo")
```

```go
server.Start("tcp", "127.0.0.1:0")
```

`RegisterName`用来注册服务，可以指定服务名和它的元数据。
```go
func (s *Server) RegisterName(name string, service interface{}, metadata ...string) 
```

另外Server还提供其它几个方法。`NewServer`用来创建一个新的Server对象。
```go
func NewServer() *Server
```

`Address`返回Server监听的地址。如果你设置的时候设置端口为0,则Go会选择一个合适的端口号作为监听的端口号，通过这个方法可以返回实际的监听地址和端口。
```go
func (s *Server) Address() string
```

`Close`则关闭监听，停止服务。
```go
func (s *Server) Close() error
```

`Auth`提供一个身份验证的方法，它在你需要实现服务权限设置的时候很有用。
客户端会将一个身份验证的token传给服务器，但是rpcx并不限制你采用何种验证方式，普通的用户名+密码的明文也可以，OAuth2也可以，只要客户端和服务器端协商好一致的验证方式即可。
rpcx会将解析的验证token,服务名称以及额外的信息传给下面的设置的方法`AuthorizationFunc`,你需要实现这个方法。比如通过OAuth2的方式，解析出access_token,然后检查这个access_toekn是否对这个服务有授权。
```go
func (s *Server) Auth(fn AuthorizationFunc) error
```

我们可以看一个例子,服务器的代码如下：
```go 
func main() {
 server := rpcx.NewServer()

 fn := func(p *rpcx.AuthorizationAndServiceMethod) error {
  if p.Authorization != "0b79bab50daca910b000d4f1a2b675d604257e42" || p.Tag != "Bearer" {
   fmt.Printf("error: wrong Authorization: %s, %s\n", p.Authorization, p.Tag)
   return errors.New("Authorization failed ")
  }

  fmt.Printf("Authorization success: %+v\n", p)
  return nil
 }

 server.Auth(fn)

 server.RegisterName("Arith", new(Arith)) server.Serve("tcp", "127.0.0.1:8972")}
```
这个简单的例子演示了只有用户使用"Bear 0b79bab50daca910b000d4f1a2b675d604257e42" acces_token 访问时才提供服务，否则报错。

我们可以看一下客户端如何设置这个access_token:
```go 
func main() {
 s := &rpcx.DirectClientSelector{Network: "tcp", Address: "127.0.0.1:8972", DialTimeout: 10 * time.Second}
 client := rpcx.NewClient(s)

 //add Authorization info
 err := client.Auth("0b79bab50daca910b000d4f1a2b675d604257e42_ABC", "Bearer")
 if err != nil {
  fmt.Printf("can't add auth plugin: %#v\n", err)
 }

 args := &Args{7, 8}
 var reply Reply
 err = client.Call("Arith.Mul", args, &reply)
 if err != nil {
  fmt.Printf("error for Arith: %d*%d, %v \n", args.A, args.B, err) 
 } else {
  fmt.Printf("Arith: %d*%d=%d \n", args.A, args.B, reply.C)
 }

 client.Close()
}
```
客户端很简单，调用`Auth`方法设置access_token和token_type(optional)即可。


rpcx创建了一个缺省的`Server`，并提供了一些辅助方法来暴露Server的方法，因此你也可以直接调用`rpcx.XXX`方法来调用缺省的Server的方法。


`Server`是一个struct类型，它还包含一些有用的字段：
```go 
type Server struct {
 ServerCodecFunc ServerCodecFunc //PluginContainer must be configured before starting and Register plugins must be configured before invoking RegisterName method
 PluginContainer IServerPluginContainer
 //Metadata describes extra info about this service, for example, weight, active status Metadata string
 Timeout time.Duration
 ReadTimeout time.Duration
 WriteTimeout time.Duration
 // contains filtered or unexported fields
 }
```

`ServerCodecFunc`





