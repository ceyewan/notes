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

## 2 Session

Session 主要复制与数据库会话和 CRUD 操作。

```go
// Session 是会话管理的主要结构，包含会话的所有操作
type Session struct {
db *sql.DB // 数据库连接
sql strings.Builder // sql 用于拼接 SQL 语句
sqlVars []interface{} // sqlVars 用于存储 SQL 语句中的参数
dialect dialect.Dialect // dialect 记录了该 Session 所使用的数据库方言
refTable *schema.Schema // refTable 记录 Model 对应的表结构
clause clause.Clause // clause 是记录 SQL 语句中的各种子句
tx *sql.Tx // tx 提供事务支持，如果 tx 不为 nil，则执行所有操作都在事务中
}
```

首先，Session 封装了 Exec、QueryRow、QueryRows 这三个方法，分别调用标准库里的这三个函数，使用 Seesion.sql 作为 SQL 语句，Seesion.sqlVars 作为 SQL 语句中的参数。

> [!NOTE]
> Exec 方法用于执行不需要返回行的 SQL 语句，例如 INSERT、DELETE、UPDATE 等；返回值 result 是 sql.Result 类型，用于返回执行结果（受影响的行数等）。
> QueryRow 方法用于执行需要返回单行结果的 SQL 查询，例如 SELECT 查询，返回一个 \*sql.Row 对象，表示查询结果的单行记录。

### 2.1 事务的实现

数据库事务(transaction)是访问并可能操作各种数据项的一个数据库操作序列，这些操作要么全部执行,要么全部不执行，是一个不可分割的工作单位。事务由事务开始与事务结束之间执行的全部数据库操作组成。

> [!NOTE] ACID
> 原子性(Atomicity)：事务中的全部操作在数据库中是不可分割的，要么全部完成，要么全部不执行。
> 一致性(Consistency): 几个并行执行的事务，其执行结果必须与按某一顺序 串行执行的结果相一致。
> 隔离性(Isolation)：事务的执行不受其他事务的干扰，事务执行的中间结果对其他事务必须是透明的。
> 持久性(Durability)：对于任意已提交事务，系统必须保证该事务对数据库的改变不被丢失，即使数据库出现故障。

在 SQLite 中，`BEGIN` 开启事务，`COMMIT` 提交事务，`ROLLBACK` 回滚事务。任何一个事务，均以 `BEGIN` 开始，`COMMIT` 或 `ROLLBACK` 结束。Go 标准库 database/sql 也提供了支持事务的接口。

调用 `db.Begin()` 得到 `*sql.Tx` 对象，使用 `tx.Exec()` 执行一系列操作，如果发生错误，通过 `tx.Rollback()` 回滚，如果没有发生错误，则通过 `tx.Commit()` 提交。

我们发现，一般是用 db 来调用的，但是在事务中是是用 tx 来调用，因此，我们将他封装一下，当 `tx` 不为空时，则使用 `tx` 执行 SQL 语句，否则使用 `db` 执行 SQL 语句。这样既兼容了原有的执行方式，又提供了对事务的支持。

```go
// 抽象出一个接口 CommonDB，包含 Query、QueryRow、Exec 三个方法
type CommonDB interface {
    Query(query string, args ...interface{}) (*sql.Rows, error)
    QueryRow(query string, args ...interface{}) *sql.Row
    Exec(query string, args ...interface{}) (sql.Result, error)
}

// 该接口的实现有 *sql.DB 和 *sql.Tx
var _ CommonDB = (*sql.DB)(nil)
var _ CommonDB = (*sql.Tx)(nil)

// DB 返回 *sql.DB 对象
func (s *Session) DB() CommonDB {
    if s.tx != nil {
        return s.tx
    }
    return s.db
}
```

这样，对于 Exec 方法，我们既不用 db.Exec() 也不用 tx.Exec() 而是使用 DB().Exec()。

接着，我们封装一个 Transaction 方法

```go
// 使用一个这种接口函数封装事务内的操作
type TxFunc func(*session.Session) (interface{}, error)  

// 用于执行一个事务，无论事务是否失败，可以通过 defer 中的回滚确保事务的正确结束
// s.Begin()、s.Rollback()、s.Commit() 简单封装了一下标准库函数
func (engine *Engine) Transaction(f TxFunc) (result interface{}, err error) {  
	s := engine.NewSession()  
	if err := s.Begin(); err != nil {  
		return nil, err  
	}  
	defer func() {  
		if p := recover(); p != nil {  
			_ = s.Rollback()  
			panic(p) // 在回滚之后继续抛出 panic
		} else if err != nil {  
			_ = s.Rollback() // 回滚
		} else {  
			err = s.Commit() // 提交
		}  
	}()  
	return f(s)  
}
```
## 3 Clause

这部分主要负责 SQL 语句的生成，因为每个函数的

## 4 Dialect

## 5 Schema

用于实现对象（object）和表（table）之间的映射关系，是 ORM 的核心部分之一。

```go
// Field 表示数据库表的一列
type Field struct {
    Name string // 列名
    Type string // 列的数据类型
    Tag string // 列的额外信息（标签）
}

// Schema 表示数据库中的一张表
type Schema struct {
    Model interface{} // 表对应的对象
    Name string // 表名
    Fields []*Field // 表的所有列
    FieldNames []string // 表的所有列名
    fieldMap map[string]*Field // 列名到 Field 对象的映射
}
```

这个部分主要函数有三个，GetField() 可以根据列名获取 Field 对象；Parse() 解析对象，创建 Schema 对象。