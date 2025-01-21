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

