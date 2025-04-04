## 1 基础概念

### 1.1 **进程（Process）**

- **定义**：进程是操作系统分配资源的基本单位。每个进程都有独立的内存空间、文件描述符、环境变量等资源。
- **特点**：
    - 进程之间相互隔离，一个进程崩溃不会影响其他进程。
    - 进程间通信（IPC）需要通过管道、消息队列、共享内存等机制。
    - 创建和切换进程的开销较大，因为需要分配独立的内存空间和资源。
- **应用场景**：适合需要强隔离的任务，例如运行独立的应用程序。

---

### 1.2 **线程（Thread）**

- **定义**：线程是进程内的执行单元，是操作系统调度的基本单位。一个进程可以包含多个线程，这些线程共享进程的内存空间和资源。
- **特点**：
    - 线程间共享内存，通信更加高效，但也容易引发竞争条件（Race Condition）。
    - 创建和切换线程的开销比进程小，但仍需要操作系统介入。
    - 线程的数量受限于操作系统的资源限制。
- **应用场景**：适合需要并发执行的任务，例如多线程服务器。

---

### 1.3 **协程（Coroutine）**

- **定义**：协程是一种用户态的轻量级线程，由程序员显式控制调度，而不是由操作系统调度。
- **特点**：
     - 协程的切换不需要操作系统的介入，开销极小。
    - 协程是单线程的，通过协作式调度实现并发，避免了多线程的竞争问题。
    - 协程的执行是顺序的，需要显式地让出控制权（yield）。
- **应用场景**：适合高并发的 I/O 密集型任务，例如网络爬虫、异步编程。

---

### 1.4 **Goroutine**

- **定义**：Goroutine 是 Go 语言中的并发执行单元，是一种轻量级的线程实现，由 Go 运行时（runtime）管理。
- **特点**：
    - Goroutine 的创建和切换开销极小，通常只需要几 KB 的栈空间。
    - Goroutine 由 Go 运行时调度，采用 M:N 调度模型（多个 Goroutine 映射到多个操作系统线程）。
    - Goroutine 之间通过通道（Channel）进行通信，避免了共享内存的竞争问题。
    - Goroutine 的数量可以轻松达到数十万甚至更多。
- **应用场景**：适合高并发的任务，尤其是 Go 语言中的并发编程。

---

### 1.5 对比总结

| 特性       | 进程            | 线程     | 协程      | Goroutine |
| -------- | ------------- | ------ | ------- | --------- |
| **资源隔离** | 完全隔离          | 共享进程资源 | 共享线程资源  | 共享线程资源    |
| **调度方式** | 操作系统调度        | 操作系统调度 | 用户态调度   | Go 运行时调度  |
| **创建开销** | 大             | 中等     | 小       | 极小        |
| **通信方式** | IPC（管道、共享内存等） | 共享内存   | 共享内存    | Channel   |
| **并发能力** | 低             | 中等     | 高       | 极高        |
| **典型应用** | 独立应用程序        | 多线程服务器 | 异步编程、爬虫 | Go 并发编程   |

---

## 2 GMP 概述

**GMP 架构** 是 Go 运行时（runtime）用来管理和调度 Goroutine 的核心机制。GMP 是 **Goroutine**、**Machine**（线程）和 **Processor**（处理器）的缩写，它们共同协作，实现了高效的并发调度。

### 2.1 **Goroutine（G）**

- **定义**：Goroutine 是 Go 语言中的并发执行单元，是 golang 中对协程的抽象。
- **特点**：
    - 每个 Goroutine 只需要几 KB 的栈空间，且栈空间可以动态扩展。
    - Goroutine 的创建和切换开销极小。
    -  Goroutine 有自己的运行栈、生命周期状态、以及执行的任务（通过 go func 指定）
    - Goroutine 的数量可以轻松达到数十万甚至更多。
    - Goroutine 需要绑定在 M 上，可以将 M 理解为 G 的 CPU。

### 2.2 **Machine（M）**

