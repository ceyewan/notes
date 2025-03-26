## 1 网络结构模式

### 1.1 C/S 结构

服务器 - 客户机，即 Client - Server 结构。服务器负责数据的管理，客户机负责完成与用户的交互任务。

优点：

- 能充分发挥客户端 PC 的处理能力，响应速度快。
- 安全性较高。

缺点：

- 客户端需要安装专用的客户端软件
- 对客户端的操作系统一般也会有限制，不能跨平台

### 1.2 B/S 结构

浏览器 - 服务端，即 Browser - Server 模式。

优点：

- 总体成本低，维护方便，分布性强，开发简单

缺点：

- 通信开销大
- 协议固定，http/https
- 客户端服务器端的交互是请求 - 响应模式，通常动态刷新页面，响应速度降低

## 2 MAC 地址、IP 地址、端口

- 网卡是一块被设计用来允许计算机在计算机网络上进行通讯的计算机硬件，每个网卡都有一个独一无二的称为 MAC 地址的号码。

- MAC 地址，也称为局域网地址、以太网地址，是一个用来确定网络设备位置的地址，在生产时确定。

- IP 协议是为计算机网络相互连接进行通信而设计的协议。

- IP 地址是指互联网协议地址，是 IP 协议提供的一种统一的地址格式。点分十进制的 32 位二进制数。

- 子网掩码，是一种用来指明一个 IP 地址的哪些位标识的是主机所在的子网，以及哪些位标识的是主机的位掩码。作用就是将某个 IP 地址划分为网络地址和主机地址。

- 端口是设备与外界通讯交流的出口。因为一个设备和外界可能同时有很多交流，因此需要很多端口来分别处理。

## 3 网络模型

### 3.1 OSI 七层参考模型

|OSI 参考模型|各层的解释|
|---|---|
|应用层|为应用程序提供服务|
|表示层|数据格式转化|
|会话层|建立、管理和维护会话|
|传输层|建立、管理和维护端到端的连接|
|网络层|IP 地址及路由选择|
|数据链路层|提供介质访问和链路管理|
|物理层|主要定义物理设备标准，作用是传输比特流（数模转换）|

### 3.2 TCP/IP 四层模型

TCP/IP 协议族是一个分层、多协议的通信体系，是一个四层协议系统，自底而上分别是数据链路层、网络层、传输层和应用层。每一层完成不同的功能，且通过若干协议来实现，上层协议使用下层协议提供的服务。

|层次|作用|
|---|---|
|应用层|直接为应用进程提供服务的|
|传输层|TCP/UDP 协议|
|网络层|进行网络连接的建立和终止以及 IP 地址的寻找等功能。|
|网路接口层（数据链路层 + 物理层）|网络接口层既是传输数据的物理媒介，也可以为网络层提供一条准确无误的线路。|

## 4 协议

网络协议是通信计算机双方必须共同遵从的一组约定。

- 应用层：HTTP 协议（超文本传输协议）、FTP 协议（文件传输协议）
- 传输层：TCP 协议（传输控制协议）、UDP 协议（用户数据报协议）
- 网络层：IP 协议（因特网互联协议）
- 网络接口层：ARP 协议（地址解析协议）、RARP 协议（反向地址解析协议）

### 4.1 UDP 协议

16 位源端口号 + 16 位目的端口号 + 16 位 UDP 长度 + 16 位 UDP 检验和 + 数据。头部一共 8 字节。

### 4.2 TCP 协议

1. 序列号：本报文段的数据的第一个字节的序号

2. 确认序号：期望收到对方下一个报文段的第一个数据字节的序号

3. 首部长度 (数据偏移):TCP 报文段的数据起始处距离 TCP 报文段的起始处有多远,即首部长度。单位:32 位，即以 4 字节为计算单位。

![](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/image-20221027163026360.png)

### 4.3 IP 协议

1. 版本号：IP 协议的版本，当前使用最广泛的是 4（IPv4）
2. 生存时间：每经过一个路由器转发，就把 TTL 的大小减少 1

![](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/image-20221027163451383.png)

## 5 封装和分用

