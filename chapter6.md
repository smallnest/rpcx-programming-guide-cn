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

```go
func (s *Server) Auth(fn AuthorizationFunc) error
```





