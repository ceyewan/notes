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

Go 支持指针，允许在程序中通过**引用传递**来传递值和数据结构。通过 `&i` 语法来取得 `i` 的内存地址，即指向 `i` 的指针。