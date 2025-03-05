## 1 Hello World

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello World")
}
```

运行命令：`go run main.go`
编译命令：`go build main.go`

## 2 值

字符串、整型、浮点型、布尔型。字符串可以通过 + 连接。

## 3 变量

`var` 声明 1 个或者多个变量，Go 会自动推断已经有初始值的变量的类型。
`:=` 语法是声明并初始化变量的简写。

## 4 常量

`const` 用于声明一个常量，支持字符、字符串、布尔、数值常量。
常数表达式可以执行**任意精度**的运算，数值型常量没有确定的类型，直到被给定某个类型，比如**显式类型转化**。

## 5 For 循环

`for` 是 Go 中唯一的循环结构。但是用法可以实现类似 while 的效果。

```go
i := 1
for i <= 3 {
	fmt.Println(i)
	i = i + 1
}

for j := 7; j <= 9; j++ {
	fmt.Println(j)
}

for {
	fmt.Println("loop")
	break
}
```

在循环内部，可以使用 `break`、`return` 跳出循环，也可以使用 `continue` 直接进入下一次循环。

## 6 if/else 分支

在条件语句之前可以有一个**声明语句**；在这里声明的变量可以在这个语句**所有**的条件分支中使用。

```go
if num := 9; num < 0 {
	fmt.Println(num, "is negative")
} else if num < 10 {
	fmt.Println(num, "has 1 digit")
} else {
	fmt.Println(num, "has multiple digits")
}
```

在 Go 中，条件语句的圆括号不是必需的，但是花括号是必需的。 Go 没有三目运算符！

## 7 Switch

## 8 数组

在 Go 中，**数组** 是一个具有编号且长度固定的元素序列。

```go
var a [5]int
b := [5]int{1, 2, 3, 4, 5}
var twoD [2][3]int // 两行三列
```

内置函数 `len` 可以返回数组的长度。

## 9 Slice

_Slice_ 是 Go 中一个重要的数据类型，它提供了比数组更强大的序列交互方式。

```go
s := make([]string, 3)
s = append(s, "d")
s = append(s, "e", "f")
c := make([]string, len(s))
copy(c, s) // 深拷贝，与浅拷贝对应
twoD := make([][]int, 3) // 二维切片的声明
for i := 0; i < 3; i++ {
	innerLen := i + 1
	twoD[i] = make([]int, innerLen)
	for j := 0; j < innerLen; j++ {
		twoD[i][j] = i + j
	}
}
```

## 10 Map

_map_ 是 Go 内建的[关联数据类型](http://zh.wikipedia.org/wiki/%E5%85%B3%E8%81%94%E6%95%B0%E7%BB%84) （在一些其他的语言中也被称为 _哈希(hash)_ 或者 _字典(dict)_ ）。
要创建一个空 map，需要使用内建函数 `make`：`make(map[key-type]val-type)`。

```go
m := make(map[string]int)
v1 := m["k1"]
_, ok := m["k2"] // 第二个返回值表明了 map 中是否存在这个键。
n := map[string]int{"foo": 1, "bar": 2} // 声明并初始化一个新的 map。
```

## 11 Range 遍历

_range_ 用于迭代各种各样的数据结构。

```go
nums := []int{2, 3, 4}
for index, num := range nums {
	fmt.Printf("%d -> %d\n", index, num)
}
kvs := map[string]string{"a": "apple", "b": "banana"}
for k, v := range kvs { // 也可以只接收 k
	fmt.Printf("%s -> %s\n", k, v)
}
for i, c := range "go" { // 迭代 unicode 码点(code point)
	fmt.Println(i, c)
}
```

## 12 函数

```go
func plusPlus(a, b, c int) int {
    return a + b + c
}
```

## 13 多返回值

```go
func vals() (int, int) {
    return 3, 7
}
```
## 14 变参函数

[_可变参数函数_](https://zh.wikipedia.org/wiki/%E5%8F%AF%E8%AE%8A%E5%8F%83%E6%95%B8%E5%87%BD%E6%95%B8)。在调用时可以传递任意数量的参数。例如，`fmt.Println` 就是一个常见的变参函数。

```go
func sum(nums ...int) {
    fmt.Print(nums, " ")
    total := 0
    for _, num := range nums {
        total += num
    }
    fmt.Println(total)
}
nums := []int{1, 2, 3, 4}
sum(nums...)
```
## 15 闭包

闭包是一种可以捕获并绑定外部环境变量的匿名函数。闭包是一个函数，这个函数引用了它所在环境中的变量。即使这个函数在环境之外被调用，它仍然可以访问并操作这些变量。

```go
func main() {
    // 闭包示例
    add := createAdder(10) // 创建一个闭包，捕获了参数 base 的值 10
    fmt.Println(add(5))   // 输出：15
    fmt.Println(add(20))  // 输出：30
}
// createAdder 返回一个闭包
func createAdder(base int) func(int) int {
    return func(value int) int {
        return base + value // 闭包捕获了 base 变量
    }
}
```

## 16 递归

闭包也可以是递归的，但这要求在定义闭包之前用类型化的 `var` 显式声明闭包。

```go
func main() {
    // 声明一个类型化的变量，用于保存递归闭包
    var factorial func(int) int
    // 定义递归闭包
    factorial = func(n int) int {
        if n == 0 {
            return 1 // 递归终止条件
        }
        return n * factorial(n-1) // 调用自身
    }
    // 调用闭包计算阶乘
    fmt.Println(factorial(5)) // 输出：120
}
```
## 17 指针

Go 支持指针，允许在程序中通过**引用传递**来传递值和数据结构。通过 `&i` 语法来取得 `i` 的内存地址，即指向 `i` 的指针。函数体内的 `*iptr` 会 _解引用_ 这个指针，从它的内存地址得到这个地址当前对应的值。

## 18 字符串和 rune 类型

Go 语言中的字符串是一个只读的 byte 类型的切片。 Go 语言和标准库特别对待字符串作为以 UTF-8 为编码的文本容器。在其他语言当中，字符串由”字符”组成。

在Go语言当中，字符的概念被称为 `rune`：
- 是 Go 中的内建类型，实际是 int32 类型的别名。
- 用于表示 Unicode 代码点，可以处理多字节字符。

```go
func main() {
	// 定义一个字符串
	str := "Go语言"
	// 遍历字符串（按字节）
	fmt.Println("按字节遍历:")
	for i := 0; i < len(str); i++ {
		fmt.Printf("索引 %d: %x\n", i, str[i]) // 输出字节的十六进制值
	}
	// 按 rune 遍历（Unicode 代码点）
	fmt.Println("\n按 Unicode 遍历:")
	for i, r := range str {
		fmt.Printf("索引 %d: %c (Unicode: %U)\n", i, r, r) // 输出字符和代码点
	}
	// 字符串与 rune 转换
	fmt.Println("\n字符串与 rune 转换:")
	runes := []rune(str) // 转换为 rune 切片
	fmt.Printf("runes: %v\n", runes) // 输出代码点
	fmt.Println("还原为字符串:", string(runes)) // 还原为字符串
}
```

## 19 结构体

Go 的结构体(struct)是带类型的字段(fields)集合。这在组织数据时非常有用。这是一个可变(mutable)数据类型，即不可对它进行哈希。

## 20 方法

Go 支持为结构体类型定义**方法**(methods) 。

```go
type rect struct {
    width, height int
}
func (r *rect) area() int { // 为指针类型的接收者定义方法
    return r.width * r.height
}
func (r rect) perim() int { // 为值类型的接收者定义方法
    return 2*r.width + 2*r.height
}
```

调用方法时，Go 会自动处理值和指针之间的转换。为了避免在调用方法时产生一个拷贝，可以使用指针来调用方法；但是使用指针来调用方法也会导致对结构体的修改作用到元素上。

## 21 接口

要在 Go 中实现一个接口，我们只需要实现接口中的所有方法。如果一个变量实现了某个接口，我们就可以调用指定接口中的方法。

```go
type geometry interface {
    area() float64
    perim() float64
}
```

这是一个几何体的基本接口，如果我们为圆形或者方形实现了这两个方法，那么他们就能当成 geometry 来用。

## 22 Embedding

Go 支持对于结构体(struct)和接口(interfaces)的**嵌入**(embedding)以表达一种更加无缝的组合(composition)类型。
- 结构体嵌入允许一个结构体包含另一个结构体作为字段，而不需要显式声明字段名。嵌入的结构体方法和字段会“提升”到外层结构体中。
- 接口嵌入允许一个接口包含多个接口或方法，从而形成更复杂的接口。

```go
type base struct {
    num int
}
func (b base) describe() string {
    return fmt.Sprintf("base with num=%v", b.num)
}
type container struct {
    base
    str string
}
```

如上，container 类型的变量会拥有 base 的变量和 base 的方法。

```go
// 定义基础接口
type Speaker interface {
    Speak()
}
type Greeter interface {
    Greet()
}
// 嵌入接口
type Human interface {
    Speaker
    Greeter
}
```

## 23 泛型

```go
func MapKeys[K comparable, V any](m map[K]V) []K {
    r := make([]K, 0, len(m))
    for k := range m {
        r = append(r, k)
    }
    return r
}
```

`MapKeys` 接受任意类型的Map并返回其Key的切片。这个函数有2个类型参数 `K` 和 `V`； `K` 是 `comparable` 类型，也就是说我们可以通过 `==` 和 `!=` 操作符对这个类型的值进行比较。这是Go中Map的Key所必须的。 `V` 是 `any` 类型，意味着它不受任何限制 (`any` 是 `interface{}` 的别名类型)。

## 24 错误处理

符合 Go 语言习惯的做法是使用一个独立、明确的返回值来传递错误信息。Go 语言的处理方式能清楚的知道哪个函数返回了错误，并使用跟其他（无异常处理的）语言类似的方式来处理错误。返回错误值为 nil 代表没有错误。

## 25 协程 goroutine

协程是轻量级的执行线程。

```go
func f(from string) {
    for i := 0; i < 3; i++ {
        fmt.Println(from, ":", i)
    }
}
func main() {
    f("direct")
    go f("goroutine")
    go func(msg string) {
        fmt.Println(msg)
    }("going")
    time.Sleep(time.Second)
    fmt.Println("done")
}
```

使用 `go f(s)` 在一个协程中调用这个函数。这个新的 Go 协程将会**并发地**执行这个函数。也可以为匿名函数启动一个协程。
## 26 通道

通道(channels) 是连接多个协程的管道。你可以从一个协程将值发送到通道，然后在另一个协程中接收。

```go
func main() {
    messages := make(chan string) // 创建一个 string 类型的通道
    go func() { messages <- "ping" }() // 向通道中写值
    msg := <-messages	// 从通道中读值
    fmt.Println(msg)
}
```

默认发送和接收操作是**阻塞**的，直到发送方和接收方都就绪。这个特性允许我们，不使用任何其它的同步操作，就可以在程序结尾处等待消息 `"ping"`。
## 27 通道缓冲

默认情况下，通道是 _无缓冲_ 的，这意味着只有对应的接收（`<- chan`） 通道准备好接收时，才允许进行发送（`chan <-`）。 _有缓冲通道_ 允许在没有对应接收者的情况下，缓存一定数量的值。

```go
messages := make(chan string, 2)
messages <- "buffered"
messages <- "channel"
fmt.Println(<-messages)
fmt.Println(<-messages)
```

## 28 通道同步

```go
func worker(done chan bool) {
    fmt.Print("working...")
    time.Sleep(time.Second)
    fmt.Println("done")
    done <- true
}
func main() {
    done := make(chan bool, 1)
    go worker(done)
    <-done
}
```

使用  `done` 通道将被用于通知其他协程这个函数已经完成工作，即实现同步。

## 29 通道方向

当使用通道作为函数的参数时，你可以指定这个通道是否为只读或只写。该特性可以提升程序的类型安全。

```go
func ping(pings chan<- string, msg string) {
    pings <- msg
}
func pong(pings <-chan string, pongs chan<- string) {
    msg := <-pings
    pongs <- msg
}
```

## 30 通道选择器

Go 的**选择器**（select）让你可以同时等待多个通道操作。将协程、通道和选择器结合，是 Go 的一个强大特性。

```go
func main() {
    c1 := make(chan string)
    c2 := make(chan string)
    go func() {
        time.Sleep(1 * time.Second)
        c1 <- "one"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        c2 <- "two"
    }()
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-c1:
            fmt.Println("received", msg1)
        case msg2 := <-c2:
            fmt.Println("received", msg2)
        }
    }
}
```

## 31 超时处理

**超时** 对于一个需要连接外部资源，或者有耗时较长的操作的程序而言是很重要的。得益于通道和 `select `，在 Go 中实现超时操作是简洁而优雅的。

```go
c1 := make(chan string, 1)
    go func() {
        time.Sleep(2 * time.Second)
        c1 <- "result 1"
    }()