应用程序数据在发送到物理网络上之前，将沿着协议栈从上往下依次传递。每层协议都将在上层数据的基础上加上自己的头部信息（有时还包括尾部信息），以实现该层的功能，这个过程就称为封装。

当帧到达目的主机时，将沿着协议栈自底向上依次传递。各层协议依次处理帧中本层负责的头部数据，以获取所需的信息，并最终将处理后的帧交给目标应用程序。这个过程称为分用。

## 6 Socket 介绍

socket 套接字，就是对网络中不同主机上的应用进程之间进行双向通信的端点的抽象。

socket 本身有 " 插座 " 的意思，在 Linux 环境下，用于表示进程间网络通信的特殊文件类型。本质为内核借助缓冲区形成的伪文件。既然是文件，那么理所当然的，我们可以使用文件描述符引用套接字。与管道类似的，Linux 系统将其封装成文件的目的是为了统一接口，使得读写套接字和读写文件的操作一致。区别是管道主要应用于本地进程间通信，而套接字多应用于网络进程间数据的传递。

## 7 字节序

现代 CPU 的累加器一次都能装载 (至少) 4 字节 (这里考虑 32 位机)，即一个整数。那么这 4 字节在内存中排列的顺序将影响它被累加器装载成的整数的值，这就是字节序问题。

大端序：一个整数的最高位字节存储在内存的低地址处，低位字节在内存的高位地址中。（02 00 00 00 表示 2）

小端序：一个整数的最高位字节存储在内存的高地址处，低位字节在内存的低位地址中。

发送端总是把要发送的数据转换成大端字节序数据后再发送，而接收端知道对方传送过来的数据总是采用大端字节序，所以接收端可以根据自身采用的字节序决定是否对接收到的数据进行转换 (小端机转换，大端机不转换)。

```c
#include<arpa/inet.h>

// h host 主机 主机字节序
// n network 网络字节序

// 转换端口
uint16_t htons(uint16_t hostshort);
uint16_t ntohs(uint16_t hostshort);
// 转 IP
uint32_t htonl(uint32_t hostlong);
uint32_t ntohl(uint32_t netlong);
```

## 8 Socket 地址

socket 地址其实是一个结构体，封装端口号和 IP 等信息。

### 8.1 通用 Socket 地址

socket 网络编程接口中表示 socket 地址的是结构体 sockaddr :

```c
#include<bits/socket.h>

struct sockaddr {
    sa_family_t sa_family;
    char sa_data[14];
};

typedef unsigned short int sa_family_t;
```

sa_family 成员是地址族类型变量，与协议族类型对应（PF_INET 表示 IPv4 协议，PF_INET6 表示 IPv6 协议族）。

sa_data 成员用于存放 socket 地址值。

### 8.2 专用 Socket 地址

sockaddr_in 和 sockaddr_in6 方便我们操作，然后统一为 sockaddr 使用。

```c
#include <netinet/in.h>
struct sockaddr_in
{
    sa_family_t sin_family;
    in_port_t sin_port; /* Port number.*/
    struct in_addr sin_addr; /* Internet address.*/
    unsigned char xxx; // 用来和 struct sockaddr 保持大小相等的
};
struct in_addr
{
    in_addr_t s_addr;
};
struct sockaddr_in6
{
    sa_family_t sin6_family;
    in_port_t sin6_port; /* Transport layer port # */
    uint32_t sin6_flowinfo; /* IPv6 flow information */
    struct in6_addr sin6_addr; /* IPv6 address */
    uint32_t sin6_scope_id; /* IPv6 scope-id */
};
typedef unsigned short uint16_t;
typedef unsigned int uint32_t;
typedef uint16_t in_port_t;
typedef uint32_t in_addr_t;
#define __SOCKADDR_COMMON_SIZE (sizeof (unsigned short int))
```

## 9 IP 地址转换

```c
#include<arpa/inet.h>

int inet_pton(int af, const char *src, void *dst);
 /*af: 地址族 AF_INET、AF_INET6
 src: 需要转换的点分十进制的 ip字符串
 dst: 转化后的结果*/

const char *inet_ntop(int af, void *src, char *dst, socklen_t size);
 /*size: 第三个参数的大小
 返回值：和 dst 是同一个值*/
```

## 10 TCP 通信流程

