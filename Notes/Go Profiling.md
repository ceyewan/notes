## 1 **概述**

- **Profiling 的定义**：Profiling 是收集程序运行时的详细统计数据（如 CPU 使用率、内存分配等），用于优化性能。与 Benchmark 不同，Profiling 收集的是整个程序的性能数据，而非特定函数。
- **Profiling 的意义**
  - 诊断性能问题：定位程序性能下降的原因。
  - 提升代码质量：通过分析优化程序设计。
  - Go 的高性能可能因糟糕的设计而受限，Profiling 可帮助解决问题。

---

## 2 **CPU Profiling**

- 使用 `runtime/pprof` 包创建 CPU Profile

    ```go
    f, err := os.Create("profile.pb.gz")
    if err != nil {
        log.Fatal(err)
    }
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    // functions
    ```

  - 运行程序并生成 `profile.pb.gz` 文件，可以使用 `protoc` 工具来解码查看。
  - 使用 `pprof` 工具分析：

    ```bash
    $ go tool pprof yourBinary profile.pb.gz
    ```

- **关键概念**
  - **Call Stack**：程序当前运行的函数列表，按调用顺序排列。
  - **CPU 时间**：分为用户时间（执行用户代码）和系统时间（执行系统调用）。
  - **Samples**：Profiling 时定期采集的测量数据，包括调用栈和 CPU 时间。

- **pprof 命令行工具**
  - 常用命令：
    - `top`：显示 CPU 时间占用最多的函数。
    - `traces`：显示调用栈及其对应的 CPU 时间。
    - `-web`：生成 SVG 格式的可视化图表。

---

## 3 **代码优化技术**

- **常见优化方法**
  1. **删除无用代码**：清理未使用的变量或旧代码片段。
  2. **循环退出优化**：找到目标后立即退出循环，避免多余的迭代。
  3. **循环不变代码外提**：将循环中不变的计算移到循环外部。
  4. **循环合并**：将独立的、条件相同的循环合并为一个循环。
  5. **常量折叠**：将固定计算结果用常量替代，减轻运行时计算负担。

- **优化示例**
  - **循环退出优化**：

    ```go
    for _, xr := range exchangeRates {
        if xr.HotelId == hotelId {
            rate = xr.RateUSD
            break
        }
    }
    ```

  - **循环不变代码外提**：

    ```go
    var costs = 950722 + 12*uint32(len(s))
    for i := 0; i < len(s); i++ {
        turnover += s[i] - costs
    }
    ```

---

## 4 **内存 Profiling**

- **如何创建内存 Profile**
  - 使用 `pprof.WriteHeapProfile` 保存内存 Profile：

    ```go
    f, err := os.Create("mem_profile.pb.gz")
    if err != nil {
        log.Fatal(err)
    }
    runtime.GC() // 触发垃圾回收，确保统计数据准确
    pprof.WriteHeapProfile(f)
    defer f.Close()
    ```

  - 使用 `pprof` 工具分析：

    ```bash
    $ go tool pprof yourBinary mem_profile.pb.gz
    ```

- **内存统计类型**
  - `inuse_space`：当前使用的内存大小。
  - `alloc_space`：累计分配的内存大小（包括已释放的部分）。
  - `inuse_objects`：当前分配的对象数量。
  - `alloc_objects`：累计分配的对象数量（包括已释放的部分）。

---

## 5 **关键知识点总结**

- Profiling 是性能优化的基础工具，帮助定位程序瓶颈。
- 使用 `runtime/pprof` 和 `pprof` 工具可以轻松生成和分析性能数据。
- 常见优化技术包括删除死代码、优化循环逻辑、使用哈希表等。
- 内存 Profiling 可用于分析内存分配和垃圾回收的性能影响。
- 代码优化需在性能与可维护性之间权衡。
