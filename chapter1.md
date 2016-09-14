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
客户端要建立和服务器的连接，可以


## codec／序列化框架



## 参考文档
1. https://golang.org/pkg/net/rpc/