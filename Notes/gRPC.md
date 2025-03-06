## 1 RPC 核心概念与技术基础

在分布式系统中，不同的服务或组件运行在不同的机器上，它们之间需要进行通信和协作。RPC（Remote Procedure Call，远程过程调用）就是为了解决这个问题而诞生的。它允许一个程序调用另一个位于远程计算机上的程序，就像调用本地函数一样。

### 1.1 RPC 与本地函数调用的核心差异

- **网络延迟**： 本地函数调用几乎没有延迟，而 RPC 需要通过网络传输数据，因此会产生网络延迟。
- **序列化**：本地函数调用可以直接传递内存中的数据，而 RPC 需要将数据序列化成字节流才能在网络上传输，并在接收端解析。
- **可靠性**：本地函数调用通常是可靠的，而 RPC 需要考虑网络故障、服务器宕机等问题，因此需要提供重试、超时等机制来保证可靠性。

### 1.2 RPC 架构要素

- **客户端存根（Client Stub）**：负责将客户端的函数调用请求序列化成网络消息，并发送给服务端。
- **服务端骨架（Server Stub）**：负责接收客户端的请求，反序列化消息，调用相应的服务函数，并将结果序列化后返回给客户端。
- **序列化协议**：定义了数据如何序列化和反序列化，常见的序列化协议有 JSON、ProtoBuf 等。
- **传输协议**：定义了网络消息如何传输，常见的传输协议有 TCP、HTTP、gRPC（基于 HTTP/2）等。

### 1.3 应用场景

- **微服务**：微服务架构中，不同的服务之间通过 RPC 进行通信。
- **分布式计算**：分布式计算框架（如 Hadoop、Spark）使用 RPC 来协调不同节点之间的计算任务。
- **游戏服务器**：游戏服务器使用 RPC 来处理客户端的请求，并同步游戏状态。

### 1.4 Go 序列化

```go
// gob 序列化和反序列化，仅 go 支持
var buffer bytes.Buffer
encoder := gob.NewEncoder(&buffer)
encoder.Encode(data)
decoder := gob.NewDecoder(&buffer)
decoder.Decode(&data)
// json 序列化和反序列化
jsonData, _ := json.Marshal(data)
json.Unmarshal(jsonData, &data)
```

---

## 2 Go 原生 RPC 机制深度探索

### 2.1 标准库 net/rpc 全解析

Go 标准库 `net/rpc` 提供了一个基于 **TCP 或 HTTP** 的 RPC 框架，默认使用 **Gob** 进行序列化，也可以使用 JSON 来序列化。

Go 的 `net/rpc` 需要遵循特定的服务方法定义规范：
- 方法必须是 **导出（首字母大写）** 的。
- 方法必须有两个参数：
    1. **第一个参数是请求结构体（必须是指针类型且是导出的/内置类型）**
    2. **第二个参数是响应结构体（必须是指针类型且是导出的/内置类型）**
- 方法返回值必须是 `error`，用于标识调用成功或失败。

#### 2.1.1 服务器

```go
package main

import (
	"errors"
	"fmt"
	"net"
	"net/rpc"
)

// 定义一个结构体，RPC 方法必须绑定到这个结构体上
type Calculator struct{}

// RPC 方法：求和
func (c *Calculator) Add(args *Args, reply *int) error {
	*reply = args.A + args.B
	return nil
}

// RPC 方法：除法（包含错误处理）
func (c *Calculator) Divide(args *Args, reply *float64) error {
	if args.B == 0 {
		return errors.New("除数不能为零")
	}
	*reply = float64(args.A) / float64(args.B)
	return nil
}

// 定义 RPC 方法的参数结构体
type Args struct {
	A, B int
}

// 启动 RPC 服务器
func main() {
	calculator := new(Calculator)
	rpc.Register(calculator) // 注册服务
	// 监听端口，监听的是 TCP 协议 1234 端口
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		fmt.Println("监听失败:", err)
		return
	}
	defer listener.Close()
	fmt.Println("RPC 服务器启动，监听 1234 端口...")
	for {
		// 等待客户端连接，连接成功后返回一个 net.Conn 对象
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("连接失败:", err)
			continue
		}
		// 调用 rpc.ServeConn 处理连接
		go rpc.ServeConn(conn) // 处理 RPC 连接
	}
}
```