- **定义**：M 是 Golang 对操作系统线程（OS Thread）的抽象，由 Go 运行时管理。
- **特点**：
    - M 负责执行 Goroutine 的代码。
    - M 的数量通常与 CPU 核心数相关，但可以通过 `GOMAXPROCS` 调整。
    - M 与 Goroutine 之间是多对多的关系，M 与 P 一对一绑定。

### 2.3 **Processor（P）**

- **定义**：P 是 Goroutine 的执行上下文，负责调度 Goroutine 到 M 上执行。
- **特点**：
    - P 的数量由 `GOMAXPROCS` 决定，默认等于 CPU 核心数。
    - 每个 P 维护一个本地 Goroutine 队列（Local Run Queue）。
    - P 是 Goroutine 和 M 之间的桥梁，负责将 Goroutine 分配给 M 执行。

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303144221.png)

1. 每个 P 绑定一个 M，M 从 P 的本地队列中获取 Goroutine 执行。
2. 如果 P 的本地队列为空，P 会从全局队列（Global Run Queue）中获取 Goroutine。
3. 如果全局队列也为空，P 会尝试从其他 P 的本地队列中 " 偷取 " Goroutine（Work Stealing）。
4. 如果 M 阻塞（例如系统调用），P 会解绑 M，并创建一个新的 M 继续执行 Goroutine。
5. 当阻塞的 M 恢复时，它会尝试绑定一个 P，如果没有可用的 P，M 会将 Goroutine 放入全局队列。

---

### 2.4 GMP 生态

1. **内存管理**：Go 语言的内存管理模块借鉴了 **TCMalloc（Thread-Caching Malloc）** 的设计思路，并结合 GMP 模型进行了适配优化。为每个 P 分配了一个私有的高速缓存（mcache），用于无锁化地完成 P 本地的内存分配操作。当 mcache 的内存不足时，会从全局的 mcentral 或 mheap 中申请内存，确保内存分配的高效性和可扩展性。通过将内存分配与 P 绑定，减少了锁竞争，提高了内存分配的性能。
2. **并发工具**：Go 语言中的并发工具（如锁 `mutex`、通道 `channel` 等）均针对 GMP 模型进行了优化。阻塞操作被限制在 **G（Goroutine）** 粒度，而非 **M（Thread）** 粒度，这意味着一个 Goroutine 的阻塞不会影响同一个 M 下的其他 Goroutine。阻塞和唤醒操作完全在用户态完成，无需内核介入，减少了上下文切换的开销。通过将阻塞控制在 Goroutine 粒度，Go 语言能够更高效地利用系统资源，同时避免了线程阻塞带来的性能损失。
3. **IO 多路复用**：Go 语言的 I/O 模型基于 Linux 的 **epoll** 多路复用技术，但为了避免 `epoll_wait` 操作引起 M（Thread）粒度的阻塞，Go 语言设计了一套 **netpoll** 机制。使用 `gopark` 指令将 Goroutine 阻塞在用户态，而不是让 M 进入内核态等待；使用非阻塞的 `epoll_wait` 结合 `goready` 指令唤醒 Goroutine，确保 I/O 操作控制在 Goroutine 粒度。通过将 I/O 阻塞和唤醒操作限制在 Goroutine 粒度，Go 语言能够更高效地处理大量并发 I/O 请求，同时避免了线程阻塞带来的性能开销。

## 3 GMP 数据结构

### 3.1 G

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303144936.png)

