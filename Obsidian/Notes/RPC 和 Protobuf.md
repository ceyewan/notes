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

创建 `hello.proto` 文件，包装 HelloService 服务中用到的字符串类型：

```proto
syntax = "proto3"; // 采用 proto3 的语法，变量成员采用零值初始化

package main; // 和 Go 的包名保持一致，简化例子代码

option go_package="./;hello"; // ./ 表示生成文件的存放地址；hello 表示生成文件包名

// 最后 message 关键字定义一个新的 String 类型，在生成的 Go 代码中对应一个 String 结构体
// String 类型中只有一个字符串类型的 value 成员，该成员编码时用 1 编号代替名字
message String {
	string value = 1;
}
// 用 Protobuf 定义语言无关的 RPC 服务接口才是它真正的价值所在
service HelloService {
	rpc Hello (String) returns (String);
}
```

在 XML 或 JSON 等数据描述语言中，一般通过成员的名字来绑定对应的数据。但是 Protobuf 编码却是通过成员的唯一编号来绑定对应的数据，因此 Protobuf 编码后数据的体积会比较小，但是也非常不便于人类查阅。

Protobuf 核心的工具集是 C++ 语言开发的，在官方的 protoc 编译器中并不支持 Go 语言。要想从上面的文件生成对应的 Go 代码，需要安装相应的插件。

```shell
brew install protobuf # 安装 protobuf
protoc --version
go get github.com/golang/protobuf/protoc-gen-go # 安装 go 插件
export PATH=$PATH:$(go env GOPATH)/bin # 添加插件路径
protoc --go_out=. hello.proto
protoc --go-grpc_out=. hello.proto
```

生成的 `hello.pb.go` 文件，具有一个 String 结构体，还有 Reset()、String()、ProtoMessage() 等方法，其中 ProtoMessage 方法表示这是一个实现了 proto.Message 接口的方法。生成的 `hello_grpc.pb.go` 文件多了一些类似 HelloServiceServer、HelloServiceClient 的新类型。这些类型是为 gRPC 服务的，并不符合我们当前的 RPC 要求。

### 2.2 定制代码生成插件

这后面的部分主要是实现一个 Protobuf 插件每次 hello.proto 文件中 RPC 服务的变化都可以自动生成代码；也可以通过更新插件的模板，调整或增加生成代码的内容。

教程有点过时了，作用也不大，就先跳过了。

## 3 玩转 RPC

### 3.1 客户端 RPC 的实现原理

Go 语言的 RPC 库最简单的使用方式是通过 `Client.Call` 方法进行同步阻塞调用，该方法的实现如下：

```go
func (client *Client) Call(
    serviceMethod string, args interface{},
    reply interface{},
) error {
    call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
    return call.Error
}
```

 `Call` 方法是一个同步的 RPC 调用，它会阻塞直到获得响应或发生错误。内部实际使用了异步的 `Go` 方法，然后等待其完成。整个过程其实类似下面这样：

```go
// 1. 调用异步的 Go 方法
result := client.Go(serviceMethod, args, reply, make(chan *Call, 1))
// 2. 通过 Done channel 等待调用完成
call := <-result.Done
// 3. 返回调用的错误状态
return call.Error
```

因此，也可以通过 `Client.Go` 方法直接进行异步调用返回一个表示这次调用的 `Call` 结构体。然后等待 `Call` 结构体的 Done 管道返回调用结果。

在异步调用命令发出后，一般会执行其他的任务，因此异步调用的输入参数和返回值可以通过返回的 Call 变量进行获取。

```go
func doClientWork(client *rpc.Client) {
    helloCall := client.Go("HelloService.Hello", "hello", new(string), nil)
    // do some thing
    helloCall = <-helloCall.Done
    if err := helloCall.Error; err != nil {
        log.Fatal(err)
    }
    args := helloCall.Args.(string)
    reply := helloCall.Reply.(*string)
    fmt.Println(args, *reply)
}
```

执行异步调用的 `Client.Go` 方法实现如下：

```go
func (client *Client) Go(
    serviceMethod string, args interface{},
    reply interface{},
    done chan *Call,
) *Call {
    call := new(Call)
    call.ServiceMethod = serviceMethod // 要调用的服务和方法名
    call.Args = args	// 调用参数
    call.Reply = reply	// 存储响应的指针
    call.Done = make(chan *Call, 10) // buffered.
    client.send(call)
    return call // 返回本次调用的 Call 对象
}
```

