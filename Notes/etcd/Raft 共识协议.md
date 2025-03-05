# Raft 共识协议

## 1 Raft 协议概述

Raft 协议旨在提供一种易于理解和实现的分布式一致性算法，强调**强一致性**（Strong Consistency）而非最终一致性（Eventual Consistency）。其核心目标是通过 **分解问题（Leader 选举、日志复制、安全性）** 来简化协议设计。

- **强一致性**：所有节点在任何时刻看到的数据都是一致的。
- **最终一致性**：系统在某个时间点后，所有节点会达成一致，但在此之前可能存在不一致。

核心特性：

1. **Leader 选举**：通过选举机制确定唯一的 Leader 节点。
2. **日志复制**：Leader 将客户端请求的日志复制到所有 Follower 节点。
3. **安全性**：确保系统在各种异常情况下仍能保持一致。

Raft 采用强 Leader 机制，所有客户端请求都由 Leader 处理，简化了日志复制的复杂性。

---

## 2 核心概念

### 2.1 节点角色

1. **Leader**：处理所有客户端请求，负责日志复制。
2. **Follower**：被动接收 Leader 的日志，响应心跳和日志复制请求。
3. **Candidate**：在选举过程中，Follower 转换为 Candidate 角色，发起投票请求。

### 2.2 角色转换流程

- Follower → Candidate：接收 Leader 心跳包超时触发，范围内随机超时。
- Candidate → Leader：获得多数派投票，即票数过半。
- Leader → Follower：发现更高 Term 的 Leader，意味着自己网络宕机已经选举了新的 Leader。

### 2.3 任期（Term）

- **逻辑时钟作用**：用于区分不同 Leader 的任期，防止过时 Leader 干扰。
- **任期递增规则**：每次选举开始时，Candidate 自增 Term 并发起投票请求。

### 2.4 心跳机制

- **维持 Leader 权威**：Leader 定期发送心跳（AppendEntries RPC）以维持其权威。
- **超时触发选举**：Follower 在未收到心跳时，超时后转换为 Candidate 并发起选举。

---

## 3 Leader 选举

Leader 选举的目的是在系统中选出一个唯一的 Leader，以确保系统的正常运行。以下是 Leader 选举的详细步骤：

### 3.1 初始状态

- 所有节点启动时都是 Follower。
- 每个 Follower 维护一个随机的选举超时时间（election timeout），通常为 150ms 到 300ms 之间。如果在超时时间内没有收到来自 Leader 的心跳消息，Follower 会认为 Leader 失效，并转变为 Candidate。

### 3.2 Follower 转变为 Candidate

- 当 Follower 的选举超时时间到期时，它会转变为 Candidate，并开始发起选举。
- Candidate 会递增自己的任期号（term），term 是一个全局单调递增的整数，用于标识选举的轮次。
- Candidate 会给自己投一票，然后向其他所有节点发送 **RequestVote RPC**，请求它们为自己投票。RequestVote RPC 包括以下字段：
    - **Term**：Candidate 的当前任期号（term）。
    - **candidateId**：发起选举的 Candidate 节点的唯一标识符（ID）。
    - **lastLogIndex**：Candidate 日志中最后一个日志条目的索引（用于说明是否是最新）。
    - **lastLogTerm**：Candidate 日志中最后一个日志条目的任期号（term）。

### 3.3 投票过程

- 其他节点收到 RequestVote RPC 后，会根据以下条件决定是否投票：
    1. **任期号**：如果 Candidate 的 term 小于当前节点的 term，则拒绝投票。
    2. **日志完整性**：如果 Candidate 的日志不如当前节点的日志新（即 Candidate 的日志条目较少或 term 较小），则拒绝投票。
    3. **是否已经投票**：如果当前节点在当前 term 中已经投票给其他 Candidate，则拒绝投票。
- 如果满足条件，节点会投票给 Candidate，并重置自己的选举超时时间。

### 3.4 选举结果