- **stack**：Goroutine 的执行栈空间，用于存储局部变量和函数调用信息。
- **stackguard0**：栈空间保护区的边界，用于探测是否需要执行栈扩容。
- **\_panic**：记录 Goroutine 执行过程中遇到的 panic 异常。
- **\_defer**：Goroutine 中挂载的 defer 函数，是一个 LIFO（后进先出）的链表结构。
- **m**：指向当前正在执行该 Goroutine 的 M（Machine）。如果 Goroutine 不处于运行状态（如阻塞或等待），此字段为空。
- **atomicstatus**：Goroutine 的生命周期状态，具体包括：
    - **\_Gidle**：Goroutine 刚被分配，还未完成初始化。
    - **\_Grunnable**：Goroutine 处于就绪态，可以被调度。
    - **\_Grunning**：Goroutine 正在被调度运行。
    - **\_Gsyscall**：Goroutine 正在执行系统调用。
    - **\_Gwaiting**：Goroutine 处于阻塞态，需要等待外部条件达成后才能恢复。
    - **\_Gdead**：Goroutine 调度结束，生命终结，或者刚被初始化准备迎接新生。
- **schedlink**：当 Goroutine 进入全局队列（grq）时，指向相邻 Goroutine 的 next 指针。

### 3.2 M

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303145612.png)

- **g0**：类似于 " 调度引擎 "，负责调度逻辑；不由用户创建，是 `m` 的伴生 Goroutine，负责为 `m` 寻找合适的普通 Goroutine（`g`）用于执行。
-  **curg**：当前正在 `m` 上执行的普通 Goroutine（由用户通过 `go func(){...}` 创建），是 `g0` 找到并分配给 `m` 执行的具体任务。
- **p**：当前与 `m` 结合的 `P（Processor）`。`P` 是 Goroutine 的执行上下文，`m` 从 `P` 的本地队列中获取 Goroutine 执行。

### 3.3 P

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303150605.png)

- **status**：表示 `p` 的生命周期状态，具体包括：
    - **\_Pidle**：`p` 因缺少 Goroutine 而进入空闲模式，此时会被添加到全局的 idle `p` 队列中。
    - **\_Prunning**：`p` 正在运行中，被 `m` 所持有，可能在运行普通 Goroutine（`g`），也可能在运行调度 Goroutine（`g0`）。
    - **\_Psyscall**：`p` 所关联的 `m` 正在执行系统调用。此时 `p` 可能被窃取并与其他 `m` 关联。
    - **\_Pdead**：`p` 已被终止。
- **m**：当前与 `p` 结合的 `m`（Machine）。如果 `p` 处于空闲状态（`_Pidle`），此字段可能为 `nil`。
- **runq**：`p` 私有的 Goroutine 队列，称为本地运行队列（Local Run Queue，简称 LRQ），用于存储待执行的 Goroutine。
- **runqhead**：LRQ 中队首节点的索引。
- **runqtail**：LRQ 中队尾节点的索引。
- **runnext**：LRQ 中的特等席，指向下一个即将执行的 Goroutine。这是一个优化机制，确保高优先级的 Goroutine 能够尽快执行。

### 3.4 Schedt

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303150621.png)

- **lock**：全局维度的互斥锁，用于保护对 `schedt` 中共享资源的并发访问。
- **midle**：空闲 `m` 队列，存储当前未绑定 `p` 的空闲 `m`；当 `m` 因缺少 Goroutine 而进入空闲状态时，会被添加到 `midle` 队列中。
- **pidle**：空闲 `p` 队列，存储当前未绑定 `m` 的空闲 `p`。当 `p` 因缺少 Goroutine 而进入空闲状态时，会被添加到 `pidle` 队列中。
- **runq**：全局 Goroutine 队列（Global Run Queue，简称 GRQ），存储所有待执行的 Goroutine。
- **runqsize**：GRQ 中当前存在的 Goroutine 数量。

`schedt` 是 Go 语言调度器中的全局共享资源模块，通过管理全局 Goroutine 队列（GRQ）、空闲 `m` 队列（`midle`）和空闲 `p` 队列（`pidle`），优化了资源的利用和调度效率。其设计避免了 `p` 和 `m` 因缺少 Goroutine 而导致 CPU 空转，并通过全局锁机制确保对共享资源的并发访问安全。

## 4 调度原理

### 4.1 Main & Go

- `main` 函数是 Go 程序的入口点，由全局唯一的 `m0`（主线程）执行。
- 用户通过 `go func(){...}` 启动的 `goroutine` 会以 `g` 的形式进入 GMP 架构。

