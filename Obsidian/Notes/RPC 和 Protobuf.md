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
	// 连接服务端，Dial 的含义是拨号/建立连接
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

运行结果如下：

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250122195352.png)

### 1.2 安全的 RPC 接口

一般的 RPC 应用，开发人员有三种角色，服务端实现 RPC 方法的开发人员；客户端调用 RPC 方法的人员；制定服务端和客户端 RPC 接口规范的**设计人员**。

接下来，将会重构 HelloService 服务，将其抽象到 API 中，第一步是明确服务的名字和接口：

```go
// 服务的名字，增加了包路径前缀
const HelloServiceName = "path/to/pkg.HelloService" 
// 服务要实现的详细方法列表
type HelloServiceInterface interface {
    SayHello(request string, reply *string) error
}
// 注册该类型服务的函数，供服务端使用
func RegisterHelloService(svc HelloServiceInterface) error {
    return rpc.RegisterName(HelloServiceName, svc)
}
```

同理，在接口规范部分，即 API 中，增加对客户端的简单包装：

```go
type HelloServiceClient struct {
	*rpc.Client
}
// 编译时验证：确保 HelloServiceClient 类型实现了 HelloServiceInterface 接口
// 零开销：匿名变量 _ 和 nil 的 HelloServiceClient 指针
var _ HelloServiceInterface = (*HelloServiceClient)(nil)
// DialHelloService 方法，直接拨号 HelloService 服务
func DialHelloService(network, address string) (*HelloServiceClient, error) {
	c, err := rpc.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return &HelloServiceClient{Client: c}, nil
}
// 封装客户端的请求，这样客户端可以像使用本地方法一样调用 RPC 了
func (p *HelloServiceClient) SayHello(request string, reply *string) error {
	return p.Client.Call(HelloServiceName + ".SayHello", request, reply)
}
```

接下来是新的服务端和客户端程序：

```go
// server.go
func main() {
    RegisterHelloService(new(HelloService))
    listener, err := net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }
    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Fatal("Accept error:", err)
        }
        go rpc.ServeConn(conn)
    }
}
// client.go
func main() {
    client, err := DialHelloService("tcp", "localhost:1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }
    var reply string
    err = client.SayHello("hello", &reply) // 像调用本地方法一样
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(reply)
}
```

### 1.3 跨语言的 RPC

标准库的 RPC 默认采用 Go 语言特有的 gob 编码，因此从其它语言调用 Go 语言实现的 RPC 服务将比较困难。

> Gob 是 Go 语言标准库中的一种二进制编码格式，位于 encoding/gob 包中。它用于将 Go 的数据结构序列化为字节流（序列化），以及从字节流还原为 Go 的数据结构（反序列化）。
> 由于 Gob 是 Go 特有的格式，与其他语言的互操作性有限。如果需要与其他语言通信，则必须选择跨语言的序列化工具，例如 JSON 和 Protobuf。

