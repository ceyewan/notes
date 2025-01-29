## 1 什么是分布式系统

1. 互联网、云、客户端、数据中心、服务器
2. 多个计算机、网络数据包交互

## 2 为什么是分布式系统

1. 连接物理上分散的机器
2. 用户之间共享资源
3. 并行处理提升容量
4. 容错性 tolerate faults
5. 实现安全性
## 3 分布式系统历史

- 局域网出现，AFS，DNS，
- 数据中心兴起、大型网站（网页搜索、购物）
- 云计算 
## 4 Challenges

- 大量并发组件
- 必须应对部分故障（网络分区）

## 5 为什么 824

- 有趣，挑战但是有解决方案
- 现实世界应用广泛
- 活跃的研究

## 6 核心结构

- 讲座；big ideas，papers；case study；课前阅读
- Labs，编程实验，mapreduce；raft（复制）；Replicate KV；shared kv  
- exams，两次考试

## 7 视角

- 基础设施，支持应用程序的基础设计
	- 分布式存储、计算（mapreduce）、通信（RPC）。抽象 

## 8 主要话题

- 容错（可用性 availability、可恢复性 recoverability）
- 一致性
- 性能；吞吐量、低延迟
- 实现 implementation
## 9 mapreduce

- 背景，谷歌
- 目标，让非专家也可以轻松开发分布式函数
- map + reduce
- 适配大数据分析
- 一堆输入