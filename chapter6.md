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


```go
func (s *Server) RegisterName(name string, service interface{}, metadata ...string) 
```

```go
func NewServer() *Server
func (s *Server) Address() string
func (s *Server) Auth(fn AuthorizationFunc) error
func (s *Server) Close() error
```



