## 1 火山模型

**火山模型（Volcano Model）** 是数据库查询执行的一种经典模型，它通过一种统一的迭代器接口（Iterator Interface）实现查询计划的执行，是许多传统关系型数据库（如 MySQL、PostgreSQL）的基础执行引擎设计。

火山模型的核心是 **基于迭代器的流水线执行**：
- 每个查询操作符（如 Scan、Filter、Join、Aggregation）被抽象为一个独立的**迭代器**。
- 迭代器通过 `next()` 方法逐行（Row-at-a-Time）向上层返回数据，形成类似 " 火山喷发 " 的自顶向下调用链。

每个操作符的迭代器需要实现以下三个核心方法：
1. **`open()`**：初始化操作符（例如打开表、初始化内存结构）。
2. **`next()`**：从子节点获取下一行数据，处理后返回给父节点。
3. **`close()`**：释放资源（例如关闭文件、清理内存）。

执行流程：
- **查询计划树**：查询被编译为由多个操作符组成的树形结构（查询计划树）。
- **自顶向下的调用链**：从根节点开始，递归的调用其子节点的 `next()`。

这种模型的 CPU 效率低，需要逐行处理和大量的函数调用开销，无法利用 CPU 的缓存局部性和 SIMD 指令，不适应批量处理和向量化计算。因此，出现了一些替代模型，例如向量化模型。

> [!NOTE] OLTP and OLAP
> **OLTP（Online Transaction Processing，联机事务处理）**，支持高并发、低延迟的**实时事务操作**，例如增删改查（CRUD）。
> **OLAP（Online Analytical Processing，联机分析处理）**，支持复杂的**数据分析与决策**，例如统计报表、数据挖掘。

## 2 SQL 执行流程

