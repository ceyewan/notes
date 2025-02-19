对象关系映射（ORM，Object Relational Mapping）是一种将面向对象语言中的对象与关系数据库中的数据进行映射的技术。ORM 框架的目标是使开发人员不需要直接编写 SQL 语句，通过对象操作来完成数据库操作。

| 数据库元素     | 面向对象元素         |
| --------- | -------------- |
| 表（Table）  | 类（Class）       |
| 记录（Row）   | 对象（Object）     |
| 字段（Field） | 对象属性（Property） |

下面我们给出一个简单的示例用于辅助了解，其中的 SQL 语句和 ORM 框架代码效果相同：

```go
-- SQL 语句
CREATE TABLE User (Name text, Age integer);  
INSERT INTO User (Name, Age) VALUES ("Tom", 18);  
SELECT * FROM User;

-- ORM 框架代码
type User struct {  
    Name string  
    Age  int  
}  
  
orm.CreateTable(&User{})  
orm.Save(&User{"Tom", 18})  
var users []User  
orm.Find(&users)
```

ORM 框架通过结构体映射数据库，避免了繁琐的 SQL 语句，并且提供了跨数据库平台的通用性。开发者只需要操作对象即可完成数据持久化。

ORM 框架的通用性需要使用 Go 语言的**反射机制**来实现动态获取结构体的元数据。反射可以让我们在运行时获取对象的类型、字段、方法等信息。

```go
func main() {
    user := User{Name: "Alice", Age: 25}
    // 获取 reflect.Value
    value := reflect.Indirect(reflect.ValueOf(&user)）
    fmt.Printf("Value: %v\n", value)
    // 获取 reflect.Type
    typ := value.Type()
    fmt.Printf("Type: %v\n", typ)
    // 遍历结构体字段
    for i := 0; i < value.NumField(); i++ {
        field := value.Field(i) // 获取字段值
        fieldType := typ.Field(i) // 获取字段类型信息
        // 打印字段名称、类型和值
        fmt.Printf("Field Name: %s, Field Type: %s, Field Value: %v\n",
        fieldType.Name, fieldType.Type, field.Interface())
    }
}
```

运行结果如下：

```
Value: {Alice 25}
Type: main.User
Field Name: Name, Field Type: string, Field Value: Alice
Field Name: Age, Field Type: int, Field Value: 25
```

我们可以知道：
- `reflect.ValueOf()` 获取对象的反射值。
- `reflect.Indirect()` 获取指针指向的对象的反射值。
- `reflect.Type()` 获取对象的类型信息，即 `main.User`；
- `reflect.Type.Field(i)` 获取第 i 个字段的信息，返回的是一个 StructField 类型，包含字段名称、类型等信息。

通过 Go 的反射机制，开发人员可以动态地处理结构体，ORM 框架正是依赖反射来自动映射数据库表与对象之间的关系。反射让我们能够在运行时访问类型的结构，进而动态地处理对象的属性。对于 ORM 框架来说，这种机制是其通用性和灵活性的基础。

## 1 database/sql 基础

### 1.1 初识 SQLite

SQLite 是一款轻量级的，遵守 ACID 事务原则的关系型数据库。SQLite 可以直接嵌入到代码中，不需要像 MySQL、PostgreSQL 需要启动独立的服务才能使用。因此，这里我们选择使用它来作为示例。

### 1.2 database/sql 标准库

Go 语言提供了标准库 `database/sql` 用于和数据库的交互。

```go
func main() {
    db, _ := sql.Open("sqlite3", "gee.db")
    defer func() { _ = db.Close() }()
    _, _ = db.Exec("DROP TABLE IF EXISTS User;")
    _, _ = db.Exec("CREATE TABLE User(Name text);")
    result, err := db.Exec("INSERT INTO User(`Name`) values (?), (?)", "Tom", "Sam")
    if err == nil {
        affected, _ := result.RowsAffected()
        log.Println(affected)
    }
    row := db.QueryRow("SELECT Name FROM User LIMIT 1")
    var name string
    if err := row.Scan(&name); err == nil {
        log.Println(name)
    }
}
```

使用 `sql.Open()` 连接数据库，第一个参数是驱动名称，第二个参数是数据库的名称，对于 SQLite 来说，也就是文件名，不存在会新建。返回一个 `sql.DB` 实例的指针。

`Exec()` 用于执行 SQL 语句，如果是查询语句，不会返回相关的记录。所以查询语句通常使用 `Query()` 和 `QueryRow()`，前者可以返回多条记录，后者只返回一条记录。

`Exec()`、`Query()`、`QueryRow()` 接受1或多个入参，第一个入参是 SQL 语句，后面的入参是 SQL 语句中的占位符 `?` 对应的值，占位符一般用来防 SQL 注入。

`QueryRow()` 的返回值类型是 `*sql.Row`，`row.Scan()` 接受1或多个指针作为参数，可以获取对应列(column)的值，在这个示例中，只有 `Name` 一列，因此传入字符串指针 `&name` 即可获取到查询的结果。

### 1.3 实现一个简单的 log 库

这个地方我们实现一个简易的 log 库，支持日志分级、不同分级的日志使用不同颜色区分、显示打印日志代码对应的文件名和行号。

### 1.4 核心部分 Session

这个部分用于实现与数据库的交互，现在我们只实现直接调用 SQL 语句进行交互。

### 1.5 核心结构 Engine

Session 负责与数据库的交互，那交互前的准备工作（比如连接/测试数据库），交互后的收尾工作（关闭连接）等就交给 Engine 来负责了。Engine 是 GeeORM 与用户交互的入口。