在服务端实现供 RPC 调用的服务时，需要遵循上面说的特定的规范。具体来说，我们需要定义一个接口，并在该接口中实现所有可供 RPC 调用的方法。实现接口后，可以通过 `rpc.Register` 或 `rpc.RegisterName` 将接口实例注册为 RPC 服务。注册后，客户端可以通过 **服务名.方法名** 的形式调用这些方法。
- **rpc.Register**：该函数会将接口实例注册为 RPC 服务，并使用接口实例的具体类型名作为服务名。服务名是通过**反射机制**获取的，即 `reflect.TypeOf(instance).Name()`。这种方式适用于服务名无需自定义的场景。
- **rpc.RegisterName**：该函数允许显式指定服务名。与 `rpc.Register` 相比，它多了一个参数，用于自定义服务名。这种方式适用于需要灵活命名服务名的场景，例如提高可读性或避免命名冲突。

接下来，服务器需要通过调用 `net.Listen` 方法来监听指定的网络端口，从而等待客户端的连接请求。一旦成功监听端口并建立连接，服务器会进入连接处理阶段。此时，可以启动一个独立的 Goroutine，调用 `rpc.ServeConn(conn)` 来处理每个客户端的 RPC 请求。

`rpc.ServeConn` 是一个专用于在单个网络连接上运行 `DefaultServer`（默认 RPC 服务器）的函数。它的核心职责是管理客户端与服务器之间的通信，直到客户端断开连接为止。需要注意的是，`ServeConn` 是一个阻塞式函数，这意味着它会持续运行以处理该连接上的所有请求，直到连接终止。由于它仅处理单一连接，因此在高并发场景中，通常需要为每个连接启动一个 Goroutine 来实现并发处理。

#### 2.1.2 客户端

```go
package main

import (
	"fmt"
	"net/rpc"
)

func main() {
	client, err := rpc.Dial("tcp", "localhost:1234")
	if err != nil {
		fmt.Println("连接 RPC 服务器失败:", err)
		return
	}
	defer client.Close()

	args := Args{A: 10, B: 5}
	var result1 int

	// 同步调用 Add 方法
	err = client.Call("Calculator.Add", &args, &result1)
	if err != nil {
		fmt.Println("RPC 调用失败:", err)
		return
	}
    fmt.Println("10 + 5 =", result1)
    
    var result2 float64
    // 异步调用 Divide 方法
	divCall := client.Go("Calculator.Divide", &args, &result2, nil)
	// 其他任务...
	
	// 等待 RPC 结果返回
	replyCall := <-divCall.Done
	if replyCall.Error != nil {
		fmt.Println("RPC 调用失败:", replyCall.Error)
		return
	}
	fmt.Println("10 / 5 =", result2)
}
```

首先，我们调用 `rpc.Dial` 向服务器拨号，建立连接。在 `net/rpc` 中，客户端的调用方式主要分为两种：

1. **同步调用**：方法调用会阻塞当前线程，直到服务器返回结果或发生错误。
2. **异步调用**：方法调用后立即返回一个 `Call` 结构体，结果通过 `Call` 的 `Done` 通道异步传递。

`Client.Call` 方法本质上是一个同步调用，它的实现实际上是基于异步调用 `Client.Go` 方法完成的。具体过程可以概括如下：
1. 调用 `Client.Go` 方法，发起异步请求，并返回一个 `Call` 结构体，该结构体包含调用的状态信息。
2. 通过监听 `Call` 结构体的 `Done` 通道，阻塞等待调用完成。
3. 调用完成后，返回结果或错误状态。

```go
// 1. 发起异步调用，返回 Call 结构体
result := client.Go(serviceMethod, args, reply, make(chan *Call, 1))
// 2. 等待调用完成，通过 Done 通道获取结果
call := <-result.Done
// 3. 返回调用的错误状态
return call.Error
```

如果需要直接进行异步调用，可以显式调用 `Client.Go` 方法。该方法会返回一个 `Call` 结构体，开发者可以通过监听 `Call.Done` 通道来获取调用结果，从而实现非阻塞式调用。

#### 2.1.3 连接池与超时控制

在高并发场景下，频繁调用 `rpc.Dial()` 会导致每次请求都建立一个新连接，从而引发资源浪费（例如过多的 TCP 连接开销）。为了解决这个问题，可以通过 **连接池** 来复用已有的连接，减少系统资源的消耗并提升性能。