- TCP：传输控制协议、面向连接的、可靠的、基于字节流

服务器端：

1. 创建一个用于监听的套接字（socket() 函数）
    - 监听：监听有客户端的连接
    - 套接字：这个套接字其实就是一个文件描述符
2. 将这个监听文件描述符和本地的 IP 和端口绑定（bind() 函数）
    - 客户连接服务器的时候使用的就是这个 IP 和端口
3. 设置监听，监听的 fd 开始工作（listen() 函数）
4. 阻塞等待，当有客户端发起连接，解除阻塞，接受客户端的连接，会得到一个和客户端通信的套接字（accept() 函数）
5. 通信
    - 接收数据（recv() 函数）
    - 发送数据（send() 函数）
6. 通信结束，断开连接（close() 函数）

客户端：

1. 创建一个用于通信的套接字（socket() 函数）
2. 连接服务器，需要指定连接的服务器的 IP 和端口（connect() 函数）
3. 连接成功了，客户端可以直接和服务端通信
    - 接收数据
    - 发送数据
4. 通信结束，断开连接

## 11 套接字函数

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h> // 包含了这个头文件,上面两个就可以省略
int socket(int domain, int type, int protocol);
- 功能:创建一个套接字
- 参数:
    - domain: 协议族
        AF_INET : ipv4
        AF_INET6 : ipv6
        AF_UNIX, AF_LOCAL : 本地套接字通信(进程间通信)
    - type: 通信过程中使用的协议类型
        SOCK_STREAM : 流式协议
        SOCK_DGRAM : 报式协议
    - protocol : 具体的一个协议。一般写 0
        - SOCK_STREAM : 流式协议默认使用 TCP
        - SOCK_DGRAM : 报式协议默认使用 UDP
    - 返回值:
        - 成功:返回文件描述符,操作的就是内核缓冲区。
        - 失败:-1
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen); 
- 功能:绑定,将fd 和本地的IP + 端口进行绑定
- 参数:
    - sockfd : 通过socket函数得到的文件描述符
    - addr : 需要绑定的socket地址,这个地址封装了ip和端口号的信息
    - addrlen : 第二个参数结构体占的内存大小
int listen(int sockfd, int backlog);
- 功能:监听这个socket上的连接
- 参数:
    - sockfd : 通过socket()函数得到的文件描述符
    - backlog : 未连接的和已经连接的和的最大值, 5
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
- 功能:接收客户端连接,默认是一个阻塞的函数,阻塞等待客户端连接
- 参数:
    - sockfd : 用于监听的文件描述符
    - addr : 传出参数,记录了连接成功后客户端的地址信息(ip,port)
    - addrlen : 指定第二个参数的对应的内存大小
- 返回值:
    - 成功 :用于通信的文件描述符
    - -1 : 失败
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
- 功能: 客户端连接服务器
- 参数:
    - sockfd : 用于通信的文件描述符
    - addr : 客户端要连接的服务器的地址信息
    - addrlen : 第二个参数的内存大小
- 返回值:    成功 0, 失败 -1
ssize_t write(int fd, const void *buf, size_t count); // 写数据
ssize_t read(int fd, void *buf, size_t count); // 读数据
```

## 12 TCP 通信实现

1. 服务端

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    // 创建一个套接字
    int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    /*
        AF_INET 表示使用 IPv4 地址
        SOCK_STREAM 表示使用面向连接的套接字
        IPPROTO_TCP 表示使用 TCP 协议
        socket 也是一个文件，返回文件描述符，可以当初文件来操作
    */
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    serv_addr.sin_port = htons(1234);

    // 将套接字与特定的 IP 地址和端口绑定。流经这个 IP 和端口的数据交给套接字处理
    bind(serv_sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    // 被动监听，睡眠直到客户端发起请求，第二个参数为请求队列的最大长度
    listen(serv_sock, 20);

    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size = sizeof(clnt_addr);
    // 用来接收客户端的请求，返回客户端的套接字
    int clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_addr, &clnt_addr_size);

    // 向套接字文件中写入数据
    char str[] = "Hello Socket World";
    write(clnt_sock, str, sizeof(str));

    close(clnt_sock);
    close(serv_sock);

    return 0;
}
```