- Candidate 会等待其他节点的投票响应。如果 Candidate 收到了**大多数节点**（即超过半数）的投票，它就会转变为 Leader。
- 一旦成为 Leader，它会立即向所有节点发送**心跳消息**（AppendEntries RPC），以宣告自己的领导地位，并阻止其他节点发起新的选举。旧 Leader 网络恢复收到一个比自己当前 term 更大的消息，它会更新自己的 term，并转变为 Follower。
- 如果 Candidate 没有收到大多数节点的投票，它会继续等待，直到选举超时时间再次到期，然后重新发起选举。

### 3.5 选举失败与重新选举

- 如果多个 Candidate 同时发起选举，可能会导致没有 Candidate 获得大多数投票，这种情况称为**选举分裂**（split vote）。
- 当选举分裂发生时，所有 Candidate 都会超时，并重新发起选举。为了避免再次发生选举分裂，Raft 使用随机化的选举超时时间来减少冲突的可能性。

---

## 4 日志复制

### 4.1 日志结构

在 Raft 中，每个节点维护一个日志（Log），日志由一系列日志条目（Log Entry）组成。每个日志条目包含以下信息：

- **Term**：日志条目创建时的 Leader 任期号。
- **Index**：日志条目在日志中的位置（索引）。
- **Command**：客户端请求的操作或命令。

日志条目按照顺序追加，并且每个日志条目都有一个唯一的索引和 term。

### 4.2 日志复制的流程

日志复制的过程由 Leader 协调，确保所有 Follower 的日志与 Leader 的日志保持一致。以下是日志复制的详细步骤：

#### 4.2.1 客户端请求

- 客户端向 Leader 发送一个请求（例如一个状态机命令）。
- Leader 将该请求作为一个新的日志条目追加到自己的日志中。

#### 4.2.2 日志复制到 Follower

- Leader 通过 **AppendEntries RPC** 将新的日志条目发送给所有 Follower。
- AppendEntries RPC 包含以下信息：
    - **Leader 的 term**：用于 Follower 验证 Leader 的合法性。
    - **PrevLogIndex**：新日志条目前一个日志条目的索引。
    - **PrevLogTerm**：新日志条目前一个日志条目的 term。
    - **Entries**：要复制的日志条目（可能是一个或多个）。
    - **LeaderCommit**：Leader 已提交的日志条目的最高索引。

#### 4.2.3 Follower 处理 AppendEntries RPC

- Follower 收到 AppendEntries RPC 后，会进行以下检查：
    1. **Term 检查**：如果 Leader 的 term 小于 Follower 的当前 term，Follower 会拒绝请求。
    2. **日志一致性检查**：Follower 会检查自己的日志中是否存在与 PrevLogIndex 和 PrevLogTerm 匹配的日志条目。如果不匹配，Follower 会拒绝请求。
- 如果检查通过，Follower 会将新的日志条目追加到自己的日志中，并返回成功响应。

#### 4.2.4 Leader 提交日志

- Leader 收到大多数 Follower 的成功响应后，会将该日志条目标记为**已提交**（committed）。
- Leader 更新自己的 **commitIndex**，表示已提交的日志条目的最高索引。
- Leader 在后续的 AppendEntries RPC 中会携带新的 commitIndex，通知 Follower 提交相应的日志条目。

#### 4.2.5 应用到状态机

- Leader 和 Follower 将已提交的日志条目应用到本地状态机中，执行日志条目中的命令。
- 一旦日志条目被应用到状态机，客户端请求的操作就完成了。

### 4.3 一致性保证

Raft 通过以下机制确保日志的一致性：

#### 4.3.1 日志匹配特性（Log Matching Property）

Raft 保证以下两条特性：

1. **相同索引和 term 的日志条目包含相同的命令**。
2. **如果两个日志条目在某个索引位置具有相同的 term，那么它们之前的所有日志条目也相同**。

这些特性确保了所有节点的日志在提交后是一致的。

#### 4.3.2 日志冲突处理

如果 Follower 的日志与 Leader 的日志不一致（例如由于网络分区或节点崩溃），Leader 会通过 AppendEntries RPC 强制覆盖 Follower 的日志：

- Leader 会逐步回退 Follower 的日志，找到最后一个与 Leader 日志匹配的条目。
- Leader 会从该位置开始，将自己的日志条目复制到 Follower。

---
