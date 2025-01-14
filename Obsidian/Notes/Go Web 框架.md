这是一个使用 Go 的 `net/http` 标准库来实现一个 Web 框架的一个小项目，参考的是 Gin 框架的实现。代码其实都比较简单，这里主要关注的是，如何设计，从而能够将标准库封装为一个框架。
## 1 为什么要框架

`net/http` 提供了基础的 Web 功能，即监听端口，映射静态路由，解析 HTTP 报文。一些 Web 开发中简单的需求并不支持，需要手工实现。
- 动态路由：例如 `hello/:name` 这类的规则。
- 鉴权：分组统一鉴权的能力。
## 2 http.Handler

标准库提供了几种函数，第一种就是监听特定的端口并启动 HTTP 服务；第二个函数就是为特定地址的请求创建处理函数。

```go
http.ListenAndServe(":9999", nil)
http.HandleFunc("/hello", helloHandler)
func helloHandler(w http.ResponseWriter, req *http.Request)
```

第一个函数表示在 9999 端口监听，第二个参数代表处理所有的 HTTP 请求的实例。为 nil 表示使用标准库中的实例处理。（这个参数也是我们基于标准库实现 Web 框架的入口）

第二个参数的类型是一个 Handler 接口，需要实现 ServeHTTP 方法。也就是说，只要传入任何实现了 _ServerHTTP_ 接口的实例，所有的 HTTP 请求，就都交给了该实例处理了。

比如，我们创建一个实例 Engine，并实现了这个接口，就可以使用我们的实例接管所有的 HTTP 请求。

```go
// 具体的处理函数，由用户自己写，参数一定要是这两个 
type HandlerFunc func(http.ResponseWriter, *http.Request)  
// 实现了 ServeHTTP 接口，就可以接管所有的 HTTP 请求
type Engine struct {  
	router map[string]HandlerFunc  
}  
// 创建实例  
func New() *Engine {  
	return &Engine{router: make(map[string]HandlerFunc)}  
}
// 启动一个 HTTP 服务，请求都会交给 Engine 来处理 
func (engine *Engine) Run(addr string) (err error) {  
	return http.ListenAndServe(addr, engine)  
}
// 实现 ServeHTTP 方法，解析请求路径，查找路由映射表，执行对应的处理方法
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {  
	key := req.Method + "-" + req.URL.Path  
	if handler, ok := engine.router[key]; ok {  
		handler(w, req)  
	} else {  
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)  
	}  
}
// 注册处理函数，存储到 router 中
```
## 3 上下文 Context

用户要写 handlerFunc 还是很麻烦，要处理 http.ResponseWriter 和 http.Request。因此这部分将会设计**上下文**（Context）这个结构体，封装 Request 和 Response，并提供对 JSON、HTML 的支持。

我们要构造一个完整的响应，需要考虑消息头(Header)和消息体(Body)，而 Header 包含了状态码(StatusCode)，消息类型(ContentType)等几乎每次请求都需要设置的信息。因此，如果不进行有效的封装，那么框架的用户将需要写大量重复，繁杂的代码，而且容易出错。

```go
type H map[string]interface{} // H 的 key 为 string，value 为 any
  
type Context struct {  
	// origin objects  
	Writer http.ResponseWriter  
	Req    *http.Request  
	// request info  
	Path   string  
	Method string  
	// response info  
	StatusCode int  
}
// 现在要发送一个 json 文件，只需要调用 c.JSON(obj) 就行了
func (c *Context) JSON(code int, obj interface{}) {  
	c.SetHeader("Content-Type", "application/json")  
	c.Status(code)  
	encoder := json.NewEncoder(c.Writer)  
	if err := encoder.Encode(obj); err != nil {  
		http.Error(c.Writer, err.Error(), 500)  
	}  
}
// 发送 html 文件
func (c *Context) HTML(code int, html string) {  
	c.SetHeader("Content-Type", "text/html")  
	c.Status(code)  
	c.Writer.Write([]byte(html))  
}
// 对于 ServeHTTP，我们封装一个 Context，再传参给具体的 handler
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {  
	c := newContext(w, req)  
	engine.router.handle(c)  
}
// 这样一来，具体的 handler 函数就可以这样写了。这里用来两个 kv 对初始化 H
func handler(c *gee.Context) {  
	c.JSON(http.StatusOK, gee.H{  
		"username": c.PostForm("username"),  
		"password": c.PostForm("password"),  
	})
```

对于框架来说，还需要支撑额外的功能，比如解析动态路由，需要存放参数的信息；比如支持中间件，中间件产生的信息和调用顺序也要放一个位置。Context 会随着一个请求的出现而产生，请求的结束而销毁。和当前请求强相关的信息都要由 Context 承载。

## 4 前缀树路由 Router

实现动态路由解析。之前是使用一个简单的 map 结构来存储路由表，键是请求方法+请求路径，值是处理函数。但是现在要处理类似于 /hello/:name 这样的动态路由，我们可以使用前缀树（Trie 树来实现）。

网络上也有很多开源的动态路由实现，比如 gorouter 就支持嵌入正则表达式、Gin 在早期是直接使用的的 httprouter，后来是自己实现了一个版本。

## 5 分组控制

