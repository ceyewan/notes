在 Go 语言中，`context` 包提供了一种机制，用于在 goroutine 之间传递请求范围的数据、取消信号以及截止时间。`context` 通常用于控制 goroutine 的生命周期，特别是在处理 HTTP 请求、数据库查询、RPC 调用等场景时。`context.Context` 是一个接口，定义了需要实现的四个方法，这些方法的作用和特点如下：

- `Deadline`：返回 `Context` 的截止时间（deadline）。如果 `Context` 没有设置截止时间，则返回 `ok = false`；
- `Done`：返回一个只读的 channel，用于监听 `Context` 的取消信号。当 `Context` 被取消或超时时，该 channel 会被关闭；
- `Err`：返回 `Context` 被取消的原因。如果 `Context` 尚未取消，则返回 `nil`；
- `Value`：根据 key 获取存储在 `Context` 中的值。如果 key 不存在，则返回 `nil`。

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

## 1 设计原理

`context` 的设计基于以下几个核心原则：

1. **不可变性**：`context` 是不可变的，一旦创建就不能修改。所有的修改操作（如添加值、设置截止时间等）都会返回一个新的 `context` 对象。
2. **树形结构**：`context` 通常以树形结构组织，父 `context` 的取消信号会传递给所有子 `context`。
3. **取消信号**：`context` 提供了取消机制，允许在不需要继续执行时取消 goroutine。
4. **传值**：`context` 可以携带请求范围的数据，这些数据可以在不同的 goroutine 之间传递。

查看一个示例，多个 Goroutine 同时订阅 `ctx.Done()` 管道中的消息，一旦接收到取消信号就立刻停止当前正在执行的工作。

```go
func worker(ctx context.Context, id int) {
	for {
		select {
		case <-ctx.Done(): // 监听取消信号
			fmt.Printf("Worker %d stopping...\n", id)
			return
		default:
			// 模拟工作
			fmt.Printf("Worker %d: Working...\n", id)
			time.Sleep(500 * time.Millisecond)
		}
	}
}

func main() {
	// 创建一个可取消的 Context
	ctx, cancel := context.WithCancel(context.Background())
	// 启动多个 Goroutine
	for i := 1; i <= 3; i++ {
		go worker(ctx, i)
	}
	// 模拟主程序运行一段时间
	time.Sleep(2 * time.Second)
	// 发送取消信号
	cancel()
	// 等待 Goroutine 退出
	time.Sleep(1 * time.Second)
	fmt.Println("Main: All workers stopped.")
}
```

## 2 默认上下文

`context` 包提供了两个默认的上下文：

- `context.Background()`：通常用于主函数、初始化以及测试中，作为最顶层的 `context`。
- `context.TODO()`：当不确定使用哪个 `context` 时，可以使用 `TODO`，通常用于临时占位。

```go
ctx := context.Background() // 默认上下文
```

这两个私有变量都是通过 `new(emptyCtx)` 语句初始化的，它们是指向私有结构体 `context.emptyCtx` 的指针。`context.emptyCtx` 实现了接口中的四个方法，但是却没有任何功能。

## 3 取消信号

`context` 提供了取消机制，可以通过 `context.WithCancel` 创建一个可取消的 `context`。取消操作会通知所有相关（子协程）的 goroutine 停止执行。

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // 确保在函数退出时取消 context
go func() {
    select {
    case <-ctx.Done():
        fmt.Println("Context canceled")
    }
}()
cancel() // 手动取消 context
```

## 4 传值方法

`context` 可以通过 `context.WithValue` 来传递请求范围的数据。这些数据通常是键值对的形式，键的类型必须是可比较的。

另外，由于 key、value 都是 `interface{}` 类型，因此获取到值之后还要进行类型断言。

```go
type key string

ctx := context.WithValue(context.Background(), key("user"), "Alice")

if user, ok := ctx.Value(key("user")).(string); ok {
    fmt.Println("User:", user)
}
```

## 5 思考

1. 为什么建议使用 context，即使是没有任何功能的空方法？
    **统一性**和**可扩展性**：`context.Background()` 是一个空的 `Context`，作为所有 `Context` 的根节点。即使当前不需要取消、超时或传递值，使用它可以保持代码的一致性，方便未来扩展。
2. 可以将所有的参数都使用 ctx 传递吗？
    请求 ID、用户信息、认证令牌等，这些值在整个请求生命周期内有效，因此适合通过 ctx 传递；业务逻辑参数和频繁变动的值，这些值应该通过函数参数传递，而不是 `ctx`；`ctx.Value(key)` 返回的是 `interface{}`，需要类型断言，容易引入运行时错误。

## 6 小结

`context` 是 Go 语言中用于管理 goroutine 生命周期和传递请求范围数据的重要工具。它的设计基于不可变性、树形结构、取消信号和传值机制。通过 `context`，开发者可以更好地控制并发任务的执行，避免资源泄漏，并确保在不需要继续执行时及时取消任务。
