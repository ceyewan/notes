作为 Go 核心的数据结构和 Goroutine 之间的通信方式，Channel 是支撑 Go 语言高性能并发编程模型的重要结构本节会介绍管道 Channel 的设计原理、数据结构和常见操作，例如 Channel 的创建、发送、接收和关闭。

## 1 设计原理

Go 语言中的设计哲学是：**不要通过共享内存通信，而是通过通信共享内存**。这种设计基于通信顺序进程（CSP）模型，goroutine 相当于进程实体，通道则是传递信息的媒介。

目前的 Channel 收发操作均遵循先进先出（FIFO）原则：

- 发送操作：先发送的数据会被先接收；
- 接收操作：先等待的 goroutine 会先获取数据。

> [!NOTE] 乐观锁和悲观锁
> - **悲观锁**：假设冲突可能发生，提前锁定资源，确保同一时间只有一个线程可以访问。这类似于在多人使用资源时，先锁住房间，防止其他人干扰。虽然能保证数据一致性，但如果锁时间长或竞争激烈，可能会导致性能瓶颈。
> - **乐观锁**：假设冲突较少，允许多个线程同时访问资源，在更新时检查是否有冲突（例如通过版本号或时间戳）。如果没有冲突，则更新成功；否则重试或处理冲突。这种方式在冲突较少时效率更高，因避免了锁的开销。

Go 通道当前使用互斥锁（mutex）实现，作为有锁队列，确保 FIFO 顺序和线程安全。这种实现简单且正确，但锁的休眠和唤醒可能带来上下文切换的额外开销，尤其在临界区较大或竞争激烈时，会成为性能瓶颈。曾经有一个无锁通道的提案，基于 CAS（比较并交换）原子操作，试图避免使用传统锁。但该实现无法保证 FIFO 顺序，因此提案目前被搁置。

## 2 数据结构

Channel 的底层数据结构是一个环形队列（circular buffer），包含以下关键字段：

```go
type hchan struct {
	qcount   uint           // 队列中当前元素的数量
	dataqsiz uint           // 环形队列的大小
	buf      unsafe.Pointer // 指向环形队列的指针
	elemsize uint16         // 元素的大小
	closed   uint32         // Channel 是否已关闭
    elemtype *_type         // 元素的类型
	sendx    uint           // 发送索引
	recvx    uint           // 接收索引
	recvq    waitq          // 等待接收的 goroutine 队列
	sendq    waitq          // 等待发送的 goroutine 队列
	lock     mutex          // 互斥锁，保护 Channel 的字段
}
```

## 3 创建管道

Channel 的创建通过 `make` 函数完成：

```go
ch := make(chan int)       // 无缓冲 Channel
ch := make(chan int, 10)   // 有缓冲 Channel，缓冲区大小为 10
```

- 无缓冲 Channel：发送和接收操作会直接阻塞，直到另一端准备好。
- 有缓冲 Channel：只有当缓冲区满时，发送操作才会阻塞；只有当缓冲区空时，接收操作才会阻塞。

## 4 关闭管道

Channel 的关闭通过 `close` 函数完成：

```go
close(ch)
```

- 关闭 Channel 后，不能再向 Channel 发送数据，否则会触发 panic。
- 关闭 Channel 后，接收操作仍然可以继续，直到缓冲区为空。
- 关闭 Channel 会唤醒所有等待的 goroutine（无论是发送者还是接收者）。

## 5 发送数据

在 Go 中，向 Channel 发送数据的操作通过 `ch <- i` 语句实现。编译器会将这一语句解析为 `OSEND` 节点，并在编译时将其转换为 `runtime.chansend1` 的调用。它只是简单地调用 `runtime.chansend` 并传入 Channel 和需要发送的数据。`runtime.chansend` 是发送数据的核心函数，首先，会执行如下操作。

- **加锁**：在发送数据之前，Channel 会被加锁，以确保线程安全。
- **检查关闭状态**：如果 Channel 已关闭，发送操作会触发 panic，抛出 "send on closed channel" 错误。

### 5.1 直接发送

如果 Channel 的接收队列 `recvq` 中存在等待的 goroutine，发送操作会直接将数据传递给接收者，而不会写入缓冲区。

其中，send 函数的逻辑如下：

- 调用 `runtime.sendDirect` 将数据直接拷贝到接收者的内存地址。
- 调用 `runtime.goready` 将接收者 goroutine 标记为可运行状态，并将其放入当前处理器的 `runnext` 队列，等待调度器唤醒。

### 5.2 写入缓冲区

如果 Channel 的缓冲区未满，发送操作会将数据写入缓冲区。

- 计算缓冲区中下一个可写入的位置 `sendx`。
- 将数据拷贝到缓冲区中，并更新 `sendx` 和 `qcount`。
- 如果 `sendx` 达到缓冲区大小，则将其重置为 0（环形缓冲区）。

### 5.3 阻塞发送

如果 Channel 的缓冲区已满且没有等待的接收者，发送操作会阻塞当前 goroutine，直到有接收者或缓冲区有空间。

- 获取当前 goroutine 并创建一个 `sudog` 结构，用于表示当前发送操作。
- 将 `sudog` 加入 Channel 的发送队列 `sendq`。
- 调用 `runtime.goparkunlock` 将当前 goroutine 挂起，等待被唤醒。
- 被唤醒后，清理 `sudog` 并返回成功。

## 6 接收数据

在 Go 中，从 Channel 接收数据的操作可以通过 `i <- ch` 和 `i, ok <- ch` 这两种方式来实现，这两种方式会被编译器分别解析为 `ORECV` 和 `OAS2RECV` 节点，最终调用的核心逻辑都在 `runtime.chanrecv` 中实现。首先，进入如下操作。

- **空 Channel**：如果 Channel 为 `nil`，且操作是阻塞的，当前 goroutine 会被挂起。
- **已关闭的 Channel**：如果 Channel 已关闭且缓冲区为空，接收操作会返回零值，并将 `received` 设置为 `false`。

### 6.1 直接接收

如果 Channel 的发送队列 `sendq` 中存在等待的 goroutine，接收操作会直接从发送者获取数据。

- 如果 Channel 无缓冲区，调用 `runtime.recvDirect` 将发送者的数据直接拷贝到接收者的内存地址。
- 如果 Channel 有缓冲区，将缓冲区中的数据拷贝到接收者，并将发送者的数据写入缓冲区。
- 调用 `runtime.goready` 将发送者 goroutine 标记为可运行状态，并放入当前处理器的 `runnext` 队列。

### 6.2 从缓冲区接收

如果 Channel 的缓冲区不为空，接收操作会从缓冲区中读取数据。

- 计算缓冲区中下一个可读取的位置 `recvx`。
- 将数据拷贝到接收者的内存地址，并清空缓冲区中的该位置。
- 更新 `recvx` 和 `qcount`。如果 `recvx` 达到缓冲区大小，则将其重置为 0（环形缓冲区）。

### 6.3 阻塞接收

如果 Channel 的缓冲区为空且没有等待的发送者，接收操作会阻塞当前 goroutine。

- 获取当前 goroutine 并创建一个 `sudog` 结构，用于表示当前接收操作。
- 将 `sudog` 加入 Channel 的接收队列 `recvq`。
- 调用 `runtime.goparkunlock` 将当前 goroutine 挂起，等待被唤醒。
- 被唤醒后，清理 `sudog` 并返回成功。