2. 客户端

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));           //每个字节都用0填充
    serv_addr.sin_family = AF_INET;                     //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1"); //具体的IP地址
    serv_addr.sin_port = htons(1234);                   //端口
    // 向服务器发起请求，服务器的 IP 地址和端口号保存在 sockaddr_in 结构体中
    connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    //读取服务器传回的数据
    char buffer[40];
    read(sock, buffer, sizeof(buffer) - 1);

    printf("Message form server: %s\n", buffer);

    //关闭套接字
    close(sock);
    return 0;
}
```

## 13 TCP 三次握手四次挥手

TCP 是一种面向连接的单播协议，在发送数据前，通信双方必须在彼此间建立一条连接。所谓的 " 连接 "，其实是客户端和服务器的内存里保存的一份关于对方的信息，如 IP 地址、端口号等。

TCP 可以看成是一种字节流，它会处理 IP 层或以下的层的丢包、重复以及错误问题。在连接的建立过程中，双方需要交换一些连接的参数。这些参数可以放在 TCP 头部。

TCP 提供了一种可靠、面向连接、字节流、传输层的服务，采用三次握手建立一个连接。采用四次挥手来关闭一个连接。

滑动窗口 (Sliding window) 是一种流量控制技术。早期的网络通信中,通信双方不会考虑网络的拥挤情况直接发送数据。由于大家不知道网络拥塞状况,同时发送数据,导致中间节点阻塞掉包,谁也发不了数据,所以就有了滑动窗口机制来解决此问题。滑动窗口协议是用来改善吞吐量的一种技术,即容许发送方在接收任何应答之前传送附加的包。接收方告诉发送方在某一时刻能送多少包 (称窗口尺寸)。

TCP 中采用滑动窗口来进行传输控制,滑动窗口的大小意味着接收方还有多大的缓冲区可以用于接收数据。发送方可以通过滑动窗口的大小来确定应该发送多少字节的数据。当滑动窗口为 0 时,发送方一般不能再发送数据报。

滑动窗口是 TCP 中实现诸如 ACK 确认、流量控制、拥塞控制的承载结构。

四次挥手：

![四次挥手](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/10093416_62a29f98950cf57199.png)

## 14 端口复用

- 防止服务器重启时之前绑定的端口还没释放
- 程序突然退出而系统没有释放端口

## 15 多进程实现并发服务器

1. 服务端

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    // 创建一个套截字
    int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    /*
        AF_INET 表示使用 IPv4 地址
        SOCK_STREAM 表示使用面向连接的套接字
        IPPROTO_TCP 表示使用 TCP 协议
        socket 也是一个文件，返回文件描述符，可以当初文件来操作
    */
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    serv_addr.sin_port = htons(9999);

    // 将套接字与特定的 IP 地址和端口绑定。流经这个 IP 和端口的数据交给套接字处理
    bind(serv_sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    // 被动监听，睡眠直到客户端发起请求，第二个参数为请求队列的最大长度
    listen(serv_sock, 20);

    while (1)
    {
        struct sockaddr_in clnt_addr;
        socklen_t clnt_addr_size = sizeof(clnt_addr);
        // 用来接收客户端的请求，返回客户端的套接字
        int clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_addr, &clnt_addr_size);
        pid_t pid = fork();
        if (pid == 0)
        {
            char buffer[1024];
            while (1)
            {
                memset(buffer, 0, sizeof(buffer));
                int len = read(clnt_sock, buffer, 1024);
                if (len == -1)
                {
                    break;
                }
                write(clnt_sock, buffer, strlen(buffer));
            }
            close(clnt_sock);
        }
    }
    close(serv_sock);
    return 0;
}
```

