# Transport

rpcx 可以通过 TCP、HTTP、UnixDomain、QUIC和KCP通信。你也可以使用http客户端通过网关或者http调用来访问rpcx服务。

## TCP

这是最常用的通信方式。高性能易上手。你可以使用TLS加密TCP流量。

**Example:** [101basic](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/101basic)

服务端使用 `tcp` 做为网络名并且在注册中心注册了名为 `serviceName/tcp@ipaddress:port` 的服务。
```go
s.Serve("tcp", *addr)
```

客户端可以这样访问服务：
```go
d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")
xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, client.DefaultOption)
defer xclient.Close()
```

## HTTP Connect

你可以发送 `HTTP CONNECT` 方法给 rpcx 服务器。 Rpcx 服务器会劫持这个连接然后将它作为TCP连接来使用。
需要注意，客户端和服务端并不使用http请求/响应模型来通信，他们仍然使用二进制协议。

网络名称是 `http`， 它注册的格式是 `serviceName/http@ipaddress:port`。

HTTP Connect并不被推荐。 TCP是第一选择。

如果你想使用http 请求/响应 模型来访问服务，你应该使用网关或者http_invoke。

## Unixdomain

网络名称是 `unix`。

**Example:** [unix](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/unixdomain)

## QUIC

维基百科上QUIC的介绍 [wikipedia](https://en.wikipedia.org/wiki/QUIC)

> QUIC (Quick UDP Internet Connections, pronounced quick) is an experimental transport layer network protocol designed by Jim Roskind at Google, initially implemented in 2012, and announced publicly in 2013 as experimentation broadened. QUIC supports a set of multiplexed connections between two endpoints over User Datagram Protocol (UDP), and was designed to provide security protection equivalent to TLS/SSL, along with reduced connection and transport latency, and bandwidth estimation in each direction to avoid congestion. QUIC's main goal is to improve perceived performance of connection-oriented web applications that are currently using TCP. It also moves control of the congestion avoidance algorithms into the application space at both endpoints, rather than the kernel space, which it is claimed will allow these algorithms to improve more rapidly.

> In June 2015, an Internet Draft of a specification for QUIC was submitted to the IETF for standardization. A QUIC working group was established in 2016. The QUIC working group foresees multipath support and optional forward error correction (FEC) as the next step. The working group also focuses on network management issues that QUIC may introduce, aiming to produce an applicability and manageability statement in parallel to the actual protocol work. Internet statistics in 2017 suggest that QUIC now accounts for more than 5% of Internet traffic.

网络名称是 `quic`。

**Example:** [quic](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/quic)

## KCP

[KCP](https://github.com/skywind3000/kcp) 是一个快速并且可靠的ARQ协议。

网络名称是 `kcp`。

当你使用`kcp`的时候，你必须设置`Timeout`，利用timeout保持连接的检测。因为kcp-go本身不提供keepalive/heartbeat的功能，当服务器宕机重启的时候，原有的连接没有任何异常，只会hang住，我们只能依靠`Timeout`避免hang住。

**Example:** [kcp](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/kcp)


## reuseport

网络名称是 `reuseport`。

**Example:** [reuseport](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/reuseport)

它使用`tcp`协议并且在linux/uxix服务器上开启 [SO_REUSEPORT](https://lwn.net/Articles/542629/) socket 选项。


## TLS

**Example:** [TLS](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/tls)


你可以在服务端配置 TLS：

```go
func main() {
	flag.Parse()

	cert, err := tls.LoadX509KeyPair("server.pem", "server.key")
	if err != nil {
		log.Print(err)
		return
	}

	config := &tls.Config{Certificates: []tls.Certificate{cert}}

	s := server.NewServer(server.WithTLSConfig(config))
	s.RegisterName("Arith", new(example.Arith), "")
	s.Serve("tcp", *addr)
}
```

你可以在客户端设置 TLS：

```go
func main() {
	flag.Parse()

	d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")

	option := client.DefaultOption

	conf := &tls.Config{
		InsecureSkipVerify: true,
	}

	option.TLSConfig = conf

	xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, option)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	reply := &example.Reply{}
	err := xclient.Call(context.Background(), "Mul", args, reply)
	if err != nil {
		log.Fatalf("failed to call: %v", err)
	}

	log.Printf("%d * %d = %d", args.A, args.B, reply.C)

}
```
