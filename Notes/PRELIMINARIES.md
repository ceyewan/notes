在本节中，我们介绍了贯穿全文使用的符号表示，并正式定义了 Substring-SSE（Substring-SSE）。为了完整性，我们首先介绍明文子串匹配的语法，然后给出子串 -SSE 的定义。定义基于单客户端、单服务器的设置，其中假设服务器是被动窃听的（即**半诚实模型，semi-honest**）。我们注意到，这是 SSE 文献中最广泛研究的设置（参考 [9, 10, 13, 15]）。

---

## 1 **明文子串匹配**

明文子串匹配的数据库建模如下：

- 设数据库 $\text{DB} = (S_1, \dots, S_N)$ 是一个字符串集合（称为数据库），其字符来自一个共享的字母表 $\Sigma$，即对于所有 $i \in [1, N]$，有 $S_i \in \Sigma^*$。
- 一个明文子串查询 $\text{Query} : \Sigma^* \times (\Sigma^*) \to \{0, 1\}^*$ 接收一个明文查询字符串 $q \in \Sigma^*$ 和明文数据库 $\text{DB} = (S_1, \dots, S_N)$ 作为输入，并输出一个索引对的列表 $((i_k, j_k))_{k=1}^l$，其中满足以下条件：

 $$
  S_{i_k}[j_k, (j_k + |q|)] = q, \quad \forall k = 1, \dots, l.
 $$

我们称查询 $\text{Query}$ 是**正确的**，当且仅当 $\text{Query(DB, q)}$ 能返回数据库 $\text{DB}$ 中所有与查询字符串 $q$ 匹配的索引对。为了简化表示，我们用 $\text{DB(q)}$ 表示 $\text{Query(DB, q)}$。

---

## 2 **子串 -SSE 的语法**

一个**子串可搜索对称加密**（Substring-Searchable Symmetric Encryption, Substring-SSE）方案 $\Pi$ 定义为包含以下三个算法的元组：

$$
\Pi = (\text{Gen}, \text{Setup}, \text{EQuery})
$$

1. **密钥生成**： $$
   \text{sk} \leftarrow \text{Gen}_{\text{Clt}}(1^\lambda)

  $$
   - 这是一个由客户端 $\text{Clt}$ 执行的概率算法。
   - 输入：一个安全参数 $1^\lambda$。
   - 输出：一个秘密密钥 $\text{sk}$。

2. **数据库加密设置**：
  $$

   (\perp, \text{EDB}) \leftarrow [\text{Setup}_{\text{Clt}}(\text{sk}, \text{DB}), \text{Setup}_{\text{Svr}}()]

  $$

   - 这是客户端 $\text{Clt}$ 和服务器 $\text{Svr}$ 之间的交互协议。
   - 客户端输入：秘密密钥 $\text{sk}$ 和明文数据库 $\text{DB}$。
   - 服务器输入：无。
   - 输出：服务器生成一个加密数据库 $\text{EDB}$。

3. **加密查询**：
  $$

   (((i_k, j_k))_{k=1}^l, \perp) \leftarrow [\text{EQuery}_{\text{Clt}}(\text{sk}, q), \text{EQuery}_{\text{Svr}}(\text{EDB})]

  $$

   - 这是客户端 $\text{Clt}$ 和服务器 $\text{Svr}$ 之间的交互协议。
   - 客户端输入：秘密密钥 $\text{sk}$ 和子串查询 $q$。
   - 服务器输入：加密数据库 $\text{EDB}$。
   - 输出：
     - 客户端获得一个响应列表 $((i_k, j_k))_{k=1}^l$，表示查询结果。
     - 服务器无任何输出。

一个子串 -SSE 方案需要满足特定的**正确性**和**安全性**属性（将在后续定义）。

---

## **子串 -SSE 的正确性**

我们称子串 -SSE 方案 $\Pi$ 是**正确的**，如果 $\text{EQuery}$ 能返回正确的子串匹配结果。具体来说，对于任意数据库 $\text{DB}$ 和查询字符串 $q$，在执行以下操作序列时：

1.$\text{sk} \leftarrow \text{Gen}_{\text{Clt}}(1^\lambda)$
2.$(\perp, \text{EDB}) \leftarrow [\text{Setup}_{\text{Clt}}(\text{sk}, \text{DB}), \text{Setup}_{\text{Svr}}()]$
3.$((i_k, j_k))_{k=1}^l, \perp \leftarrow [\text{EQuery}_{\text{Clt}}(\text{sk}, q), \text{EQuery}_{\text{Svr}}(\text{EDB})]$

我们有：
$$

((i_k, j_k))_{k=1}^l = \text{Query(DB, q)}。

$$

---

## **子串 -SSE 的安全性**

我们定义子串 -SSE 的安全性基于**模拟模型**，针对**半诚实对手**的攻击（即服务器遵守协议，但试图从中推断出明文数据库和查询的信息）。我们的定义采用**真实世界 - 理想世界范式**，并严格遵循 SSE 文献中广泛使用的模拟安全定义。