```go
// 存储可复用的 RPC 客户端连接，核心是一个缓冲通道
type RPCPool struct {
	conns chan *rpc.Client
}

func NewRPCPool(size int, address string) (*RPCPool, error) {
	pool := &RPCPool{conns: make(chan *rpc.Client, size)}
	for i := 0; i < size; i++ {
		client, err := rpc.Dial("tcp", address)
		if err != nil {
			return nil, err
		}
		pool.conns <- client
	}
	return pool, nil
}
// 从连接池中获取一个可用的 RPC 客户端连接
func (p *RPCPool) Get() *rpc.Client {
	return <-p.conns
}
// 将使用完毕的 RPC 客户端连接归还到连接池中
func (p *RPCPool) Put(client *rpc.Client) {
	p.conns <- client
}
```

可以使用 `net.DialTimeout()` 限制连接时间，避免长时间阻塞：

```go
conn, err := net.DialTimeout("tcp", "localhost:1234", 2*time.Second)
```

或者在 RPC 调用时使用 `context.WithTimeout()`：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
err := client.CallContext(ctx, "Calculator.Add", &args, &result)
```

### 2.2 JSON-RPC 实战

> [!NOTE] 序列化
> `Gob` 是 Go 语言标准库中提供的一种高效的二进制编码格式。它的主要功能是将 Go 数据结构序列化为字节流（序列化），以及从字节流还原为原始的 Go 数据结构（反序列化）。Gob 的设计目标是为 Go 应用提供一种高性能、紧凑的序列化方式，特别适合在 Go 程序之间进行数据传输或持久化。
> 然而，Gob 的编码格式是 Go 特有的，并未设计为跨语言的通用序列化方案。因此，与其他编程语言的互操作性非常有限。如果需要在分布式系统中与其他语言进行通信，建议选择支持多语言的序列化工具，例如：
> - **JSON**：一种广泛使用的文本格式，具有良好的可读性和跨语言支持，但性能和数据大小不如二进制格式。
> - **Protobuf**（Protocol Buffers）：Google 开发的高效二进制序列化协议，支持多种语言，适用于高性能和跨平台的通信场景。

默认的 `net/rpc` 提供以下几种传输协议和序列化方式的组合：
1. **默认方式（TCP + Gob）**：`rpc.Dial("tcp", "localhost:1234")`，如上；
2. **使用 HTTP 承载 RPC（HTTP + Gob）**：`rpc.DialHTTP("tcp", "localhost:8080")`；
3. **使用 TCP 承载 JSON-RPC（TCP + JSON）**：`go jsonrpc.ServeConn(conn)`；
4. **使用 HTTP 承载 JSON-RPC（HTTP + JSON）**：`jsonrpc.NewServerCodec()`。

如果我们要实现 TCP+JSON，我们只需要将服务端的 `rpc.ServeConn` 修改为 `jsonrpc.ServeConn`，将客户端的 `rpc.Dial` 修改为 `jsonrpc.Dial` 即可。实际上，这二者是一样的，进行了一次封装罢了：

```go
func ServeConn(conn io.ReadWriteCloser) {
    rpc.ServeCodec(NewServerCodec(conn))
}
```

- `rpc.ServeCodec` 是 `net/rpc` 包提供的一个底层函数，用于处理自定义的编解码器（Codec）。
- `jsonrpc.NewServerCodec(conn)` 创建了一个 JSON-RPC 的编解码器，用于将 JSON 数据解析为 RPC 请求，并将 RPC 响应编码为 JSON 数据。

| **传输协议 + 序列化**  | **优点**                     | **缺点**                      | **适用场景**                |
| --------------- | -------------------------- | --------------------------- | ----------------------- |
| **TCP + Gob**   | **最高性能**，数据量小，适合长连接        | 仅限 Go 语言，调试不直观              | **Go 内部高效通信**           |
| **TCP + JSON**  | 兼容多语言，支持长连接                | JSON 开销比 Gob 高              | **跨语言高性能 RPC**          |
| **HTTP + Gob**  | 易用，兼容 HTTP 生态（路由、认证、负载均衡等） | 仅限 Go 语言                    | **Go 内部 RPC，适合短连接**     |
| **HTTP + JSON** | **最通用，支持跨语言**，调试方便         | 最低性能（HTTP 额外开销 + JSON 序列化慢） | **适合对外开放 API，微服务，前端交互** |

#### 2.2.1 基于 HTTP 的 JSON-RPC 服务器

```go
func main() {
	rpc.Register(new(Calculator)) // 注册 RPC 服务
	// 在 /jsonrpc 路径上配置一个 HTTP 处理函数
	http.HandleFunc("/jsonrpc", func(w http.ResponseWriter, r *http.Request) {
		// 创建连接适配器，将HTTP请求体和响应体适配为RPC连接
		var conn io.ReadWriteCloser = struct {
			io.Writer
			io.ReadCloser
		}{
			ReadCloser: r.Body,
			Writer:     w,
		}
		// 使用jsonrpc处理连接
		jsonrpc.ServeConn(conn)
	})
	// 启动HTTP服务器
	fmt.Println("HTTP-JSON RPC 服务器启动，监听 8080 端口...")
	http.ListenAndServe(":8080", nil)
}
```

这里创建了一个匿名结构体，实现了 `io.ReadWriteCloser` 接口：
- `io.Writer`：用于写入数据（响应）
- `io.ReadCloser`：用于读取数据（请求）并支持关闭

这个结构体将 HTTP 请求体和 HTTP 响应写入器组合起来，形成一个 RPC 连接适配器。

#### 2.2.2 基于 HTTP 的 JSON-RPC 客户端

```python
import json
import requests