select {
case res := <-c1:
	fmt.Println(res)
case <-time.After(1 * time.Second):
	fmt.Println("timeout 1")
}
```

## 32 非阻塞通道

常规的通过通道发送和接收数据是阻塞的。然而，我们可以使用带一个 `default` 子句的 `select` 来实现**非阻塞**的发送、接收，甚至是非阻塞的多路 `select`。

```go
messages := make(chan string)
signals := make(chan bool)
select {
case msg := <-messages:
	fmt.Println("received message", msg)
case sig := <-signals:
	fmt.Println("received signal", sig)
default:
	fmt.Println("no activity")
}
```

我们可以在 `default` 前使用多个 `case` 子句来实现一个多路的非阻塞的选择器。这里我们试图在 `messages` 和 `signals` 上同时使用非阻塞的接收操作。

## 33 通道的关闭

关闭一个通道意味着不能再向这个通道发送值了。该特性可以向通道的接收方传达工作已经完成的信息。

```go
func main() {
    jobs := make(chan int, 5)
    done := make(chan bool)
    go func() {
        for {
            j, more := <-jobs // 第二个参数，表示还有没有更多的 job
            if more {
                fmt.Println("received job", j)
            } else {
                fmt.Println("received all jobs")
                done <- true // 通知任务完成
                return
            }
        }
    }()
    for j := 1; j <= 3; j++ {
        jobs <- j
        fmt.Println("sent job", j)
    }
    close(jobs) // 发送三个 job，然后就关闭通道
    fmt.Println("sent all jobs")
    <-done // 等待任务完成
}
```

## 34 通道遍历

```go
func main() {
    queue := make(chan string, 2)
    queue <- "one"
    queue <- "two"
    close(queue)
    for elem := range queue {
        fmt.Println(elem)
    }
}
```
## 35 Timer 定时器

定时器表示在未来某一时刻的独立事件。你告诉定时器需要等待的时间，然后它将提供一个用于通知的通道。这里的定时器将等待 2 秒。

```go
timer1 := time.NewTimer(2 * time.Second)
<-timer1.C // 等待 2 秒
```

`<-timer1.C` 会一直阻塞，直到定时器的通道 `C` 明确的发送了定时器失效的值。如果只需要等待一段时间，使用 `time.Sleep` 就可以了。使用定时器的一个优点是，可以在定时器触发之前将其取消，例如 `timer1.Stop()`。

## 36 Ticker 打点器

定时器是当你想要在未来某一刻执行一次时使用的；打点器则是为你想要以固定的时间间隔重复执行而准备的。

```go
func main() {
    ticker := time.NewTicker(500 * time.Millisecond)
    done := make(chan bool)
    go func() {
        for {
            select {
            case <-done:
                return
            case t := <-ticker.C:
                fmt.Println("Tick at", t)
            }
        }
    }()
    time.Sleep(1600 * time.Millisecond)
    ticker.Stop()
    done <- true
    fmt.Println("Ticker stopped")
}
```

使用一个通道来发送数据。这里我们使用通道内建的 `select`，等待每 500ms 到达一次的值。

## 37 工作池

这一小节，将使用协程和通道实现一个工作池。

```go
package main
import (
    "fmt"
    "time"
)
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs { // 从通道中取任务，然后做任务
        fmt.Println("worker", id, "started  job", j)
        time.Sleep(time.Second)
        fmt.Println("worker", id, "finished job", j)
        results <- j * 2
    }
}
func main() {
    const numJobs = 5
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)
    // 启动了 3 个 worker，初始时是阻塞的，因为还没有传递任务
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }
    // 发送五个任务
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)
    for a := 1; a <= numJobs; a++ {
        <-results
    }
}
```

## 38 WaitGroup 

想要等待多个协程完成，我们可以使用 wait group。

```go
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
	wg.Add(1) // 创建一个协程，就递增一下计数器
	go func() {
		defer wg.Done()
		worker(i) // 将 worker 封装在闭包中，可以确保通知 WaitGroup 此工作线程已完成。
	}()
}
wg.Wait()
```

## 39 速率限制

速率限制是控制服务资源利用和质量的重要机制。基于协程、通道和打点器，Go 可以优雅的支持速率限制。

```go
burstyLimiter := make(chan time.Time, 3)
for i := 0; i < 3; i++ {
	burstyLimiter <- time.Now()
}
go func() {
	for t := range time.Tick(200 * time.Millisecond) {
		burstyLimiter <- t
	}
}()
// 先向通道中写入 3 个时间，然后每隔 200ms 写入一个时间
burstyRequests := make(chan int, 5)
for i := 1; i <= 5; i++ {
	burstyRequests <- i
}
close(burstyRequests)
for req := range burstyRequests {
	<-burstyLimiter // 接收到一个时间，就处理一个请求；开始会处理 3 个，后面就是定时加任务
	fmt.Println("request", req, time.Now())
}
}
```
## 40 原子计数器

使用 `sync/atomic` 包在多个协程中进行**原子计数**。

```go
func main() {
    var ops uint64
    var wg sync.WaitGroup
    for i := 0; i < 50; i++ {
        wg.Add(1)
        go func() {
            for c := 0; c < 1000; c++ {
                atomic.AddUint64(&ops, 1)
            }
            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println("ops:", ops)
}
```

使用 `AddUint64` 来让计数器自动增加，使用 `&` 语法给定 `ops` 的内存地址。
## 41 互斥锁

上一节，了解了使用原子操作来管理简单的计数器。对于更加复杂的情况，可以使用一个互斥量来在 Go 协程间安全的访问数据。

```go
type Container struct {
    mu       sync.Mutex // 互斥锁
    counters map[string]int
}
func (c *Container) inc(name string) {
	c.mu.Lock()
    defer c.mu.Unlock()
    c.counters[name]++
}
```

由于不能复制互斥锁，如果需要传递这个 `struct`，应使用指针完成。

## 42 状态协程 ToDo

## 43 排序

Go 的 `sort` 包实现了内建及用户自定义数据类型的排序功能。排序方法是针对内置数据类型的；且是一个原地排序。

```go
sort.Strings(strs)
sort.Ints(ints)
s := sort.IntsAreSorted(ints) // 返回布尔值
```

## 44 使用函数自定义排序

```go
type byLength []string
func (s byLength) Len() int {
    return len(s)
}
func (s byLength) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}
func (s byLength) Less(i, j int) bool {
    return len(s[i]) < len(s[j])
}
func main() {
    fruits := []string{"peach", "banana", "kiwi"}
    sort.Sort(byLength(fruits)) // 实现上面三个方法就行，实际上只用实现 Less 方法
    fmt.Println(fruits)
    sort.Slice(nums, func(i, j int) bool { // 只需实现比较函数
		return nums[i] >= nums[j]
	})
}
```

我们为该类型实现了 `sort.Interface` 接口的 `Len`、`Less` 和 `Swap` 方法，这样我们就可以使用 `sort` 包的通用 `Sort` 方法了， `Len` 和 `Swap` 在各个类型中的实现都差不多， `Less` 将控制实际的自定义排序逻辑。

## 45 Panic

`panic` 意味着有些出乎意料的错误发生。通常我们用它来表示程序正常运行中不应该出现的错误，或者我们不准备优雅处理的错误。即程序会崩溃了。不可修复的 error 才会触发 panic。

```go
func main() {
    panic("a problem")
    _, err := os.Create("/tmp/file")
    if err != nil {
        panic(err)
    }
}
```

## 46 Recover

Go 通过使用 `recover` 内置函数，可以从 panic 中恢复 recover。 `recover` 可以阻止 `panic` 中止程序，并让它继续执行。

```go
func mayPanic() {
    panic("a problem")
}

func main() {

    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered. Error:", r)
        }
    }()
    panic("a problem")
    fmt.Println("After mayPanic()") // 不会执行到这一步
}
```

## 47 Defer

用于确保程序在执行完成后，会调用某个函数，一般是执行清理工作。比如加锁时解锁、打开文件时关闭文件。

```go
func (c *Container) inc(name string) {
	c.mu.Lock()
    defer c.mu.Unlock()
    c.counters[name]++
}
```

