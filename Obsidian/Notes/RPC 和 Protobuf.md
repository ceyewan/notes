## 1 RPC 入门

RPC 是远程过程调用的简称，是分布式系统中不同节点间流行的通信方式。

> **RPC（Remote Procedure Call）**: 远程过程调用是一种通信协议，使程序可以像调用本地函数一样调用远程服务器上的函数。RPC 屏蔽了底层网络通信的细节，开发者无需关心底层的传输逻辑。常见的实现有 gRPC、Thrift 等，广泛应用于分布式系统中，以实现不同服务之间的高效协作。
> **IPC（Inter-Process Communication）**: 进程间通信是一种操作系统机制，用于不同进程之间交换数据或协调操作。IPC 的方法包括管道、消息队列、共享内存、信号和套接字等。它在多进程编程中尤为重要，旨在提高系统的并发能力和资源利用效率。

### 1.1 Hello World

Go 语言的 RPC 包的路径为 `net/rpc`，也就是放在了 net 包目录下面，即该 RPC 包是建立在 net 包基础之上的。

首先，构造一个 HelloService 类型，其中的 Hello 方法用于实现打印功能。

```go
type HelloService struct {}

func (p *HelloService) SayHello(request string, reply *string) error {
    *reply = "Hello World:" + request
    return nil
}
```

其中 Hello 方法必须满足 Go 语言的 RPC 规则：方法只能有**两个可序列化**的参数；第二个参数是指针类型，并且返回一个 error 类型，且必须是公开的（方法名为大写开头）方法。

接下来就可以将 HelloService 类型的对象注册为一个 RPC 服务：

```go
func main() {
	// 将对象类型中所有满足 RPC 规则的对象方法注册为 RPC 函数，放在 HelloService 服务空间下
    rpc.RegisterName("HelloService", new(HelloService))
    // 服务端监听
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}
	for {
		// 等待客户端连接
		conn, err := listener.Accept()
		if err != nil {
			log.Fatal("Accept error:", err)
		}
		// 开启 goroutine 在该 TCP 连接上为对方提供 RPC 服务
		go rpc.ServeConn(conn)
	}
}
```

服务端的程序就写好了，接下来编写客户端的程序，请求 HelloService 服务。

```go
func main() {
	// 连接服务端，
	conn, err := rpc.Dial("tcp", "localhost:1234")
	if err != nil {
		log.Fatal("DialTCP error:", err)
	}
	// 构建请求参数
	var reply string
	// 调用具体的 RPC 方法
	// 第一个参数是用点号连接的 RPC 服务名字和方法名字
	// 第二和第三个参数分别我们定义 RPC 方法的两个参数。
	err = conn.Call("HelloService.SayHello", "hello", &reply)
	if err != nil {
	log.Fatal("Call error:", err)
	}
	fmt.Println(reply)
}
```