首先是构造一个表示当前调用的 call 变量，然后通过 `client.send` 将 call 的完整参数发送到 RPC 框架。`client.send` 方法调用是线程安全的，因此可以从多个 Goroutine 同时向同一个 RPC 连接发送调用指令。

当调用完成或者发生错误时，将调用 `call.done` 方法通知完成：

```go
func (call *Call) done() {
    select {
    case call.Done <- call:
        // ok
    default:
        // We don't want to block here. It is the caller's responsibility to make
        // sure the channel has enough buffer space. See comment in Go().
    }
}
```

### 3.2 基于 RPC 的 Watch 功能

> Watch 是一种**订阅机制**，允许客户端监听系统中某些资源的状态变化。当指定的条件（如数据更新、节点变化、配置更改等）发生时，Watch 功能会通知客户端，或直接返回满足条件的结果。

我们通过 RPC 构造一个简单的内存 KV 数据库。首先定义服务如下：

```go
type KVStoreService struct {
    m      map[string]string	// 用于存储 KV 数据
    filter map[string]func(key string) // 每个 Watch 调用时定义的过滤器函数列表
    mu     sync.Mutex
}

func NewKVStoreService() *KVStoreService {
    return &KVStoreService{
        m:      make(map[string]string),
        filter: make(map[string]func(key string)),
    }
}
```

然后是 Get 和 Set 方法：

```go
func (p *KVStoreService) Get(key string, value *string) error {
    p.mu.Lock()
    defer p.mu.Unlock()
    if v, ok := p.m[key]; ok {
        *value = v
        return nil
    }
    return fmt.Errorf("not found")
}

func (p *KVStoreService) Set(kv [2]string, reply *struct{}) error {
    p.mu.Lock()
    defer p.mu.Unlock()
    key, value := kv[0], kv[1]
    if oldValue := p.m[key]; oldValue != value {
        for _, fn := range p.filter {
            fn(key)
        }
    }
    p.m[key] = value
    return nil
}
```

在 Set 方法中，输入参数是 key 和 value 组成的数组，用一个匿名的空结构体表示忽略了输出参数。当修改某个 key 对应的值时会调用每一个过滤器函数。

而过滤器列表在 Watch 方法中提供：

```go
func (p *KVStoreService) Watch(timeoutSecond int, keyChanged *string) error {
    id := fmt.Sprintf("watch-%s-%03d", time.Now(), rand.Int())
    ch := make(chan string, 10) // buffered
    p.mu.Lock()
    p.filter[id] = func(key string) { ch <- key }
    p.mu.Unlock()
    select {
    case <-time.After(time.Duration(timeoutSecond) * time.Second):
        return fmt.Errorf("timeout")
    case key := <-ch:
        *keyChanged = key
        return nil
    }
    return nil
}
```

Watch 方法的输入参数是超时的秒数。当有 key 变化时将 key 作为返回值返回。如果超过时间后依然没有 key 被修改，则返回超时的错误。Watch 的实现中，用唯一的 id 表示每个 Watch 调用，然后根据 id 将自身对应的过滤器函数注册到 `p.filter` 列表。

KVStoreService 服务的注册和启动过程我们不再赘述。下面我们看看如何从客户端使用 Watch 方法：

```go
func doClientWork(client *rpc.Client) {
    go func() {
        var keyChanged string
        err := client.Call("KVStoreService.Watch", 30, &keyChanged)
        if err != nil {
            log.Fatal(err)
        }
        fmt.Println("watch:", keyChanged)
    } ()
    err := client.Call(
        "KVStoreService.Set", [2]string{"abc", "abc-value"},
        new(struct{}),
    )
    if err != nil {
        log.Fatal(err)
    }
    time.Sleep(time.Second*3)
}
```

首先启动一个独立的 Goroutine 监控 key 的变化。同步的 watch 调用会阻塞，直到有 key 发生变化或者超时。然后在通过 Set 方法修改 KV 值时，服务器会将变化的 key 通过 Watch 方法返回。这样我们就可以实现对某些状态的监控。