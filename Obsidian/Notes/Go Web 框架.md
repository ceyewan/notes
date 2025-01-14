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