分组控制(Group Control)是 Web 框架应提供的基础功能之一。所谓分组，是指路由的分组。如果没有路由分组，我们需要针对每一个路由进行控制。但是真实的业务场景中，往往某一组路由需要相似的处理。比如中间件，有些路由需要认证，而另外一些路由不需要认证。

那么，一个 Group 需要具备的属性有前缀（用于确定分组），并且中间件也是应用在分组上。此外，Group 对象还需要有访问 Router 的能力，为了方便，可以直接访问 engine。并且 Group 是一个递归结构。

```go
RouterGroup struct {  
	prefix      string  
	middlewares []HandlerFunc // support middleware  
	parent      *RouterGroup  // support nesting  
	engine      *Engine       // all groups share a Engine instance  
}
// Engine 抽象为最顶层的分组，也就是 Engine 也拥有 RouterGroup 的能力
Engine struct {  
	*RouterGroup  // 匿名嵌套结构体
	router *router  
	groups []*RouterGroup // store all groups  
}
// 接下来无论有没有分组，注册函数都是在分组上注册的
```

> **匿名嵌套结构体**：实现了一种轻量级的**继承**机制，允许嵌套结构体的字段和方法直接提升为外部结构体的成员。嵌套的匿名字段的所有字段和方法会自动提升到外层结构体 Engine，使得外层结构体可以直接访问内层结构体的方法或字段，就像它们属于外层结构体一样。
## 6 中间件 Middleware

中间件是位于请求和响应处理流程中的逻辑组件，用于对请求进行处理、修改、或执行特定功能（如认证、日志记录）。它是 Web 框架中一个重要的设计模式。
1. **预处理请求**：在请求被路由到目标处理函数之前，对请求数据进行检查、修改、或者添加额外的上下文信息。
2. **后处理响应**：在响应被返回给客户端之前，对响应数据进行修改、格式化、或者记录。

中间件和路由映射的 Handler 一致，处理的输入是 Context 对象。插入点是框架接收到请求初始化 Context 对象之后，允许用户使用自己定义的中间件做一些额外的处理，例如记录日志等，对 Context 进行二次加工。另外，通过调用 `(*Context).Next()` 函数，中间件可以等待用户自己定义的 Handler 处理结束之后，再做一些额外的操作。`c.Next()` 表示等待执行其他的中间件或用户的 `Handler`。

```go
func Logger() HandlerFunc {  
	return func(c *Context) {   
		t := time.Now() // handler 之前执行
		c.Next() // handler
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))  
	}  
}
```

中间件信息也保留在 Context 中，即在 Context 中添加一个函数数组，表示中间件函数，并且为 Context 实现 Next() 方法，调用下一个处理函数。

```go
type Context struct {  
	// ...
	// middleware  
	handlers []HandlerFunc  // 中间件+handler
	index    int  
}
func (c *Context) Next() {  
	c.index++  
	s := len(c.handlers)  
	for ; c.index < s; c.index++ {  
		c.handlers[c.index](c)  
	}  
}
// 添加中间件，包括父节点的中间件
func (group *RouterGroup) Use(middlewares ...HandlerFunc) {  
	group.middlewares = append(group.middlewares, middlewares...)  
}
```

在接收到一个具体的请求时，要判断该请求适用于哪些中间件，得到中间件列表后，赋值给 `c.handlers`。在路由匹配时，还需要将得到的 Handler 添加到 `c.handlers` 列表中，执行 `c.Next()`。也就是说，可以认为中间件和 handler 函数差不多。一个函数调用一个 c.Next()，直到所有的处理函数都执行完成。

## 7 错误恢复

Go 语言中，比较常见的错误处理方法是返回 error，由调用者决定后续如何处理。但是如果是无法恢复的错误或者手动触发 panic，都会中止当前执行的程序，退出。但是 panic 导致的程序中止在退出之前，会先处理完当前协程上已经 defer 的任务，执行完成之后再退出，类似于 `try...catch`。

Go 语言还提供了 recover 函数，可以避免因为 panic 发生而导致整个程序终止，recover 函数只在 defer 中生效。

```go
// hello.go  
func test_recover() {  
	defer func() {  
		fmt.Println("defer func")  
		if err := recover(); err != nil {  
			fmt.Println("recover success")  // 输出
		}  
	}()  
	arr := []int{1, 2, 3}  
	fmt.Println(arr[4])  		// 不输出
	fmt.Println("after panic")  // 不输出
}  
  
func main() {  
	test_recover()  
	fmt.Println("after recover")  // 输出
}
```

对于一个 Web 框架，错误处理机制是非常必要的。因此，需要实现 Recovery 中间件，这个也很简单，只需要使用 defer 挂载上错误恢复函数，在这个函数中调用 recover()，捕获 panic 然后恢复程序。也容易知道，Recover 应该是第一个中间件，这样才能恢复所有中间件和处理函数的错误。

```go
func Recovery() HandlerFunc {  
	return func(c *Context) {  
		defer func() {  
			if err := recover(); err != nil {  
				message := fmt.Sprintf("%s", err)  
				log.Printf("%s\n\n", trace(message))  
				c.Fail(http.StatusInternalServerError, "Internal Server Error")  
			}  
		}()  
		c.Next()  
	}  
}
```