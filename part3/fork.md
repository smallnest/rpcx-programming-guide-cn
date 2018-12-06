# Fork

**Example:** [fork](https://github.com/rpcx-ecosystem/rpcx-examples3/tree/master/fork)


`Fork` is a method of `XClient` and you can use it to send a request to all servers that contains this service.

If any of servers returns response whithout an error, `Fork` will return for this XClient. If all servers return errors, `Fork` returns an error of those errors.

It is like thee `Failbackup` mode. `Failbackup` uses at most two requests but `Fork` uses more requests (same to count of servers).

```go

func main() {
    ……

	xclient := client.NewXClient("Arith", client.Failover, client.RoundRobin, d, client.DefaultOption)
	defer xclient.Close()

	args := &example.Args{
		A: 10,
		B: 20,
	}

	for {
		reply := &example.Reply{}
		err := xclient.Fork(context.Background(), "Mul", args, reply)
		if err != nil {
			log.Fatalf("failed to call: %v", err)
		}

		log.Printf("%d * %d = %d", args.A, args.B, reply.C)
		time.Sleep(1e9)
	}

}
```