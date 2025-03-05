## 1 Host

```go
package main

import (
	"context"
	"log"
	"time"
	
	"github.com/libp2p/go-libp2p/p2p/net/connmgr"
	"github.com/libp2p/go-libp2p"
	dht "github.com/libp2p/go-libp2p-kad-dht"
	"github.com/libp2p/go-libp2p/core/crypto"
	"github.com/libp2p/go-libp2p/core/host"
	"github.com/libp2p/go-libp2p/core/routing"
	"github.com/libp2p/go-libp2p/p2p/security/noise"
	libp2ptls "github.com/libp2p/go-libp2p/p2p/security/tls"
)

func main() {
	run()
}

func run() {
	// 构建一个具有所有默认设置的简单主机，只需使用 `New`
	h, err := libp2p.New()
	defer h.Close()
	log.Printf("Hello World, my hosts ID is %s\n", h.ID())
	// 通常情况下，你不只是想要一个简单的主机，你希望它能够完全配置以最佳支持你的p2p应用程序。
	// 让我们创建一个第二个主机，并设置更多的选项。
	// 设置你自己的密钥对
	priv, _, err := crypto.GenerateKeyPair(
		crypto.Ed25519, // 选择你的密钥类型。Ed25519 是较短的密钥类型
		-1,             // 选择密钥长度（如果可能，例如 RSA）。
	)
	// idht 是分布式哈希表的实例
	var idht *dht.IpfsDHT
	// 创建一个连接管理器
	connmgr, err := connmgr.NewConnManager(
		100,                                  // 低水位
		400,                                  // 高水位
		connmgr.WithGracePeriod(time.Minute), // 设置宽限期
	)
	// 创建第二个主机，并设置更多的选项
	h2, err := libp2p.New(
		libp2p.Identity(priv), // 使用我们生成的密钥对
		libp2p.ListenAddrStrings( // 多个监听地址
			"/ip4/0.0.0.0/tcp/9000",         // 常规 TCP 连接
			"/ip4/0.0.0.0/udp/9000/quic-v1", // QUIC 传输的 UDP 端点
		),
		libp2p.Security(libp2ptls.ID, libp2ptls.New), // 支持 TLS 连接
		libp2p.Security(noise.ID, noise.New), // 支持 noise 连接
		libp2p.DefaultTransports, // 支持任何其他默认传输（TCP）
		libp2p.ConnectionManager(connmgr), // 通过附加连接管理器来防止我们的对等节点拥有太多连接
		libp2p.NATPortMap(), // 尝试使用 uPNP 为 NATed 主机打开端口
		// 让这个主机使用 DHT 来找到其他主机
		libp2p.Routing(func(h host.Host) (routing.PeerRouting, error) {
			idht, err = dht.New(context.Background(), h)
			return idht, err
		}),
		// 如果你想帮助其他对等节点确定它们是否在 NAT 后面，你可以启动 AutoNAT 的服务器端（AutoRelay 已经运行客户端）
		// 这个服务是高度限速的，不会造成任何性能问题。
		libp2p.EnableNATService(),
	)
	defer h2.Close()
	// 最后一步是连接到引导对等节点（或任何其他对等节点）。我们将其注释掉，因为这是一个示例，
	// 对等节点将在完成后立即关闭，因此没有必要给网络带来负担。
	/*
	   // 连接到公共引导节点
	   for _, addr := range dht.DefaultBootstrapPeers {
	       pi, _ := peer.AddrInfoFromP2pAddr(addr)
	       // 我们忽略错误，因为某些引导对等节点可能会关闭，这没关系。
	       h2.Connect(ctx, *pi)
	   }
	*/
	// 打印第二个主机的ID
	log.Printf("Hello World, my second hosts ID is %s\n", h2.ID())
}
```

最后的运行结果如下：

```
go run host.go                  
2025/02/17 23:03:40 Hello World, my hosts ID is 12D3KooWBZuGeneswYGNzdWRKKiFwoQ99fVgQieUxv5GYAXgdyKc
2025/02/17 23:03:40 Hello World, my second hosts ID is 12D3KooWNzWGPZDSUg4S4Kgn8YiDJ5ZuqnLtDaY15u4xHvpAjntX
```

## 2 Echo

