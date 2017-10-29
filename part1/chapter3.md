# 其它 Go RPC 库介绍

当然，其它的一些 RPC框架也有提供了Go的绑定，知名的比如[Thrift](https://thrift.apache.org)。

## Thrift

2007年开源，2008看5月进入Apache孵化器，2010年10月成为Apache的顶级项目。

Thrift是一种接口描述语言和二进制通讯协议，它被用来定义和创建跨语言的服务。
它被当作一个远程过程调用（RPC）框架来使用，是由Facebook为“大规模跨语言服务开发”而开发的。
它通过一个代码生成引擎联合了一个软件栈，来创建不同程度的、无缝的跨平台高效服务，可以使用C#、C++（基于POSIX兼容系统）、Cappuccino、Cocoa、Delphi、Erlang、Go、Haskell、Java、Node.js、OCaml、Perl、PHP、Python、Ruby和Smalltalk编程语言开发。
2007由Facebook开源，2008年5月进入Apache孵化器， 2010年10月成为Apache的顶级项目。 

Thrift包含一套完整的栈来创建客户端和服务端程序。[7]顶层部分是由Thrift定义生成的代码。而服务则由这个文件客户端和处理器代码生成。在生成的代码里会创建不同于内建类型的数据结构，并将其作为结果发送。协议和传输层是运行时库的一部分。有了Thrift，就可以定义一个服务或改变通讯和传输协议，而无需重新编译代码。除了客户端部分之外，Thrift还包括服务器基础设施来集成协议和传输，如阻塞、非阻塞及多线程服务器。栈中作为I/O基础的部分对于不同的语言则有不同的实现。

![来自wikipedia](/part1/ch3-Thrift.png)


Thrift一些已经明确的优点包括：[
* 跟一些替代选择，比如SOAP相比，跨语言序列化的代价更低，因为它使用二进制格式。
* 它有一个又瘦又干净的库，没有编码框架，没有XML配置文件。
* 绑定感觉很自然。例如，Java使用java/util/ArrayList.html ArrayList<String>；C++使用std::vector<std::string>。
* 应用层通讯格式与序列化层通讯格式是完全分离的。它们都可以独立修改。
* 预定义的序列化格式包括：二进制、对HTTP友好的和压缩的二进制。
* 兼作跨语言文件序列化。
* 支持协议的[需要解释]。Thrift不要求一个集中的和明确的机制，象主版本号/次版本号。松耦合的团队可以自由地进化RPC调用。
* 没有构建依赖或非标软件。不混合不兼容的软件许可证。

首先你需要安装 thrift编译器: [download](https://thrift.apache.org/download)。
然后安装thift-go库：
```go 
git.apache.org/thrift.git/lib/go/thrift
```

对于一个服务，你需要定义thrift文件 (helloworld.thrift)：
```thrift 
namespace go greeter

service Greeter {

    string sayHello(1:string name);

}
```

编译创建相应的stub,它会创建多个辅助文件:
```sh 
thrift -r --gen go  -out src thrift/helloworld.thrift
```

服务端的代码:
```go 
package main

import (
    "fmt"
	"os"
	
    "git.apache.org/thrift.git/lib/go/thrift"
    
	"greeter"
)

const (
	NetworkAddr = "localhost:9090"
)


type GreeterHandler struct {
	
}

func NewGreeterHandler() *GreeterHandler {
	return &GreeterHandler{}
}

func (p *GreeterHandler) SayHello(name string)(r string, err error) { 
	return "Hello " + name, nil
}

func main() {
	var protocolFactory thrift.TProtocolFactory = thrift.NewTBinaryProtocolFactoryDefault()
	var transportFactory thrift.TTransportFactory = thrift.NewTBufferedTransportFactory(8192)
	transport, err := thrift.NewTServerSocket(NetworkAddr)
	if err != nil {
		fmt.Println("Error!", err)
		os.Exit(1)
	}

    handler := NewGreeterHandler()
    processor := greeter.NewGreeterProcessor(handler)
    server := thrift.NewTSimpleServer4(processor, transport, transportFactory, protocolFactory)
	fmt.Println("Starting the simple server... on ", NetworkAddr)
	server.Serve()
}
```

客户端的测试代码：
```go 
package main

import (
	"os"
	"fmt"
	"time"
	"strconv"
	"sync"
	
	"greeter"
	
	"git.apache.org/thrift.git/lib/go/thrift"
	
)

const (
	address     = "localhost:9090"
	defaultName = "world"
)

func syncTest(client *greeter.GreeterClient, name string) {
	i := 10000
	t := time.Now().UnixNano()	
	for ; i>0; i-- {
		client.SayHello(name)
	}
	fmt.Println("took", (time.Now().UnixNano() - t) / 1000000, "ms")
}

func asyncTest(client [20]*greeter.GreeterClient, name string) {
	var locks [20]sync.Mutex
	var wg sync.WaitGroup
    wg.Add(10000)
	
	i := 10000
	t := time.Now().UnixNano()	
	for ; i>0; i-- {
		go func(index int) {
		locks[index % 20].Lock()
		client[ index % 20].SayHello(name)
		wg.Done()
		locks[index % 20].Unlock()
		}(i)
	}	
	wg.Wait()
	fmt.Println("took", (time.Now().UnixNano() - t) / 1000000, "ms")
}


func main() {
	transportFactory := thrift.NewTBufferedTransportFactory(8192)
	protocolFactory := thrift.NewTBinaryProtocolFactoryDefault()

	var client [20]*greeter.GreeterClient
	
	//warm up
	for i := 0; i < 20; i++ {
		transport, err := thrift.NewTSocket(address)
		if err != nil {
			fmt.Fprintln(os.Stderr, "error resolving address:", err)
			os.Exit(1)
		}
		useTransport := transportFactory.GetTransport(transport)
		defer transport.Close()
		
		if err := transport.Open(); err != nil {
			fmt.Fprintln(os.Stderr, "Error opening socket to localhost:9090", " ", err)
			os.Exit(1)
		}
	
		client[i] = greeter.NewGreeterClientFactory(useTransport, protocolFactory)
		client[i].SayHello(defaultName)
	}
	
	sync := true
	if len(os.Args) > 1 {
		sync, _ = strconv.ParseBool(os.Args[1])
	}
	
	if sync {
		syncTest(client[0], defaultName)
	} else {
		asyncTest(client, defaultName)
	}
}
```

其它一些Go RPC框架如
1. http://www.gorillatoolkit.org/pkg/rpc
2. https://github.com/valyala/gorpc
3. https://github.com/micro/go-micro
4. https://github.com/go-kit/kit

## 参考文档
1. https://thrift.apache.org/tutorial/go
2. https://github.com/smallnest/RPC-TEST