2. 客户端

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));           //每个字节都用0填充
    serv_addr.sin_family = AF_INET;                     //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1"); //具体的IP地址
    serv_addr.sin_port = htons(9999);                   //端口
    // 向服务器发起请求，服务器的 IP 地址和端口号保存在 sockaddr_in 结构体中
    connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
    //读取服务器传回的数据
    char buffer[40];
    int i = 0;
    while (1)
    {
        sprintf(buffer, "client send %d", i++);
        write(sock, buffer, strlen(buffer));
        memset(buffer, 0, sizeof(buffer));
        sleep(1);
        read(sock, buffer, sizeof(buffer) - 1);
        printf("Message form server: %s\n", buffer);
    }
    //关闭套接字
    close(sock);
    return 0;
}
```

3. 运行结果

![](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/image-20220718230913449.png)

## 16 多线程实现并发服务器

只需要修改服务端，在创建进程的地方创建线程即可。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>

void *myserve(void *arg)
{
    int clnt_sock = *(int *)arg;
    char buffer[1024];
    while (1)
    {
        memset(buffer, 0, sizeof(buffer));
        int len = read(clnt_sock, buffer, 1024);
        if (len == -1)
        {
            break;
        }
        write(clnt_sock, buffer, strlen(buffer));
    }
    close(clnt_sock);
}
int main()
{
    // 创建一个套截字
    int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    /*
        AF_INET 表示使用 IPv4 地址
        SOCK_STREAM 表示使用面向连接的套接字
        IPPROTO_TCP 表示使用 TCP 协议
        socket 也是一个文件，返回文件描述符，可以当初文件来操作
    */
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_ad);dr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    serv_addr.sin_port = htons(9999);

    // 将套接字与特定的 IP 地址和端口绑定。流经这个 IP 和端口的数据交给套接字处理
    bind(serv_sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    // 被动监听，睡眠直到客户端发起请求，第二个参数为请求队列的最大长度
    listen(serv_sock, 20);
    int i = 0;
    pthread_t pid[20];
    while (1)
    {
        struct sockaddr_in clnt_addr;
        socklen_t clnt_addr_size = sizeof(clnt_addr);
        // 用来接收客户端的请求，返回客户端的套接字
        int *clnt_sock = (int *)malloc(sizeof(int));
        *clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_addr, &clnt_addr_size);
        pthread_create(&pid[(i++) % 20], NULL, myserve, (void *)clnt_sock);
    }
    close(serv_sock);
    return 0;
}
```

## 17 I/O 多路复用

主要目的是为了使得程序能同时监听多个文件描述符，能够提高程序的性能。（socket 通信其实就是一种对文件描述符的读取和写入）

几种常见的 I/O 模型：

1. 阻塞等待（BIO 模型）
    1. 好处：不占用 CPU 宝贵的时间片
    2. 缺点：同一时刻只能处理一个操作，效率低
    3. 解决方法：多线程或者多进程解决。但是线程、进程也会消耗较多资源
2. 非阻塞，忙轮询（NIO 模型）
    1. 优点：提高了程序的执行效率
    2. 缺点：需要占用更多的资源
    3. 解决方法：I/O 多路复用

I/O 多路转接技术

1. select/poll
2. epoll

### 17.1 Select API 介绍

首先构造一个关于文件描述符的列表，将要监听的文件描述符添加到该列表中。然后调用一个系统函数，监听列表中的文件描述符，直到该列表中有文件描述符进行了 I/O 操作时，该函数才返回。在返回时，它会告诉进程有多少描述符要进行 I/O 操作。

```c
// sizeof(fd_set) = 128 1024
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
    /*- 参数:
        - nfds : 委托内核检测的最大文件描述符的值 + 1
        - readfds : 要检测的文件描述符的读的集合,委托内核检测哪些文件描述符的读的属性
            - 一般检测读操作
            - 对应的是对方发送过来的数据,因为读是被动的接收数据,检测的就是读缓冲区
            - 是一个传入传出参数
        - writefds : 要检测的文件描述符的写的集合,委托内核检测哪些文件描述符的写的属性
            - 委托内核检测写缓冲区是不是还可以写数据(不满的就可以写)
        - exceptfds : 检测发生异常的文件描述符的集合
        - timeout : 设置的超时时间
            struct timeval {
                long tv_sec;    seconds
                long tv_usec;   microseconds
            };
            - NULL : 永久阻塞,直到检测到了文件描述符有变化
            - tv_sec = 0 tv_usec = 0, 不阻塞
            - tv_sec > 0 tv_usec > 0, 阻塞对应的时间
    - 返回值 :
        - -1 : 失败
        - >0(n) : 检测的集合中有n个文件描述符发生了变化 
*/
// 将参数文件描述符fd对应的标志位设置为0
void FD_CLR(int fd, fd_set *set);
// 判断fd对应的标志位是0还是1, 返回值 : fd对应的标志位的值,0,返回0, 1,返回1
int FD_ISSET(int fd, fd_set *set);
// 将参数文件描述符fd 对应的标志位,设置为1
void FD_SET(int fd, fd_set *set);
// fd_set一共有1024 bit, 全部初始化为0
void FD_ZERO(fd_set *set);
```