```go
package main

import (
    "bufio"
    "context"
    "crypto/rand"
    "flag"
    "fmt"
    "io"
    "log"
    mrand "math/rand"
    "github.com/libp2p/go-libp2p"
    "github.com/libp2p/go-libp2p/core/crypto"
    "github.com/libp2p/go-libp2p/core/host"
    "github.com/libp2p/go-libp2p/core/network"
    "github.com/libp2p/go-libp2p/core/peer"
    "github.com/libp2p/go-libp2p/core/peerstore"
    ma "github.com/multiformats/go-multiaddr"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    // 解析命令行参数
    listenF := flag.Int("l", 0, "等待传入连接的端口")
    targetF := flag.String("d", "", "目标对等点地址")
    insecureF := flag.Bool("insecure", false, "使用不加密的连接")
    seedF := flag.Int64("seed", 0, "设置随机种子以生成ID")
    flag.Parse()
    // 创建一个监听给定地址的主机，和上文一样，New 一个 host
    ha, err := makeBasicHost(*listenF, *insecureF, *seedF)
	// 如果是接收者，那么就开始监听；如果是发送者，就运行发送消息的程序
    if *targetF == "" {
        startListener(ctx, ha, *listenF, *insecureF)
        <-ctx.Done()
    } else {
        runSender(ctx, ha, *targetF)
    }
}

// 获取主机地址
func getHostAddress(ha host.Host) string {
    hostAddr, _ := ma.NewMultiaddr(fmt.Sprintf("/p2p/%s", ha.ID()))
    addr := ha.Addrs()[0]
    return addr.Encapsulate(hostAddr).String()
}

// 启动监听器
func startListener(ctx context.Context, ha host.Host, listenPort int, insecure bool) {
    fullAddr := getHostAddress(ha)
    log.Printf("我是 %s\n", fullAddr)
	// 创建流处理器，接收到 echo 流，就调用 doEcho 函数
    ha.SetStreamHandler("/echo/1.0.0", func(s network.Stream) {
        log.Println("监听器收到新流")
        if err := doEcho(s); err != nil {
            log.Println(err)
            s.Reset()
        } else {
            s.Close()
        }
    })
    log.Println("正在监听连接")
}

// 运行发送者
func runSender(ctx context.Context, ha host.Host, targetPeer string) {
    fullAddr := getHostAddress(ha)
    log.Printf("我是 %s\n", fullAddr)
	// 将目标地址从 string 转换为 multiaddr
    maddr, err := ma.NewMultiaddr(targetPeer)
    info, err := peer.AddrInfoFromP2pAddr(maddr)
	// 在 Peerstore 中存储对等节点的信息，用于后面打开流（只用 ID，Addrs 可以查找）
    ha.Peerstore().AddAddrs(info.ID, info.Addrs, peerstore.PermanentAddrTTL)
    log.Println("发送者打开流")
    s, err := ha.NewStream(context.Background(), info.ID, "/echo/1.0.0")
    log.Println("发送者说你好")
    _, err = s.Write([]byte("Hello, world!\n"))
    out, err := io.ReadAll(s)
    log.Printf("读取回复: %q\n", out)
}

// 从流中读取一行数据并将其写回
func doEcho(s network.Stream) error {
    buf := bufio.NewReader(s)
    str, err := buf.ReadString('\n')
    if err != nil {
        return err
    }
    log.Printf("读取: %s", str)
    _, err = s.Write([]byte(str))
    return err
}
```

## 3 pubsub（mDNS）