Goroutine 的创建与执行流程：

1. 创建一个新的 `g` 实例，并将其置为就绪状态。
2. 将 `g` 添加到就绪队列中：
    - 如果当前 `P` 的本地队列（`lrq`）未满，则添加到 `lrq` 中。
    - 如果 `lrq` 满了，则加锁并添加到全局队列（`grq`）中。

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303152605.png)

### 4.2 G0 & G

从 `g0` 的视角出发，其调度流程主要包括两个核心方法：

#### 4.2.1 **`schedule` 方法**

- **作用**：`g0` 调用 `findRunnable` 方法，获取可执行的 `g`。
- **流程**：
    1. 获取当前 `g0` 和 `p`（Processor）。
    2. 调用 `findRunnable` 方法，按照优先级从本地队列（`lrq`）、全局队列（`grq`）、`netpoll` 或其他 `p` 的本地队列中获取可执行的 `g`。
    3. 如果没有可执行的 `g`，则将 `p` 和 `m` 阻塞并添加到空闲队列中。
    4. 调用 `execute` 方法，将执行权切换到获取到的 `g`。

#### 4.2.2 **`execute` 方法**

- **作用**：将执行权从 `g0` 切换到普通 `g`。
- **流程**：
    1. 建立 `m` 和 `gp` 的关系：
        - 将 `m` 的 `curg` 字段指向 `gp`。
        - 将 `gp` 的 `m` 字段指向当前 `m`。
    2. 更新 `gp` 的状态为 `running`。
    3. 设置 `gp` 的栈空间保护区边界。
    4. 调用 `gogo` 方法，将执行权切换到 `gp`。

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303153322.png)

### 4.3 Find G

`findRunnable` 方法按照优先级从多个来源获取可执行的 `G`，其核心步骤如下：

1. **每 61 次调度处理全局队列（防止饥饿）**：
    - 每经历 61 次调度后，优先从全局队列（`grq`）中获取一个 `G`，避免全局队列中的 `G` 长时间得不到执行。
    - 调用 `globrunqget` 方法（需要加锁）。
2. **从本地队列（`lrq`）获取 `G`**：
    - 优先从当前 `P` 的本地队列中获取 `G`，使用无锁的 `runqget` 方法。
3. **从全局队列（`grq`）获取 `G`**：
    - 如果本地队列为空，则从全局队列中获取 `G`，调用 `globrunqget` 方法（需要加锁）。
4. **从 `netpoll` 获取 I/O 就绪的 `G`**：
    - 如果本地队列和全局队列都为空，则通过 `netpoll` 方法（非阻塞模式）获取 I/O 就绪的 `G`。
5. **从其他 `P` 的本地队列窃取 `G`**：
    - 如果上述步骤仍未找到 `G`，则尝试从其他 `P` 的本地队列中窃取 `G`，调用 `stealWork` 方法。
6. **再次检查全局队列（Double Check）**：
    - 如果窃取失败，再次检查全局队列是否有 `G`，调用 `globrunqget` 方法（需要加锁）。
7. **将 `P` 置为空闲状态**：
    - 如果仍然没有找到 `G`，则将当前 `P` 置为空闲状态，并添加到 `schedt` 的空闲 `P` 队列（`pidle`）中。
8. **确保有一个 `M` 监听 I/O 事件**：
    - 在阻塞当前 `M` 之前，确保至少有一个 `M` 以阻塞模式执行 `netpoll`，以便及时处理 I/O 就绪事件。
9. **将 `M` 置为空闲状态**：
    - 如果 `M` 仍然无事可做，则将其添加到 `schedt` 的空闲 `M` 队列（`midle`）中，并暂停 `M` 的运行。