### 17.2 Select 代码编写

```c
#include <stdio.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/select.h>

int main()
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = inet_addr("127.0.0.1");
    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));
    listen(lfd, 8);

    // tmp 是内核返回的，rdset 是我们操作的（不要直接操作内核）
    fd_set rdset, tmp;
    FD_ZERO(&rdset);
    FD_SET(lfd, &rdset);
    int maxfd = lfd;

    while (1)
    {
        tmp = rdset;

        int ret = select(maxfd + 1, &tmp, NULL, NULL, NULL);
        if (ret == -1)
        {
            perror("select");
            exit(-1);
        }
        // 没有任何改变，因此可以不执行什么操作
        else if (ret == 0)
        {
            continue;
        }
        // 说明检测到了缓冲区变化
        else if (ret > 0)
        {
            // 如果是 lfd 发生改变了，那么就说明建立的新的连接
            if (FD_ISSET(lfd, &tmp))
            {
                struct sockaddr_in clinaddr;
                int len = sizeof(clinaddr);
                int cfd = accept(lfd, (struct sockaddr *)&clinaddr, &len);
                FD_SET(cfd, &rdset);
                maxfd = maxfd > cfd ? maxfd : cfd;
            }

            // 遍历所有的 cfd ，如果有更改，那么就和它通信
            for (int i = lfd + 1; i <= maxfd; i++)
            {
                if (FD_ISSET(i, &tmp))
                {
                    char buf[1024] = {0};
                    int len = read(i, buf, sizeof(buf));
                    if (len == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    // 断开通信了
                    else if (len == 0)
                    {
                        printf("client closed...\n");
                        close(i);
                        FD_CLR(i, &rdset);
                    }
                    // 通信，先读再写
                    else
                    {
                        printf("read buf = %s\n", buf);
                        write(i, buf, strlen(buf) + 1);
                        memset(buf, 0, sizeof(buf));
                    }
                }
            }
        }
    }
    close(lfd);
    return 0;
}
```

### 17.3 Poll API 介绍及代码编写

select 是有一个遍历的，并且还有 fdset 从用户态拷贝到内核态，并且 select 支持的文件描述符数量太少了。

```c
#include <poll.h>
struct pollfd {
    int fd; /* 委托内核检测的文件描述符 */
    short events; /* 委托内核检测文件描述符的什么事件 */
    short revents; /* 文件描述符实际发生的事件 */
};
struct pollfd myfd;
myfd.fd = 5;
myfd.events = POLLIN | POLLOUT; // 读事件和写事件
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
- 参数:
    - fds : 是一个struct pollfd 结构体数组,这是一个需要检测的文件描述符的集合
    - nfds : 这个是第一个参数数组中最后一个有效元素的下标 + 1
    - timeout : 阻塞时长
        0 : 不阻塞
        -1 : 阻塞,当检测到需要检测的文件描述符有变化,解除阻塞
        >0 : 阻塞的时长
- 返回值:
    -1 : 失败
    >0(n) : 成功,n表示检测到集合中有n个文件描述符发生变化
```

和 select 的区别在于，自行指定大小，并且分离了内核区和用户区，代码如下：

