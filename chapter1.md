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



## 参考文档
1. https://golang.org/pkg/net/rpc/