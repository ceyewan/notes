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

Go 语言的 RPC 框架有两个比较有特色的设计：一个是 RPC 数据打包时可以通过插件实现自定义的编码和解码；另一个是 RPC 建立在抽象的 io.ReadWriteCloser 接口之上的，我们可以将 RPC 架设在不同的通讯协议之上。

接下来基于 json 编码重新实现 RPC 服务，在服务端，只需要用 rpc.ServeCodec 函数替代 rpc.ServeConn 函数，并传入针对服务端的 json 编解码器。客户端需要手工调用 `net.Dial` 函数建立 TCP 连接，然后基于该连接建立针对客户端的 json 编解码器。

```go
// server.go
go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
// client.go
conn, err := net.Dial("tcp", "localhost:1234")
client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
```

下面，我们使用 nc 创建一个普通的 TCP 服务器代替服务端，分别查看 Json 编码和 Gob 编码传过来的内容。对于 Json 编码，其中 method 部分对应要调用的 rpc 服务和方法组合成的名字，params 部分的第一个元素为参数，id 是由调用端维护的一个唯一的调用编号。

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250122211328.png)

同理，运行 RPC 服务端，向服务器发送 json 数据模拟 RPC 方法调用，可以获得响应数据的格式。

```shell
echo -e '{"method":"HelloService.Hello","params":["hello"],"id":1}' | nc localhost 1234
```

### 1.4 HTTP 上的 RPC

Go 语言内在的 RPC 框架已经支持在 Http 协议上提供 RPC 服务。但是框架的 http 服务同样采用了内置的 gob 协议，并且没有提供采用其它协议的接口，因此从其它语言依然无法访问的。所以我们尝试自己在 http 协议上提供 jsonrpc 服务。

```go
func main() {
    rpc.RegisterName("HelloService", new(HelloService))
    // 在 /jsonrpc 路径上配置一个 HTTP 处理函数
    http.HandleFunc("/jsonrpc", func(w http.ResponseWriter, r *http.Request) {
        // io.ReadWriteCloser 是一个接口，表示既可以读取又可以写入的对象
        // 将 HTTP 请求体（r.Body）作为读取端，将 HTTP 响应写入端（w）作为写入端
        // 使用匿名结构体封装 ReadCloser 和 Writer，组合成一个 ReadWriteCloser
        var conn io.ReadWriteCloser = struct {
            io.Writer
            io.ReadCloser
        }{
            ReadCloser: r.Body,
            Writer:     w,
        }
        // 基于 conn 创建一个 JSON 编解码器，并处理 RPC 请求
        rpc.ServeRequest(jsonrpc.NewServerCodec(conn))
    })
    http.ListenAndServe(":1234", nil)
}
```

请求响应的结果如下：

```
curl localhost:1234/jsonrpc -X POST \  
    --data '{"method":"HelloService.SayHello","params":["hello"],"id":0}'

{"id":0,"result":"Hello World:hello","error":null}
```

## 2 Protobuf

**Protocol Buffers**（简称 Protobuf）是由 Google 开发的一种**高效**的、**跨语言**的序列化数据格式。它被广泛用于网络通信、数据存储等场景，尤其适合高性能和跨平台的需求。

### 2.1 Protobuf 入门

Protobuf 是一个**接口描述语言**，使用 `.proto` 文件定义数据结构和接口。通过 `.proto` 文件，Protobuf 可以生成多种语言的代码（如 Java、Python、C++ 等）以支持**跨语言通信**。

Protobuf 可以执行**序列化**（将数据结构或对象转换为字节流，以便存储或传输）和**反序列化**（从字节流恢复数据结构或对象）。

一般来说，Protobuf 的工作流程如下：
1. **定义 .proto 文件**：描述数据结构和 RPC 接口。
2. 使用 `protoc` 编译器将 `.proto` 文件生成目标语言的代码。
3. 在项目中使用生成的代码进行序列化和反序列化操作。

创建 hello.proto 文件，包装 HelloService 服务中用到的字符串类型：

```proto
syntax = "proto3"; // 采用 proto3 的语法，变量成员采用零值初始化

package main; // 和 Go 的包名保持一致，简化例子代码

// 最后 message 关键字定义一个新的 String 类型，在生成的 Go 代码中对应一个 String 结构体
// String 类型中只有一个字符串类型的 value 成员，该成员编码时用 1 编号代替名字
message String {
	string value = 1;
}
```

在 XML 或 JSON 等数据描述语言中，一般通过成员的名字来绑定对应的数据。但是 Protobuf 编码却是通过成员的唯一编号来绑定对应的数据，因此 Protobuf 编码后数据的体积会比较小，但是也非常不便于人类查阅。

Protobuf 核心的工具集是 C++ 语言开发的，在官方的 protoc 编译器中并不支持 Go 语言。要想从上面的文件生成对应的 Go 代码，需要安装相应的插件。