```c
#include <stdio.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <poll.h>

int main()
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = inet_addr("127.0.0.1");
    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));
    listen(lfd, 8);

    struct pollfd fds[1024];
    for (int i = 0; i < 1024; i++)
    {
        fds[i].fd = -1;
        fds[i].events = POLLIN;
    }
    fds[0].fd = lfd;
    int nfds = 0;

    while (1)
    {
        int ret = poll(fds, nfds + 1, -1);
        if (ret == -1)
        {
            perror("poll");
            exit(-1);
        }
        // 没有任何改变，因此可以不执行什么操作
        else if (ret == 0)
        {
            continue;
        }
        // 说明检测到了缓冲区变化
        else if (ret > 0)
        {
            // 如果是 lfd 发生改变了，那么就说明建立的新的连接
            if (fds[0].revents & POLLIN)
            {
                struct sockaddr_in clinaddr;
                int len = sizeof(clinaddr);
                int cfd = accept(lfd, (struct sockaddr *)&clinaddr, &len);
                for (int i = 1; i < 1024; i++)
                {
                    if (fds[i].fd == -1)
                    {
                        fds[i].fd = cfd;
                        fds[i].events = POLLIN;
                        break;
                    }
                }
                nfds = nfds > cfd ? nfds : cfd;
            }

            // 遍历所有的 cfd ，如果有更改，那么就和它通信
            for (int i = 1; i <= nfds; i++)
            {
                if (fds[i].revents & POLLIN)
                {
                    char buf[1024] = {0};
                    int len = read(fds[i].fd, buf, sizeof(buf));
                    if (len == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    // 断开通信了
                    else if (len == 0)
                    {
                        printf("client closed...\n");
                        close(fds[i].fd);
                        fds[i].fd = -1;
                    }
                    // 通信，先读再写
                    else
                    {
                        printf("read buf = %s\n", buf);
                        write(fds[i].fd, buf, strlen(buf) + 1);
                        memset(buf, 0, sizeof(buf));
                    }
                }
            }
        }
    }
    close(lfd);
    return 0;
}
```

### 17.4 Epoll API 介绍

就是 poll 的优化版本，poll 之间用个数组，太低效了，因此 epoll 使用的是红黑树，插入和删除效率更高，并且直接返回哪几个而不是几个，省去了一次遍历。

```c
#include <sys/epoll.h>
// 创建一个新的epoll实例。在内核中创建了一个数据,这个数据中有两个比较重要的数据,一个是需要检测的文件描述符的信息(红黑树),还有一个是就绪列表,存放检测到数据发送改变的文件描述符信息(双向链表)。
int epoll_create(int size);
- 参数:
    size : 目前没有意义了。随便写一个数,必须大于0
- 返回值:
    -1 : 失败
    > 0 : 文件描述符,操作epoll实例的
typedef union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
struct epoll_event {
    uint32_t events; /* Epoll events */
    epoll_data_t data;/* User data variable */
};
常见的Epoll检测事件:
- EPOLLIN
- EPOLLOUT
- EPOLLERR
- EPOLLET （ET 模式）
// 对epoll实例进行管理:添加文件描述符信息,删除信息,修改信息
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
- 参数:
    - epfd : epoll实例对应的文件描述符
    - op : 要进行什么操作
        EPOLL_CTL_ADD:添加
        EPOLL_CTL_MOD:修改
        EPOLL_CTL_DEL:删除
    - fd : 要检测的文件描述符
    - event : 检测文件描述符什么事情
// 检测函数
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
- 参数:
    - epfd : epoll实例对应的文件描述符
    - events : 传出参数,保存了发送了变化的文件描述符的信息
    - maxevents : 第二个参数结构体数组的大小
    - timeout : 阻塞时间
        - 0 : 不阻塞
        - -1 : 阻塞,直到检测到fd数据发生变化,解除阻塞
        - > 0 : 阻塞的时长(毫秒)
- 返回值:
    - 成功,返回发送变化的文件描述符的个数 > 0
    - 失败 -1        
```

### 17.5 Epoll 代码编写

```c
#include <stdio.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>

int main()
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(9999);

    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    listen(lfd, 8);

    int epfd = epoll_create(1000);

    struct epoll_event epev;
    epev.events = EPOLLIN;
    epev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &epev);

    struct epoll_event epevs[1024];

    while (1)
    {
        int ret = epoll_wait(epfd, epevs, 1024, -1);
        if (ret == -1)
        {
            perror("epoll_wait");
            exit(-1);
        }

        printf("ret = %d\n", ret);

        for (int i = 0; i < ret; i++)
        {
            int curfd = epevs[i].data.fd;

            if (curfd == lfd)
            {
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);
                epev.events = EPOLLIN;
                epev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &epev);
            }
            else
            {
                if (epevs[i].events & EPOLLOUT)
                {
                    continue;
                }
                char buf[1024] = {0};
                int len = read(curfd, buf, sizeof(buf));
                if (len == -1)
                {
                    perror("read");
                    exit(-1);
                }
                else if (len == 0)
                {
                    printf("client closed...\n");
                    epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                    close(curfd);
                }
                else
                {
                    printf("read buf = %s\n", buf);
                    write(curfd, buf, strlen(buf) + 1);
                    memset(buf, 0, sizeof(buf));
                }
            }
        }
    }
    close(epfd);
    close(lfd);
    return 0;
}
```