令 $\Pi = (\text{Gen}, \text{Setup}, \text{EQuery})$ 是一个子串 -SSE 方案，假设：
-$\mathcal{A}$：一个有状态的概率多项式时间（PPT）对手；
-$\mathcal{S}$：一个 PPT 模拟器；
-$\mathcal{L} = (\mathcal{L}_{\text{Setup}}, \mathcal{L}_{\text{EQuery}})$：一个有状态的泄漏函数。

我们定义以下两个概率实验：

1. **真实世界实验**$\text{Real}_{\Pi, \mathcal{A}}(1^\lambda)$：
   - 挑战者运行 $\text{Gen}_{\text{Clt}}(1^\lambda)$ 生成一个秘密密钥 $\text{sk}$。
   - 对手 $\mathcal{A}$ 输出一个数据库 $\text{DB}$。
   - 挑战者与对手交互，执行 $(\perp, \text{EDB}) \leftarrow [\text{Setup}_{\text{Clt}}(\text{sk}, \text{DB}), \text{Setup}_{\text{Svr}}()]$，其中挑战者扮演客户端，$\mathcal{A}$ 扮演服务器。
   - 对手 $\mathcal{A}$ 发起多项式次自适应子串匹配查询：每次查询运行 $[\text{EQuery}_{\text{Clt}}(\text{sk}, q), \text{EQuery}_{\text{Svr}}(\text{EDB})]$，其中 $\mathcal{A}$ 自适应选择查询字符串 $q$。
   - 最后，对手 $\mathcal{A}$ 输出一个比特 $b$，作为实验的输出。

2. **理想世界实验**$\text{Ideal}_{\Pi, \mathcal{A}, \mathcal{S}}(1^\lambda)$：
   - 对手 $\mathcal{A}$ 输出一个数据库 $\text{DB}$。
   - 模拟器 $\mathcal{S}$ 和对手交互，执行 $(\perp, \text{EDB}) \leftarrow [\mathcal{S}(\mathcal{L}_{\text{Setup}}(\text{DB})), \text{Setup}_{\text{Svr}}()]$，其中 $\mathcal{S}$ 扮演客户端，$\mathcal{A}$ 扮演服务器。
   - 对手 $\mathcal{A}$ 发起多项式次自适应子串匹配查询：每次查询运行模拟协议 $[\mathcal{S}(\mathcal{L}_{\text{EQuery}}(q)), \text{EQuery}_{\text{Svr}}(\text{EDB})]$，其中模拟器 $\mathcal{S}$ 扮演客户端，$\mathcal{A}$ 扮演服务器。
   - 最后，对手 $\mathcal{A}$ 输出一个比特 $b$，作为实验的输出。

我们称子串 -SSE 方案 $\Pi$ 在自适应选择查询攻击下是 $\mathcal{L}$- 安全的，如果对于任意 PPT 对手 $\mathcal{A}$，存在一个 PPT 模拟器 $\mathcal{S}$，使得以下不等式成立：

$$

\Big| \Pr[\text{Real}_{\Pi, \mathcal{A}}(1^\lambda) = 1] - \Pr[\text{Ideal}_{\Pi, \mathcal{A}, \mathcal{S}}(1^\lambda) = 1] \Big| \leq \text{negl}(\lambda)。

$$

其中 $\text{negl}(\lambda)$ 表示一个关于安全参数 $\lambda$ 的可忽略函数。

## 2.4 **基于推断的泄漏密码分析**

需要注意的是，上述安全性定义非正式地表达了以下事实：一个**半诚实的对手服务器**无法超出泄漏函数 $\mathcal{L} = (\mathcal{L}_{\text{Setup}}, \mathcal{L}_{\text{EQuery}})$ 所描述的范围，了解客户端明文数据库和查询的任何信息。在本文中，我们设计了**查询重构攻击**，其中半诚实对手利用泄漏函数 $\mathcal{L}$ 来恢复客户端的查询。
我们的攻击属于**推断攻击**。更具体地说，我们假设攻击者可以访问一些关于底层明文数据库的辅助统计信息。基于这些数据库的统计信息以及加密查询执行的通信记录，半诚实攻击者试图推断底层的明文查询。

---

## 2.5 **半诚实与恶意腐败**

需要在此指出的是，Chase 和 Shen 在 [14] 中最初声称其 CS 方案在一个更严格的安全模型下是安全的，该模型允许服务器被**恶意腐败**。在上述的安全性定义中，我们仅考虑了服务器的**半诚实腐败**，这是一种更弱的对手模型（除此之外，上述安全性定义与 [14] 中的定义相同）。我们选择在半诚实对手模型下定义子串 -SSE 的安全性，以与我们提出的攻击场景保持一致，该场景假设对手是半诚实的。
