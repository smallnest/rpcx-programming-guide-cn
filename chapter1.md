# 官方RPC库

Go官方提供了一个［RPC库］(https://golang.org/pkg/net/rpc/): `net/rpc`。

包rpc提供了通过网络访问一个对象的方法的能力。
服务器需要注册对象， 通过对象的类型名暴露这个服务。注册后这个对象的输出方法就可以远程调用，这个库封装了底层传输的细节，包括序列化。
服务器可以注册多个不同类型的对象，但是注册相同类型的多个对象的时候回出错。

同时，如果对象的方法要能远程访问，它们必须满足一定的条件，否则这个对象的方法回呗忽略。

这些条件是：

- 方法的类型是可输出的 (the method's type is exported)
- 方法本身也是可输出的 （the method is exported）
- 方法必须由两个参数，必须是输出类型或者是内建类型 (the method has two arguments, both exported or builtin types)
- 方法的第二个参数是指针类型 (the method's second argument is a pointer)
- 方法返回类型为 error (the method has return type error)

所以一个输出方法的格式如下：
```go 
func (t *T) MethodName(argType T1, replyType *T2) error
```

这里的`T`、`T1`、`T2`能够被`encoding/gob`序列化，即使使用其它的序列化框架，将来这个需求可能回被弱化。

这个方法的第一个参数代表调用者(client)提供的参数，
第二个参数代表要返回给调用者的计算结果，
方法的返回值如果不为空， 那么它作为一个字符串返回给调用者。
如果返回error，则reply参数不会返回给调用者。

服务器通过调用`ServeConn`在一个连接上处理请求，更典型地， 它可以创建一个network listener然后accept请求。
对于HTTP listener来说，可以调用 `HandleHTTP` 和 `http.Serve`。细节会在下面介绍。

客户端可以调用`Dial`和`DialHTTP`建立连接。 客户端有两个方法调用服务: `Call` 和 `Go`，可以同步地或者异步地调用服务。
当然，调用的时候，需要吧服务名、方法名和参数传递给服务器。异步方法调用`Go` 通过 `Done` channel通知调用结果返回。

除非显示的设置`codec`,否则这个库默认使用包`encoding/gob`作为序列化框架。

## 简单例子
首选介绍一个简单的例子。

这个例子中提供了对两个数相乘和相除的两个方法。

第一步你需要定义传入参数和返回参数的数据结构：
```go
package server

type Args struct {
	A, B int
}

type Quotient struct {
	Quo, Rem int
}
```

第二步定义一个服务对象，这个服务对象可以很简单， 比如类型是`int`或者是`interface{}`,重要的是它输出的方法。
这里我们定义一个算术类型`Arith`，其实它是一个int类型，但是这个int的值我们在后面方法的实现中也没用到，所以它基本上就起一个辅助的作用。
```
type Arith int
```

第三步实现这个类型的两个方法， 乘法和除法：
```go
func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		return errors.New("divide by zero")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}
```

目前为止，我们的准备工作已经完成，喝口茶继续下面的步骤。

第四步实现RPC服务器:
```go
arith := new(Arith)
rpc.Register(arith)
rpc.HandleHTTP()
l, e := net.Listen("tcp", ":1234")
if e != nil {
	log.Fatal("listen error:", e)
}
go http.Serve(l, nil)
```

这里我们生成了一个Arith对象，并使用`rpc.Register`注册这个服务，然后通过HTTP暴露出来。

客户端可以看到服务`Arith`以及它的两个方法`Arith.Multiply`和`Arith.Divide`。

第五步创建一个客户端，建立客户端和服务器端的连接:
```go
client, err := rpc.DialHTTP("tcp", serverAddress + ":1234")
if err != nil {
	log.Fatal("dialing:", err)
}
```

然后客户端就可以进行远程调用了。比如同步的方式：
```go
args := &server.Args{7,8}
var reply int
err = client.Call("Arith.Multiply", args, &reply)
if err != nil {
	log.Fatal("arith error:", err)
}
fmt.Printf("Arith: %d*%d=%d", args.A, args.B, reply)
```

或者异步的方式：
```go
quotient := new(Quotient)
divCall := client.Go("Arith.Divide", args, quotient, nil)
replyCall := <-divCall.Done	// will be equal to divCall
// check errors, print, etc.
```

## 服务器代码分析
首先， ｀net/rpc｀定义了一个缺省的Server,所以Server的很多方法你可以直接调用，这对于一个简单的Server的实现更方便，但是你如果需要配置不同的Server，
比如不同的监听地址或端口，就需要自己生成Server:
``` go 
var DefaultServer = NewServer()
```

Server有多种Socket监听的方式:
```go 
    func (server *Server) Accept(lis net.Listener)
    func (server *Server) HandleHTTP(rpcPath, debugPath string)
    func (server *Server) ServeCodec(codec ServerCodec)
    func (server *Server) ServeConn(conn io.ReadWriteCloser)
    func (server *Server) ServeHTTP(w http.ResponseWriter, req *http.Request)
    func (server *Server) ServeRequest(codec ServerCodec) error
```

其中, `ServeHTTP` 实现了处理 http请求的业务逻辑， 它首先处理http的 `CONNECT`请求， 接收后就Hijacker这个连接conn， 然后调用`ServeConn`在这个连接上　处理这个客户端的请求。
它其实是实现了 `http.Handler`接口，我们一般不直接调用这个方法。
｀Server.HandleHTTP｀设置rpc的上下文路径，｀rpc.HandleHTTP｀使用默认的上下文路径｀DefaultRPCPath｀、 `DefaultDebugPath`。
这样，当你启动一个http server的时候 ｀http.ListenAndServe｀，上面设置的上下文将用作RPC传输，这个上下文的请求会教给`ServeHTTP`来处理。

以上是RPC over http的实现，可以看出 `net/rpc`只是利用 http CONNECT建立连接，这和普通的 RESTful api还是不一样的。

｀Accept｀用来处理一个监听器，一直在监听客户端的连接，一旦监听器接收了一个连接，则还是交给`ServeConn`在另外一个goroutine中去处理：
```go 
func (server *Server) Accept(lis net.Listener) {
	for {
		conn, err := lis.Accept()
		if err != nil {
			log.Print("rpc.Serve: accept:", err.Error())
			return
		}
		go server.ServeConn(conn)
	}
}
```

可以看出，很重要的一个方法就是`ServeConn`:
```go 
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	buf := bufio.NewWriter(conn)
	srv := &gobServerCodec{
		rwc:    conn,
		dec:    gob.NewDecoder(conn),
		enc:    gob.NewEncoder(buf),
		encBuf: buf,
	}
	server.ServeCodec(srv)
}
```

连接其实是交给一个`ServerCodec`去处理，这里默认使用`gobServerCodec`去处理，这是一个未输出默认的编解码器，你可以使用其它的编解码器，我们下面再介绍，
这里我们可以看看`ServeCodec`是怎么实现的：
```go 
func (server *Server) ServeCodec(codec ServerCodec) {
	sending := new(sync.Mutex)
	for {
		service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
		if err != nil {
			if debugLog && err != io.EOF {
				log.Println("rpc:", err)
			}
			if !keepReading {
				break
			}
			// send a response if we actually managed to read a header.
			if req != nil {
				server.sendResponse(sending, req, invalidRequest, codec, err.Error())
				server.freeRequest(req)
			}
			continue
		}
		go service.call(server, sending, mtype, req, argv, replyv, codec)
	}
	codec.Close()
}
```

它其实一直从连接中读取请求，然后调用`go service.call`在另外的goroutine中处理服务调用。

我们从中可以学到：
1. 对象重用。 Request和Response都是可重用的，通过Lock处理竞争。这在大并发的情况下很有效。 有兴趣的读者可以参考[fasthttp](https://github.com/valyala/fasthttp)的实现。
2. 使用了大量的goroutine。 和Java中的线程不同，你可以创建非常多的goroutine, 并发处理非常好。 如果使用一定数量的goutine作为worker池去处理这个case，可能还会有些性能的提升，但是更复杂了。使用goroutine已经获得了非常好的性能。
3. 业务处理是异步的, 服务的执行不会阻塞其它消息的读取。

注意一个codec实例必然和一个connnection相关，因为它需要从connection中读取request和发送response。

go的rpc官方库的消息(request和response)的定义很简单， 就是消息头(header)+内容体(body)。

请求的消息头的定义如下，包括服务的名称和序列号：
```go
type Request struct {
        ServiceMethod string // format: "Service.Method"
        Seq           uint64 // sequence number chosen by client
        // contains filtered or unexported fields
}
```
消息体就是传入的参数。

返回的消息头的定义如下：
```go 
type Response struct {
        ServiceMethod string // echoes that of the Request
        Seq           uint64 // echoes that of the request
        Error         string // error, if any.
        // contains filtered or unexported fields
}
```

消息体是reply类型的序列化后的值。


Server还提供了两个注册服务的方法:
```go 
    func (server *Server) Register(rcvr interface{}) error
    func (server *Server) RegisterName(name string, rcvr interface{}) error
```

第二个方法为服务起一个别名，否则服务名已它的类型命名,它们俩底层调用`register`进行服务的注册。
```go
func (server *Server) register(rcvr interface{}, name string, useName bool) error
```

受限于Go语言的特点， 我们不可能在接到客户端的请求的时候，根据反射动态的创建一个对象，就是Java那样， 
因此在Go语言中，我们需要预先创建一个服务map这是在编译的时候完成的:
```go 
server.serviceMap = make(map[string]*service)
```

同时每个服务还有一个方法map: map[string]*methodType,通过suitableMethods建立:

```go 
func suitableMethods(typ reflect.Type, reportErr bool) map[string]*methodType
```

这样rpc在读取请求header，通过查找这两个map，就可以得到要调用的服务及它的对应方法了。

方法的调用:
```
func (s *service) call(server *Server, sending *sync.Mutex, mtype *methodType, req *Request, argv, replyv reflect.Value, codec ServerCodec) {
	mtype.Lock()
	mtype.numCalls++
	mtype.Unlock()
	function := mtype.method.Func
	// Invoke the method, providing a new value for the reply.
	returnValues := function.Call([]reflect.Value{s.rcvr, argv, replyv})
	// The return value for the method is an error.
	errInter := returnValues[0].Interface()
	errmsg := ""
	if errInter != nil {
		errmsg = errInter.(error).Error()
	}
	server.sendResponse(sending, req, replyv.Interface(), codec, errmsg)
	server.freeRequest(req)
}
```





## 客户端代码分析
客户端要建立和服务器的连接，可以有以下几种方式：

```go 
  	func Dial(network, address string) (*Client, error)
    func DialHTTP(network, address string) (*Client, error)
    func DialHTTPPath(network, address, path string) (*Client, error)
    func NewClient(conn io.ReadWriteCloser) *Client
    func NewClientWithCodec(codec ClientCodec) *Client
```
`DialHTTP` 和 `DialHTTPPath`是通过HTTP的方式和服务器建立连接，他俩的区别之在于是否设置上下文路径:
```go 
func DialHTTPPath(network, address, path string) (*Client, error) {
	var err error
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}
	io.WriteString(conn, "CONNECT "+path+" HTTP/1.0\n\n")

	// Require successful HTTP response
	// before switching to RPC protocol.
	resp, err := http.ReadResponse(bufio.NewReader(conn), &http.Request{Method: "CONNECT"})
	if err == nil && resp.Status == connected {
		return NewClient(conn), nil
	}
	if err == nil {
		err = errors.New("unexpected HTTP response: " + resp.Status)
	}
	conn.Close()
	return nil, &net.OpError{
		Op:   "dial-http",
		Net:  network + " " + address,
		Addr: nil,
		Err:  err,
	}
}
```

首先发送 `CONNECT` 请求，如果连接成功则通过`NewClient(conn)`创建client。

而`Dial`则通过TCP直接连接服务器：
```go 
func Dial(network, address string) (*Client, error) {
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return NewClient(conn), nil
}
```

根据服务是over HTTP还是 over TCP选择合适的连接方式。

`NewClient`则创建一个缺省codec为glob序列化库的客户端:
```go 
func NewClient(conn io.ReadWriteCloser) *Client {
	encBuf := bufio.NewWriter(conn)
	client := &gobClientCodec{conn, gob.NewDecoder(conn), gob.NewEncoder(encBuf), encBuf}
	return NewClientWithCodec(client)
}
```

如果你想用其它的序列化库，你可以调用`NewClientWithCodec`方法<:></:>
```go 
func NewClientWithCodec(codec ClientCodec) *Client {
	client := &Client{
		codec:   codec,
		pending: make(map[uint64]*Call),
	}
	go client.input()
	return client
}
```

重要的是`input`方法，它已一个死循环的方式不断地从连接中读取response,然后调用map中读取等待的Call.Done channel通知完成。

消息的结构和服务器一致，都是Header+Body的方式。

客户端的调用有两个方法: `Go` 和 `Call`。 `Go`方法是异步的，它返回一个 Call指针对象， 它的Done是一个channel，如果服务返回， 
Done就可以得到返回的对象(实际是Call对象，包含Reply和error信息)。 `Go`是同步的方式调用，它实际是调用`Call`实现的，
我们可以看看它是怎么实现的，可以了解一下异步变同步的方式：
```go 
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error {
	call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
	return call.Error
}
```
从一个Channel中读取对象会被阻塞住，直到有对象可以读取，这种实现很简单，也很方便。


其实从服务器端的代码和客户端的代码实现我们还可以学到锁Lock的一种实用方式，也就是尽快的释放锁，而不是`defer mu.Unlock`直到函数执行到最后才释放，那样锁占用的时间太长了。


## codec／序列化框架

前面我们介绍了rpc框架默认使用gob序列化库，很多情况下我们追求更好的效率的情况下，或者追求更通用的序列化格式，我们可能采用其它的序列化方式， 比如protobuf, json, xml等。

gob序列化库有个要求，就是对于接口类型的值，你需要注册具体的实现类型：
```go 
func Register(value interface{})
func RegisterName(name string, value interface{})
```

初次使用rpc的人容易犯这个错误，导致序列化不成功。

Go官方库实现了[JSON-RPC 1.0](http://json-rpc.org/wiki/specification)。JSON-RPC是一个通过JSON格式进行消息传输的RPC规范，因此可以进行跨语言的调用。
Go的`net/rpc/jsonrpc`库可以将JSON-RPC的请求转换成自己内部的格式，比如request header的处理：
```go 
func (c *serverCodec) ReadRequestHeader(r *rpc.Request) error {
	c.req.reset()
	if err := c.dec.Decode(&c.req); err != nil {
		return err
	}
	r.ServiceMethod = c.req.Method
	c.mutex.Lock()
	c.seq++
	c.pending[c.seq] = c.req.Id
	c.req.Id = nil
	r.Seq = c.seq
	c.mutex.Unlock()

	return nil
}
```

JSON-RPC 2.0官方库布支持，但是有第三方开发者提供了实现，比如:
* https://github.com/powerman/rpc-codec
* https://github.com/dwlnetnl/generpc

一些其它的codec如 [bsonrpc](https://godoc.org/github.com/skynetservices/skynet/rpc/bsonrpc)、[messagepack](https://godoc.org/github.com/hashicorp/net-rpc-msgpackrpc)、[protobuf](https://github.com/mars9/codec)等。
如果你使用其它特定的序列化框架，你可以参照这些实现来写一个你自己的rpc codec。


关于Go序列化库的性能的比较你可以参考 [gosercomp](https://github.com/smallnest/gosercomp)。



## 其它
有一个提案 [deprecate net/rpc](https://github.com/golang/go/issues/16844)：

> The package has outstanding bugs that are hard to fix, and cannot support TLS without major work. So although it has a nice API and allows one to use native Go types without an IDL, it should probably be retired.
>
>The proposal is to freeze the package, retire the many bugs filed against it, and add documentation indicating that it is frozen and that suggests alternatives such as GRPC.

但我认为net/rpc的设计相当的优秀，性能超好，如果不继续开发就太可惜了。提案中提到的一些bug和TLS并不是不能修复，可能Go team缺乏相应的资源，或者开发者兴趣不在这里而已。我相信这个提案有很大的反对意见。

目前看来 ｀gRPC｀ 的性能远远逊于 `net/rpc`, 不仅仅是吞吐率，还包括CPU的占有率。


## 参考文档
1. https://golang.org/pkg/net/rpc/
2. https://golang.org/pkg/encoding/gob/
3. https://golang.org/pkg/net/rpc/jsonrpc/
4. https://github.com/golang/go/issues/16844