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

`ServeXXX`方法可以以HTTP或者TCP提供服务，还可以提供TLS的设置。
