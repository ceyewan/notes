# Web 框架 Gee 教程

这是

框架是是一种提供 **预定义结构和工具** 的软件开发工具，用于简化和标准化应用程序的开发流程。以 Gin 为例，它是 Go 语言中一个高性能、轻量级的 Web 框架，常用于构建 RESTful API。

Go 标准库提供了基础的功能，例如 net/http 用于构建 Web 服务器，但是使用标准库需要手动处理许多低级细节，例如路由解析、参数绑定、错误处理等。而 Gin 内置了很多功能，大幅减少了重复劳动。

`net/http` 提供了基础的 Web 功能，即监听端口，映射静态路由，解析 HTTP 报文。一些 Web 开发中简单的需求并不支持，需要手工实现。

-   动态路由：例如 `hello/:name`，`hello/*` 这类的规则。
-   鉴权：没有分组/统一鉴权的能力，需要在每个路由映射的 handler 中实现。
-   模板：没有统一简化的 HTML 机制。

## 前置知识 (http. Handler)