![image.png](https://ceyewan.oss-cn-beijing.aliyuncs.com/typora/20250303152828.png)

## 5 让渡设计

GMP 中，**让渡**是一个关键概念，它描述了 Goroutine（g）在执行过程中主动或被动地让出执行权，使得调度器能够重新分配资源，确保系统的高效运行。让渡设计主要包括三种场景：**结束让渡**、**主动让渡**和**阻塞让渡**。下面我们将详细探讨这些设计。

### 5.1 结束让渡

当一个 Goroutine（g）完成其任务并正常退出时，它会触发**结束让渡**。这个过程的核心是将 g 的执行权交还给调度器，以便调度器可以重新分配资源。

具体流程如下：

1. **g 调用 `goexit1`**：当 g 执行结束时，它会调用 `goexit1` 方法。
2. **切换到 g0**：在 `goexit1` 中，通过 `mcall` 指令，g 的执行权被切换到 g0（调度器的 Goroutine）。
3. **g0 执行 `goexit0`**：g0 调用 `goexit0` 方法，处理 g 的退出逻辑：
   - **状态更新**：将 g 的状态从 `_Grunning`（运行中）更新为 `_Gdead`（已结束）。
   - **清理资源**：清空 g 中的数据，解除 g 与 M（Machine，线程）的绑定关系。
   - **回收 g**：将 g 添加到 P（Processor，处理器）的 `gfree` 队列中，以便后续复用。
   - **发起新一轮调度**：调用 `schedule` 方法，调度器会从就绪队列中选择下一个 g 执行。

通过这种方式，结束让渡确保了资源的及时回收和调度的高效性。

### 5.2 主动让渡

**主动让渡**是指 Goroutine 在执行过程中主动让出执行权，通常通过调用 `runtime.Gosched` 方法实现。这种让渡方式适用于 g 希望让其他 g 有机会执行的场景。

具体流程如下：

1. **g 调用 `Gosched`**：g 调用 `Gosched` 方法，表示它愿意让出执行权。
2. **切换到 g0**：通过 `mcall` 指令，g 的执行权被切换到 g0。
3. **g0 执行 `gosched_m`**：g0 调用 `gosched_m` 方法，处理 g 的让渡逻辑：
   - **状态更新**：将 g 的状态从 `_Grunning` 更新为 `_Grunnable`（就绪态）。
   - **解除绑定**：解除 g 与 M 的绑定关系。
   - **添加到全局队列**：将 g 添加到全局队列 `grq` 中，等待被调度。
   - **发起新一轮调度**：调用 `schedule` 方法，调度器会从就绪队列中选择下一个 g 执行。

主动让渡的设计使得 Goroutine 能够更加灵活地控制自己的执行，避免长时间占用 CPU 资源。

### 5.3 阻塞让渡

**阻塞让渡**发生在 Goroutine 因为某些条件未满足而需要进入阻塞状态时。例如，当 g 等待一个 Mutex 或 Channel 时，它会触发阻塞让渡。

具体流程如下：

1. **g 调用 `gopark`**：g 调用 `gopark` 方法，表示它需要进入阻塞状态。
2. **切换到 g0**：通过 `mcall` 指令，g 的执行权被切换到 g0。
3. **g0 执行 `park_m`**：g0 调用 `park_m` 方法，处理 g 的阻塞逻辑：
   - **状态更新**：将 g 的状态从 `_Grunning` 更新为 `_Gwaiting`（等待态）。
   - **解除绑定**：解除 g 与 M 的绑定关系。
   - **发起新一轮调度**：调用 `schedule` 方法，调度器会从就绪队列中选择下一个 g 执行。

需要注意的是，阻塞让渡后，g 不会进入就绪队列（`lrq` 或 `grq`），而是由调用方负责在条件满足时通过 `goready` 方法将其唤醒。

### 5.4 唤醒机制

与 `gopark` 相对应的是 `goready` 方法，它用于唤醒处于阻塞状态的 g。具体流程如下：

1. **g 调用 `goready`**：g 调用 `goready` 方法，表示某个阻塞条件已经满足。
2. **切换到 g0**：通过 `systemstack` 指令，g 的执行权被切换到 g0。
3. **g0 执行 `ready`**：g0 调用 `ready` 方法，处理 g 的唤醒逻辑：
    - **状态更新**：将 g 的状态从 `_Gwaiting` 更新为 `_Grunnable`。
    - **添加到就绪队列**：将 g 添加到当前 P 的本地队列 `lrq` 或全局队列 `grq` 中。
    - **唤醒空闲的 M 或 P**：如果有 M 或 P 处于空闲状态，调度器会将其唤醒。

通过这种机制，Go 语言能够高效地管理 Goroutine 的阻塞和唤醒，确保并发任务的顺利进行。

## 6 抢占设计

在 Go 语言的 GMP（Goroutine、M、P）调度模型中，**抢占**是一种由外部干预（而非 Goroutine 主动发起）的调度机制，用于确保长时间运行的 Goroutine 不会独占 CPU 资源，从而保证系统的公平性和响应性。与**让渡**不同，抢占是由 Go 的监控线程（`sysmon`）发起的，属于第三人称视角的调度行为。

### 6.1 监控线程（sysmon）

Go 运行时启动时，会创建一个全局唯一的监控线程——`sysmon`。`sysmon` 负责周期性地执行监控任务，主要包括：

- **网络轮询（`netpoll`）**：唤醒 I/O 就绪的 Goroutine。
- **抢占（`retake`）**：对运行时间过长的 Goroutine 或处于系统调用的 P（Processor）执行抢占操作。
- **GC 触发（`gcTrigger`）**：检测是否需要发起新的垃圾回收（GC）轮次。

### 6.2 系统调用抢占

当 Goroutine 发起系统调用时，整个 M（Machine，线程）会被阻塞，导致 P 无法执行其他任务。为了减少系统调用对调度的影响，Go 采用了**系统调用抢占**的设计。

#### 6.2.1 系统调用发起

当 Goroutine 发起系统调用时，会执行 `reentersyscall` 方法：

1. **状态更新**：将 Goroutine 的状态从 `_Grunning` 更新为 `_Gsyscall`。
2. **解除 P 与 M 的绑定**：将 P 的状态更新为 `_Psyscall`，并解除 P 与 M 的绑定关系。
3. **保留弱联系**：将 P 设置为 M 的 `oldp`，以便系统调用结束后尝试复用 P。

#### 6.2.2 系统调用结束

当系统调用结束时，会执行 `exitsyscall` 方法：

1. **尝试复用 `oldp`**：如果 `oldp` 未被其他 M 占用，则直接复用。
2. **切换到 g0**：如果无法复用 `oldp`，则通过 `mcall` 切换到 g0，执行 `exitsyscall0` 方法：
    - 将 Goroutine 的状态更新为 `_Grunnable`。
    - 尝试为 M 绑定一个新的 P，如果成功则继续执行 Goroutine，否则将 Goroutine 加入全局队列并暂停 M。

### 6.3 运行超时抢占

当 Goroutine 运行时间过长（默认超过 10ms）时，`sysmon` 会对其发起**运行超时抢占**。

#### 6.3.1 协作式抢占

Go 1.14 之前，抢占主要依赖**协作式抢占**：

1. **设置抢占标识**：在 `preemptone` 方法中，将 Goroutine 的 `stackguard0` 设置为 `stackPreempt`。
2. **检查点响应**：当 Goroutine 执行到栈扩张检查点时，如果发现抢占标识，则主动让出执行权。

#### 6.3.2 非协作式抢占

为了弥补协作式抢占的不足，Go 1.14 引入了基于信号的**非协作式抢占**：

1. **发送抢占信号**：`sysmon` 通过 `signalM` 向目标 M 发送 `sigPreempt` 信号。
2. **修改程序计数器**：在信号处理函数中，通过修改 Goroutine 的程序计数器（PC）和栈顶指针（SP），插入 `asyncPreempt` 函数。
3. **执行抢占逻辑**：`asyncPreempt` 方法会切换到 g0，执行 `gopreempt_m`，完成 Goroutine 的让渡。