### 17.6 Epoll 的两种工作模式

- LT 模式（水平触发）
    - 默认的工作方式，支持阻塞和非阻塞。内核告诉你一个文件描述符就绪了，如果不作任何操作，内核还是会继续通知你。
- ET 模式（边沿触发）
    - 是一种高速工作模式。只支持非阻塞，只会在文件描述符发生改变时才会通知。减少了 epoll 被触发的次数。

### 17.7 UDP 通信

服务器：

> socket() -> bind() -> recvfrom() -> sendto() -> close()

客户端：

> socket() -> recvfrom() -> sendto() -> close()

```c
#include <sys/types.h>
#include <sys/socket.h>
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
- 参数:
    - sockfd : 通信的fd
    - buf : 要发送的数据
    - len : 发送数据的长度
    - flags : 0
    - dest_addr : 通信的另外一端的地址信息
    - addrlen : 地址的内存大小
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
- 参数:
    - sockfd : 通信的fd
    - buf : 接收数据的数组
    - len : 数组的大小
    - flags : 0
    - src_addr : 用来保存另外一端的地址信息,不需要可以指定为NULL
    - addrlen : 地址的内存大小
```

### 17.8 广播和组播

广播：

> 向子网中多台计算机发送消息,并且子网中所有的计算机都可以接收到发送方发送的消息,每个广播消息都包含一个特殊的 IP 地址,这个 IP 中子网内主机标志部分的二进制全部为 1。
>
> 1. 只能在局域网中使用。
>
> 2. 客户端需要绑定服务器广播使用的端口,才可以接收到广播消息。

组播：

> 单播地址标识单个 IP 接口,广播地址标识某个子网的所有 IP 接口,多播地址标识一组 IP 接口。单播和广播是寻址方案的两个极端 (要么单个要么全部),多播则意在两者之间提供一种折中方案。多播数据报只应该由对它感兴趣的接口接收,也就是说由运行相应多播会话应用系统的主机上的接口接收。另外,广播一般局限于局域网内使用,而多播则既可以用于局域网,也可以跨广域网使用。
>
> 1. 组播既可以用于局域网,也可以用于广域网
>
> 2. 客户端需要加入多播组,才能接收到多播的数据

### 17.9 本地套接字

本地进程间的通信，是稍微简化了的 tcp 通信。

```go
// 本地套接字通信的流程 - tcp
// 服务器端
1. 创建监听的套接字
int lfd = socket(AF_UNIX/AF_LOCAL, SOCK_STREAM, 0);
2. 监听的套接字绑定本地的套接字文件 -> server端
struct sockaddr_un addr;
// 绑定成功之后,指定的sun_path中的套接字文件会自动生成。
bind(lfd, addr, len);
3. 监听
listen(lfd, 100);
4. 等待并接受连接请求
struct sockaddr_un cliaddr;
int cfd = accept(lfd, &cliaddr, len);
5. 通信
接收数据:read/recv
发送数据:write/send
6. 关闭连接
close();
// 客户端的流程
7. 创建通信的套接字
int fd = socket(AF_UNIX/AF_LOCAL, SOCK_STREAM, 0);
8. 监听的套接字绑定本地的IP 端口
struct sockaddr_un addr;
// 绑定成功之后,指定的sun_path中的套接字文件会自动生成。
bind(lfd, addr, len);
9. 连接服务器
struct sockaddr_un serveraddr;
connect(fd, &serveraddr, sizeof(serveraddr));
10. 通信
接收数据:read/recv
发送数据:write/send
11. 关闭连接
close();
```