```go
package main

import (
    "context"
    "flag"
    "fmt"
    "os"
    "time"
    "github.com/libp2p/go-libp2p"
    pubsub "github.com/libp2p/go-libp2p-pubsub"
    "github.com/libp2p/go-libp2p/core/host"
    "github.com/libp2p/go-libp2p/core/peer"
    "github.com/libp2p/go-libp2p/p2p/discovery/mdns"
)

// mDNS 发现间隔时间
const DiscoveryInterval = time.Hour

// mDNS 服务标签
const DiscoveryServiceTag = "pubsub-chat-example"

func main() {
    // 解析命令行参数
    nickFlag := flag.String("nick", "", "聊天昵称，如果为空将自动生成")
    roomFlag := flag.String("room", "awesome-chat-room", "加入的聊天室名称")
    flag.Parse()
    ctx := context.Background()
    // 创建一个新的libp2p主机，监听随机TCP端口
    h, err := libp2p.New(libp2p.ListenAddrStrings("/ip4/0.0.0.0/tcp/0"))
    // 创建一个新的PubSub服务，使用GossipSub路由
    ps, err := pubsub.NewGossipSub(ctx, h)
    // 设置本地mDNS发现
    if err := setupDiscovery(h); err != nil {
        panic(err)
    }
    nick := *nickFlag
    room := *roomFlag
    // 加入聊天室
    cr, err := JoinChatRoom(ctx, ps, h.ID(), nick, room)
    if err != nil {
        panic(err)
    }
    // 启动聊天UI
    ui := NewChatUI(cr)
    if err = ui.Run(); err != nil {
        printErr("运行聊天UI时出错: %s", err)
    }
}

// 发现通知结构体，用于处理通过 mDNS 发现的新节点
type discoveryNotifee struct {
    h host.Host
}

// 处理发现的peer，连接到它们。当通过 mDNS 发现新节点时，会调用这个方法
func (n *discoveryNotifee) HandlePeerFound(pi peer.AddrInfo) {
    fmt.Printf("发现新peer %s\n", pi.ID)
    err := n.h.Connect(context.Background(), pi)
    if err != nil {
        fmt.Printf("连接peer %s时出错: %s\n", pi.ID, err)
    }
}

// 设置mDNS发现服务
func setupDiscovery(h host.Host) error {
	// 创建一个新的 mDNS 服务
    s := mdns.NewMdnsService(h, DiscoveryServiceTag, &discoveryNotifee{h: h})
    return s.Start()
}
```

下面是 ChatRoom 的实现：

```go
package main

import (
    "context"
    "encoding/json"
    "github.com/libp2p/go-libp2p/core/peer"
    pubsub "github.com/libp2p/go-libp2p-pubsub"
)

// ChatRoomBufSize 是每个主题的消息缓冲区大小
const ChatRoomBufSize = 128

// ChatRoom 表示对单个 PubSub 主题的订阅
type ChatRoom struct {
    Messages chan *ChatMessage // 接收其他节点消息的通道
    ctx      context.Context
    ps       *pubsub.PubSub
    topic    *pubsub.Topic
    sub      *pubsub.Subscription
    roomName string
    self     peer.ID
    nick     string
}

// ChatMessage 表示聊天消息，会被转换为 JSON 并发送
type ChatMessage struct {
    Message    string
    SenderID   string
    SenderNick string
}

// JoinChatRoom 尝试订阅房间名对应的 PubSub 主题，成功时返回 ChatRoom
func JoinChatRoom(ctx context.Context, ps *pubsub.PubSub, selfID peer.ID, nickname string, roomName string) (*ChatRoom, error) {
    // 加入 pubsub 主题
    topic, err := ps.Join(topicName(roomName))
    // 订阅主题
    sub, err := topic.Subscribe()
    cr := &ChatRoom{
        ctx:      ctx,
        ps:       ps,
        topic:    topic,
        sub:      sub,
        self:     selfID,
        nick:     nickname,
        roomName: roomName,
        Messages: make(chan *ChatMessage, ChatRoomBufSize),
    }
    // 启动读取消息的循环
    go cr.readLoop()
    return cr, nil
}

// Publish 发送消息到 pubsub 主题
func (cr *ChatRoom) Publish(message string) error {
    m := ChatMessage{
        Message:    message,
        SenderID:   cr.self.String(),
        SenderNick: cr.nick,
    }
    msgBytes, err := json.Marshal(m)
    if err != nil {
        return err
    }
    return cr.topic.Publish(cr.ctx, msgBytes)
}

// readLoop 从 pubsub 主题中读取消息并推送到 Messages 通道
func (cr *ChatRoom) readLoop() {
    for {
        msg, err := cr.sub.Next(cr.ctx)
        if err != nil {
            close(cr.Messages)
            return
        }
        // 只转发其他节点发送的消息
        if msg.ReceivedFrom == cr.self {
            continue
        }
        cm := new(ChatMessage)
        err = json.Unmarshal(msg.Data, cm)
        if err != nil {
            continue
        }
        // 发送有效消息到 Messages 通道
        cr.Messages <- cm
    }
}

// topicName 返回房间名对应的主题名
func topicName(roomName string) string {
    return "chat-room:" + roomName
}
```