url = "http://localhost:8080/jsonrpc"
headers = {"Content-Type": "application/json"}

# 发送 JSON-RPC 请求
payload = {
    "method": "Calculator.Add",
    "params": [{"A": 10, "B": 5}],
    "id": 1,
    "jsonrpc": "2.0"
}

response = requests.post(url, headers=headers, data=json.dumps(payload))

print("服务器响应:", response.json())
```

和正常的 HTTP 请求差不多，只需要打包一个参数发送过去。

#### 2.2.3 负载均衡

在生产环境中，单个 RPC 服务器的吞吐量通常难以满足高并发场景的需求，因此需要部署**多个 RPC 服务器**来分担流量。这时，就涉及到**负载均衡**的设计与实现。通过负载均衡器，可以将客户端的请求均匀分发到多个后端服务器，从而提升系统的整体性能和可用性。

一种常见的解决方案是使用 **Nginx 反向代理** 作为负载均衡器，将客户端请求统一分发到多个 RPC 服务器。以下是一个示例配置，假设有两个 Go RPC 服务器分别运行在 `127.0.0.1:8081` 和 `127.0.0.1:8082`，可以通过 Nginx 的轮询机制实现负载均衡：

```nginx
http {
    upstream jsonrpc_servers {
        least_conn; # 将请求分发到当前连接数最少的服务器
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
    }

    server {
        listen 8080;
        
        location /rpc {
            proxy_pass http://jsonrpc_servers; # 将请求转发到上游服务器组
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

启动 Nginx 后，客户端请求 `http://localhost:8080/rpc`，Nginx 会自动在 `8081` 和 `8082` 之间按照负载均衡策略分发。

---

## 3 gRPC

gRPC 支持 **四种调用方式**：

| **模式**                          | **客户端**   | **服务器**   |
| ------------------------------- | --------- | --------- |
| **Unary RPC（普通 RPC）**           | 发送单个请求    | 返回单个响应    |
| **Server Streaming RPC**        | 发送单个请求    | 返回多个响应（流） |
| **Client Streaming RPC**        | 发送多个请求（流） | 返回单个响应    |
| **Bidirectional Streaming RPC** | 发送多个请求（流） | 返回多个响应（流） |

### 3.1 Protocol Buffers 语法（v3）

Protocol Buffers（简称 Protobuf）是 Google 开发的一种用于**序列化结构化数据**的语言中立、平台中立、可扩展的机制。它广泛用于数据交换、存储和通信协议。

Proto3 文件以 `.proto` 为扩展名，文件结构通常包括以下部分：

- **syntax**：指定使用的 Protobuf 版本（`proto3`）。
- **package**：定义包名，用于防止命名冲突。
- **message**：定义数据结构。
- **service**：定义 RPC 服务接口。

```proto
syntax = "proto3";  // 指定使用 proto3 语法
package example;    // 定义包名

message Example {
  int32 id = 1;                  // 标量类型
  repeated string tags = 2;      // 重复类型
  map<string, int32> scores = 3; // 映射类型
}

service Greeter {
  rpc SayHello (Example) returns (Example); // 定义 RPC 方法
}
```

1. **字段编号**：每个字段需分配唯一编号（1-536870911），用于序列化标识。
2. **默认值**：Proto3 为每种类型提供默认值（如 `0`、空字符串等）。
3. **复合类型**：支持嵌套 `message`、`enum`，以及 `repeated`（列表）和 `map`（键值对）。
4. **RPC 服务**：通过 `service` 和 `rpc` 定义服务接口，常用于 gRPC。

Protobuf 核心的工具集是 C++ 语言开发的，在官方的 protoc 编译器中并不支持 Go 语言。要想从上面的文件生成对应的 Go 代码，需要安装相应的插件。

```shell
brew install protobuf # 安装 protobuf
protoc --version
go get github.com/golang/protobuf/protoc-gen-go # 安装 go 插件
export PATH=$PATH:$(go env GOPATH)/bin # 添加插件路径
protoc --go_out=. hello.proto
protoc --go-grpc_out=. hello.proto
```

生成两个文件：
- `helloworld.pb.go`（数据结构和序列化代码）
- `helloworld_grpc.pb.go`（gRPC 服务接口）

### 3.2 HelloWorld（Unary RPC）

#### 3.2.1 Proto 文件

```proto
syntax = "proto3";

package helloworld;

// 添加这一行指定生成的Go代码的包路径
option go_package = "./proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

定义了一个 RPC 方法，指定了输入和输出的参数，可以自动生成相关的代码，不需要手动实现了。

#### 3.2.2 服务端

```go
package main

import (
	"context"
	"log"
	"net"

	pb "helloworld/proto/helloworld"

	"google.golang.org/grpc"
)

type server struct {
    // 内嵌UnimplementedGreeterServer以保证向前兼容
    // 这个东西也实现了 SayHello，就算用户忘记了实现也不报错
	pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello, " + req.Name}, nil
}

// 匿名类型转换，编译时验证 server 实现了 GreeterServer 接口
var _ pb.GreeterServer = (*server)(nil)

func main() {
    // 监听端口
	listener, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("监听失败: %v", err)
	}
	// 启动 grpc 服务器并注册 GreeterServer RPC 服务
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	log.Println("gRPC 服务器启动...")
	// 提供服务
	if err := s.Serve(listener); err != nil {
		log.Fatalf("启动失败: %v", err)
	}
}
```

#### 3.2.3 客户端

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	pb "helloworld/proto/helloworld"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
    // 拨号连接服务器，使用不安全的连接
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("连接失败: %v", err)
	}
	defer conn.Close()
    // 注册 GreeterClient 客户端
	client := pb.NewGreeterClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
    // 调用 RPC 服务
	resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "Alice"})
	if err != nil {
		log.Fatalf("调用失败: %v", err)
	}
	fmt.Println("服务端返回:", resp.Message)
}
```

我们可以发现，其实使用了 gRPC 后，代码更加简单了，也更方面维护。

### 3.3 Pub/Sub（Bidirectional Streaming RPC）

#### 3.3.1 Proto

```proto
syntax = "proto3";

package pubsub;

option go_package = "./proto/pubsub";

// PubSub 服务定义
service PubSub {
    // 发布消息
    rpc Publish (PublishRequest) returns (PublishResponse);
    // 订阅消息
    rpc Subscribe (SubscribeRequest) returns (stream SubscribeResponse);
}

// 发布消息请求
message PublishRequest {
    string topic = 1; // 主题
    string message = 2; // 消息内容
}

// 发布消息响应
message PublishResponse {
    bool success = 1; // 是否成功
}

// 订阅消息请求
message SubscribeRequest {
    string topic = 1; // 主题
}

// 订阅消息响应
message SubscribeResponse {
    string message = 1; // 收到的消息
}
```

#### 3.3.2 服务器

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/go-redis/redis/v8"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"

	pb "pubsub/proto/pubsub"
)

type pubSubServer struct {
	pb.UnimplementedPubSubServer
	redisClient *redis.Client
}

func NewPubSubServer() *pubSubServer {
	redisClient := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	_, err := redisClient.Ping(context.Background()).Result()
	if err != nil {
		log.Fatalf("failed to connect to Redis: %v", err)
	}
	return &pubSubServer{
		redisClient: redisClient,
	}
}

func (s *pubSubServer) Publish(ctx context.Context, req *pb.PublishRequest) (*pb.PublishResponse, error) {
	err := s.redisClient.Publish(ctx, req.Topic, req.Message).Err()
	if err != nil {
		return nil, status.Errorf(codes.Internal, "failed to publish message: %v", err)
	}
	return &pb.PublishResponse{Success: true}, nil
}

func (s *pubSubServer) Subscribe(req *pb.SubscribeRequest, stream pb.PubSub_SubscribeServer) error {
	pubsub := s.redisClient.Subscribe(stream.Context(), req.Topic)
	defer pubsub.Close()

	ch := pubsub.Channel()
	for msg := range ch {
		err := stream.Send(&pb.SubscribeResponse{Message: msg.Payload})
		if err != nil {
			return status.Errorf(codes.Internal, "failed to send message: %v", err)
		}
	}
	return nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	server := grpc.NewServer()
	pb.RegisterPubSubServer(server, NewPubSubServer())

	log.Println("Server is running on port 50051")
	if err := server.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

#### 3.3.3 发布者

```go
package main

import (
	"context"
	"log"
	"math/rand"
	"strings"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	pb "pubsub/proto/pubsub"
)

// 生成随机消息
func generateRandomMessage(r *rand.Rand) string {
	words := []string{
		"你好", "世界", "消息", "发布", "订阅", "gRPC", "Golang",
		"微服务", "分布式", "通信", "实时", "数据", "流", "推送",
		"事件", "系统", "云原生", "应用", "接口", "协议",
	}
	var sb strings.Builder
	wordCount := 3 + r.Intn(5) // 3-7个词
	for i := 0; i < wordCount; i++ {
		if i > 0 {
			sb.WriteString(" ")
		}
		sb.WriteString(words[r.Intn(len(words))])
	}
	return sb.String()
}

func publishMessage(client pb.PubSubClient, topic, message string) {
	resp, err := client.Publish(context.Background(), &pb.PublishRequest{
		Topic:   topic,
		Message: message,
	})
	if err != nil {
		log.Printf("发布到主题 %s 失败: %v", topic, err)
		return
	}
	log.Printf("发布到主题 %s 成功: %v, 消息内容: %s", topic, resp.Success, message)
}

func main() {
	// 初始化随机数生成器
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("无法连接: %v", err)
	}
	defer conn.Close()
	client := pb.NewPubSubClient(conn)
	topics := []string{"topic1", "topic2", "topic3"}
	log.Println("开始轮流发布消息到不同主题...")
	// 无限循环，每隔一段时间发送一条消息
	for i := 0; ; i++ {
		// 选择主题 (轮询方式)
		topic := topics[i%len(topics)]
		// 生成随机消息
		message := generateRandomMessage(r)
		// 发布消息
		publishMessage(client, topic, message)
		// 等待1-3秒
		sleepTime := 1000 + rand.Intn(2000)
		time.Sleep(time.Duration(sleepTime) * time.Millisecond)
	}
}
```

#### 3.3.4 订阅者

```go
package main

import (
	"context"
	"log"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	pb "pubsub/proto/pubsub"
)

func subscribeToTopic(client pb.PubSubClient, topic string, wg *sync.WaitGroup) {
	defer wg.Done()
	log.Printf("Subscribing to topic: %s", topic)
	stream, err := client.Subscribe(context.Background(), &pb.SubscribeRequest{
		Topic: topic,
	})
	if err != nil {
		log.Printf("Could not subscribe to topic %s: %v", topic, err)
		return
	}
	for {
		msg, err := stream.Recv()
		if err != nil {
			log.Printf("Error receiving message from topic %s: %v", topic, err)
			break
		}
		log.Printf("Received message from topic %s: %s", topic, msg.Message)
	}
}

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	client := pb.NewPubSubClient(conn)
	var wg sync.WaitGroup
	topics := []string{"topic1", "topic2", "topic3"}
	// 创建三个协程分别订阅不同的主题
	for _, topic := range topics {
		wg.Add(1)
		go subscribeToTopic(client, topic, &wg)
	}
	// 等待所有协程完成（实际上不会结束，除非出错）
	wg.Wait()
}
```

## 4 RPC 底层协议栈拆解

1. **序列化协议性能之战**
   - Protobuf 编码原理与 varint 优化
   - FlatBuffers 零拷贝特性解析
   - Avro Schema 演进策略

2. **传输层关键技术**
   - HTTP/2 多路复用实现机制
   - QUIC 协议在 RPC 中的创新应用
   - 自定义私有协议设计实践

---

## 5 高可用架构设计模式
1. **服务治理三剑客**
   - 基于 Etcd/ZooKeeper 的服务发现
   - 负载均衡算法实现（P2C/一致性哈希）
   - 熔断器模式与弹性恢复策略（Hystrix-go）

---

## 6 分布式系统挑战赛
   - 分布式事务协调器开发
   - 最终一致性保障机制
   - 全局唯一 ID 生成器集群
