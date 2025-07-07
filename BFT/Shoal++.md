这篇文章题为《Shoal++: High Throughput DAG BFT Can Be Fast and Robust!》，由Balaji Arun、Zekun Li、Florian Suri-Payer、Sourav Das和Alexander Spiegelman共同撰写，发表于2025年。文章提出了一种新型的基于有向无环图（DAG）的拜占庭容错（BFT）共识协议Shoal++，旨在解决现有DAG-BFT协议在高吞吐量和低延迟之间的权衡问题。

### 主要内容：
1. **研究背景**：
   - 传统的BFT协议（如PBFT）优化了延迟，但在高吞吐量场景下表现不佳。
   - 现有的DAG-BFT协议（如Bullshark和Shoal）虽然实现了高吞吐量，但延迟较高（平均10.5个消息延迟）。

2. **Shoal++的创新**：
   - **更快的锚点提交规则**：通过优化Bullshark的直接提交规则，将锚点提交延迟从6个消息延迟减少到4个。
   - **动态锚点调度**：尝试将更多节点动态标记为锚点，减少非锚点节点的等待时间。
   - **并行DAG实例**：运行多个交错的DAG实例，减少事务排队延迟，提高吞吐量。

3. **性能提升**：
   - Shoal++将端到端共识延迟从Shoal的10.5个消息延迟降低到4.5个，同时保持高吞吐量。
   - 实验表明，Shoal++在无故障情况下实现了亚秒级延迟，并在高负载下支持每秒14万笔交易（tps）。

4. **实验评估**：
   - 在100个全球分布的节点上，Shoal++的延迟显著低于Bullshark、Shoal和Mysticeti，同时吞吐量优于传统的Jolteon协议。
   - 在故障和网络不稳定的情况下，Shoal++表现出更强的鲁棒性。

5. **理论证明**：
   - 文章通过形式化证明验证了Shoal++的安全性和活性，确保其在部分同步网络模型下的正确性。

### 结论：
Shoal++通过创新的设计显著降低了DAG-BFT协议的延迟，同时保持了高吞吐量和鲁棒性，为现代分布式系统（如区块链）提供了一种高效的共识解决方案。




## 典型的 **DAG-based BFT 共识协议中 proposal → vote → certificate → DAG 构建的流程**。


### 📋 协议步骤

#### **Step 1: 提议 (Proposal)**

* 一个节点（replica）在 **第 r 轮** 发起提议，广播一个签名的提议：

  $$
  P := \langle r, B, edges \rangle_\sigma
  $$

  其中：

  * $r$：当前轮次。
  * $B$：一个交易批次（batch of transactions）。
  * $edges$：来自上一轮 $r-1$ 的 $n-f$ 个唯一的提议证书（proposal certificates），用来构建 DAG 边。

---

#### **Step 2: 投票 (Vote)**

* 每个副本节点收到提议后，检查：

  * 自己是否已经收到该提议者在第 $r$ 轮的其他提议。
  * 如果已经收到别的提议，就忽略（避免双重投票）。
  * 否则，计算提议 $P$ 的 hash 值 $d$，向提议者发送一个签名投票：

  $$
  S := \langle r, d \rangle_\sigma
  $$

---

#### **Step 3: 聚合证书 (Certificate)**

* 提议者等待收集到 $n-f$ 个对 $P$ 的匹配投票签名。
* 然后把这些签名聚合成一个证书：

  $$
  C := \langle r, d, \{S\} \rangle
  $$
* 将这个证书 $C$ 广播给所有副本。

---

#### **Step 4: 更新 DAG**

* 其他副本节点收到一个有效的 proposal certificate 后：

  * 把它当作一个新节点加入本地的 DAG 中。
  * 节点包括提议的内容、对应的证书以及与 $edges$ 中提到的父节点相连。

---

### 📝 总结

| 阶段     | 参与者  | 行为           | 消息                                    |
| ------ | ---- | ------------ | ------------------------------------- |
| 1️⃣ 提议 | 提议者  | 广播一个交易提议及其依赖 | $P = \langle r,B,edges\rangle_\sigma$ |
| 2️⃣ 投票 | 所有副本 | 检查 & 投票      | $S = \langle r,hash(P)\rangle_\sigma$ |
| 3️⃣ 证书 | 提议者  | 收集投票 & 聚合成证书 | $C = \langle r,hash(P),\{S\}\rangle$  |
| 4️⃣ 更新 | 所有副本 | 将节点加入 DAG    | 更新本地 DAG                              |

---


## 共识层？？？

### 🧱 1. **DAG 本身是什么**

* **DAG 只是一个强大的 mempool**

  * 它通过网络不断扩展，保证高可用性和高吞吐。
  * 每个节点都是经过证书认证的，确保不会出现双花、不会被丢失。
  * 但：DAG 本身并没有共识（即全局交易顺序）。

---

### ⬆️ 2. **为什么需要共识层**

* 为了在全网达成一个一致的顺序，需要一个 **共识层**。
* 共识层有两种实现方式：
  1️⃣ 外部黑盒共识：单独跑一个共识协议来决定 DAG 的“快照”。

  * 例如：**Narwhal + Hotstuff**
    2️⃣ 嵌入式共识：直接利用 DAG 自身的结构和消息达成共识，不需要额外的消息。
  * 例如：**Tusk、Bullshark、Shoal**
* 本文关注的是第二种方式：**嵌入式 DAG-BFT**，因为它更高效。

---

### ⚓ 3. **嵌入式共识的核心思想：Anchor**

* 在 DAG 里选出一些特殊节点叫做 **anchor（锚点）**。

  * anchor 类似传统 BFT 里的 “leader”。
  * DAG 中的边则被解释成对 leader 的投票。
* **提交逻辑**：

  * 当一个 anchor 被 **f+1 个后续节点** 直接链接时，就可以提交（Direct Commit Rule）。
  * 提交后，它的全部因果历史也随之提交。
  * 提交顺序可以通过拓扑排序等确定性方法推导出来。

---

### 🧭 4. **Anchor 的选择规则**

* **Bullshark**：

  * 锚点提前按照轮次确定（类似 round-robin），每隔一轮选择一个锚点候选。
  * 每个锚点候选在轮 $r$ 被 $f+1$ 个轮 $r'>r$ 的节点链接后提交。
  * 提交后，所有后续锚点必须指向它，保证全局一致的锚点序列。
  * 安全性保障：
    1️⃣ 锚点经过认证，无双重提议。
    2️⃣ 后续节点必须链到前面的锚点，使锚点“稳固”。

---

### 🧹 5. **Indirect Commit Rule**

* 如果一个锚点没有直接在某个副本的本地 DAG 里被提交，它也能通过遍历锚点因果历史被提交。
* 这样，即便本地 DAG 落后于其他节点，最终还是会和全网达成一致。
* 确保即便某些提交节点暂时未被直接确认，也能最终达成一致性。

---

### ⏳ 6. **Bullshark 的问题**

* 因为每个节点的出边只有 $2f+1$，而提交锚点需要 $f+1$ 个链接。
* 在最坏情况下，一个锚点候选可能被不断跳过，提交进度变慢。
* 依赖保守的超时时间来保证活性，这导致延迟大。

---

### ⚡ 7. **Shoal 的优化**

* 相比 Bullshark，Shoal 提供两个关键改进来降低延迟、提高活性：
  1️⃣ 动态解释锚点：
  \- 不固定锚点只能每隔一轮产生，可以每一轮都尝试选出锚点，加快提交。
  2️⃣ 锚点声誉机制：
  \- 优先选择连接快、可靠的节点作为锚点候选，提高被链接的概率，减少依赖超时。

---

## 📝 总结对比

| 特性     | Bullshark            | Shoal        |
| ------ | -------------------- | ------------ |
| 锚点选择   | round-robin，每隔一轮一个锚点 | 每轮都可以选锚点     |
| 锚点提交条件 | $f+1$ 个后续链接          | 相同           |
| 锚点声誉机制 | 无                    | 有，优先选可靠、快速节点 |
| 提交延迟   | 依赖保守超时，可能较高          | 延迟更低，几乎不需要超时 |
| 安全性    | 保证                   | 保证           |

---

### 🔗 核心思想一句话

DAG 提供高吞吐的交易传播，但必须用共识机制（如 anchor 提交逻辑）把它们线性化，而 Shoal 改进了 Bullshark 的锚点选择和提交机制，让它更快、更稳健。



# 📋 Shoal & Bullshark 延迟分析

### 🎯 定义

> **end-to-end (e2e) 共识延迟**：从某个副本第一次收到一笔交易，到它被排序（达成共识）的时间。

## e2e 延迟被拆成三个组成部分：

## ⏳ 1️⃣ **Queuing Latency（排队延迟）**

* 含义：

  * 交易进入本地 mempool 后，要等到它被包含进某一轮的 proposal 里。
  * 因为 DAG 按轮同步推进，每轮每个节点只能发出一个 proposal（包含一个交易批）。
  * 每轮需要一个可靠广播来认证 proposal，花费时间 $3md$。
* 最坏情况：

  * 交易刚错过当前轮，需要等下一个轮次才能被提议。
  * 延迟最多 $3md$。
* 平均情况：

  * 假设交易均匀到达，则平均排队延迟 ≈ $1.5md$。

---

## ⬆️ 2️⃣ **Anchoring Latency（锚定延迟）**

* 含义：

  * 提议的交易需要被某个最终提交的锚点引用（即被锚定）才能进入全局顺序。
  * 取决于：

    1. 锚点候选的出现频率。
    2. 锚点候选被提交的可靠性（有没有被跳过）。
* **Bullshark：**

  * 锚点候选每两轮出现一次（在奇数轮）。
  * 最好情况：

    * 如果交易在偶数轮提出，需要等 1 轮（$3md$）被锚定。
    * 如果在奇数轮提出，需要等 2 轮（$6md$）被锚定。
  * 平均锚定延迟 ≈ $4.5md$。
* **Shoal：**

  * 锚点候选每轮出现一次（更频繁）。
  * 所有交易都能在 1 轮（$3md$）内被锚定。
* 说明：

  * Shoal 还引入了锚点声誉机制，确保锚点更容易被提交，从而提高锚定的可靠性。

---

## 🔗 3️⃣ **Anchor Commit Latency（锚点提交延迟）**

* 含义：

  * 锚点被足够多的后续节点引用后正式提交。
  * 提交的规则：

    * **直接提交（Direct Commit Rule）**：

      * 锚点被下一轮中至少 $f+1$ 个节点直接引用。
      * 至少需要 2 个轮次：

        1. 认证锚点的 proposal（$3md$）。
        2. 收集到 $f+1$ 个引用它的 proposal（$3md$）。
      * 共计 ≈ $6md$。
    * **间接提交（Indirect Commit Rule）**：

      * 锚点未直接提交，但出现在未来某个已提交锚点的因果历史里。
      * 延迟 = 未来锚点的锚定延迟 + 未来锚点的提交延迟。

---

## 🧮 总结：最优情况

把三个阶段相加，得到在理想网络条件下的平均延迟：

| 延迟阶段                  | Bullshark (best case) | Shoal (best case) |
| --------------------- | --------------------- | ----------------- |
| Queuing Latency       | $1.5md$               | $1.5md$           |
| Anchoring Latency     | $4.5md$               | $3md$             |
| Anchor Commit Latency | $6md$                 | $6md$             |
| **总计**                | $12md$                | $10.5md$          |

但这里作者保守地按 Bullshark 的分析写成了总计 ≈ **12md**，表示 Shoal 至少比 Bullshark 更快一些。

---

## 🔍 为什么 Shoal 更优？

✅ 更频繁的锚点（每轮都有候选）。
✅ 锚点声誉机制提高锚点被引用和提交的概率。
✅ 总体减少了 Anchoring latency，从平均 $4.5md$ → $3md$。

---

## 📑 总结公式化：

$$
\text{e2e latency} ≈ \text{Queuing} + \text{Anchoring} + \text{Anchor Commit}
$$

在最佳情况下：

$$
\text{e2e latency} ≈ 1.5md + 4.5md + 6md = 12md
$$

Shoal 的优化主要体现在 **Anchoring Latency** 这个阶段，比 Bullshark 更低。

---

### **Shoal++ 的三大优化技术**  

#### **1. 更快的锚点提交（Faster Anchors）**  
Shoal++ 发现，当锚点（anchor）被**超多数（supermajority, 2f+1）未认证节点**引用时，可以优化 Bullshark 的**直接提交规则（Direct Commit Rule）**。具体来说：  
- **快速提交条件**：一旦观察到 **2f+1 个提案**（即使尚未完成认证）引用了某个锚点，副本就可以**提前提交**该锚点，因为这意味着最终一定能满足传统的 **f+1 认证节点**提交条件。  
- **备份机制**：由于该条件比传统规则更严格，可能无法总是满足，因此 Shoal++ **保留原 Bullshark 提交规则作为后备**，并利用 **Shoal 的声誉机制**选择高可信锚点，确保快速提交规则的可靠性。  
- **效果**：在实践中，这一优化将**锚点提交延迟（Anchor Commit Latency）**从 6 个消息延迟（md）降低到 **4md**。  

---

#### **2. 更多锚点（More Anchors）**  
Shoal++ 进一步扩展了 Shoal 的**高频率锚点策略**，尝试让**更多节点成为锚点**，以减少非锚点节点的等待时间（Anchoring Latency）。但这一优化需要权衡：  
- **优点**：增加锚点数量可减少事务等待时间，因为更多节点能直接触发提交。  
- **挑战**：过多的锚点可能导致**未提交的锚点堆积**，拖慢整体进度。  
- **解决方案**：  
  - **(i) 同步轮次推进**：Shoal++ 引入**小规模超时**，让副本在进入下一轮前稍作等待，确保 DAG 保持同步，减少遗漏锚点的情况。  
  - **(ii) 动态锚点调度**：根据 DAG 的实际进展**动态调整锚点候选**，跳过不再需要的锚点，避免阻塞后续提交。  

---

#### **3. 并行多 DAG（More DAGs）**  
为了最小化**排队延迟（Queuing Latency）**，Shoal++ 采用**多个并行 DAG 实例**，并让它们**错开时间运行**：  
- **机制**：如果一个事务错过当前 DAG 的轮次，可以立即加入**下一个 DAG 的新轮次**，而不必等待 3md。  
- **优势**：  
  - **降低排队延迟**：从平均 **1.5md** 减少到 **0.5md**（假设 3 个 DAG 交错运行）。  
  - **提高吞吐量**：更频繁的小批次提案（而非大批次低频提交）能更高效地利用网络和计算资源，接近**流式处理（streaming）**的效果。  
  - **资源优化**：多个 DAG 并行运行，避免单 DAG 的带宽瓶颈，提高整体可扩展性。  

---

### **总结**  
Shoal++ 通过 **（1）快速锚点提交、（2）动态锚点调度、（3）并行多 DAG** 三大优化，将**端到端延迟**从 Shoal 的 **10.5md** 降至 **4.5md**，同时保持高吞吐量。这些改进使 DAG-BFT 协议在**低延迟**和**高吞吐**之间取得更好平衡，适用于现代区块链和分布式系统。





# 🧰 Shoal++ DAG API & Node 结构

## 📄 **DAG API**

这些函数定义了 DAG 和共识逻辑的核心操作：

| 📋 函数                                    | 输入/输出                                | 含义                                                   |
| ---------------------------------------- | ------------------------------------ | ---------------------------------------------------- |
| **PROCESS\_NODE(v: Node)**               | 输入：节点 $v$                            | 处理一个收到的提议节点，检查有效性并投票。                                |
| **PROCESS\_CERTIFIED\_NODE(v: Node)**    | 输入：节点 $v$                            | 处理一个收到的已认证节点，将其加入 DAG 并触发提交逻辑。                       |
| **CAUSAL\_HISTORY(v: Node) → Vec<Node>** | 输入：节点 $v$<br>输出：节点序列                 | 返回 $v$ 的因果历史（即所有它引用的前驱节点）。                           |
| **RUN\_BULLSHARK(anchor: Node) → Node**  | 输入：一个 anchor 节点<br>输出：被提交的 anchor 节点 | 执行 Bullshark 的锚点提交逻辑，从给定 anchor 开始找到第一个应该提交的 anchor。 |
| **GET\_ANCHORS(round: int) → Vec<Node>** | 输入：轮次号<br>输出：候选 anchor 列表            | 返回该轮次的锚点候选节点（包含声誉逻辑）。                                |

---

## 🪪 **Node 结构**

一个 DAG 节点（提议或已认证）的基本属性如下：

| 字段            | 含义                                            |
| ------------- | --------------------------------------------- |
| **n.round**   | 该节点所在的轮次。                                     |
| **n.source**  | 发起该提议的副本 ID（谁提的）。                             |
| **n.parents** | 从上一轮 $v.round - 1$ 中选择的 $n-f$ 个父节点（形成 DAG 边）。 |


 **Algorithm 2: Core Shoal++ Utilities** 的伪代码实现。

---

# 🚀 Algorithm 2: Core Shoal++ Utilities

---

## 📦 **局部变量初始化**

| 变量              | 含义                                    | 初始值 |
| --------------- | ------------------------------------- | --- |
| **dag**         | DAG 数据结构                              | 空   |
| **weak\_votes** | $round \times source$ 的投票计数矩阵，用于统计弱投票 | 空矩阵 |
| **round**       | 当前轮次号                                 | 0   |
| **anchors**     | 当前轮的锚点候选集合                            | 空   |

---

## 🔄 主逻辑：`NEXT_ORDERED_NODES()`

找到下一个可以提交的交易序列。

```pseudo
1: procedure NEXT_ORDERED_NODES()
2:   while true do
3:     if anchors.IS_EMPTY() then
4:       round++
5:       anchors ← dag.GET_ANCHORS(round)  // 获取当前轮的锚点候选
6:     anchor ← anchors.POP()             // 取出一个锚点候选
7:     anchor_to_order ← select          // 确定实际要提交的锚点
8:     FAST_COMMIT(anchor)               // 等待弱投票达到阈值
9:     dag.RUN_BULLSHARK(anchor)         // 调用 Bullshark 逻辑提交
10:    if anchor_to_order ≠ anchor then
11:       SKIP_TO(anchor_to_order)       // 如果实际提交的锚点不是当前的，跳转
12:    return dag.CAUSAL_HISTORY(anchor_to_order)  // 返回提交的因果历史
```

---

## 🧭 **辅助过程**

### `SKIP_TO(anchor)`

跳转到给定的锚点，更新当前轮次 & 候选集。

```pseudo
13: procedure SKIP_TO(anchor: Node)
14:   round ← anchor.round
15:   anchors ← dag.GET_ANCHORS(round)
16:   anchors.REMOVE(anchor)  // 移除已经选择的 anchor
```

---

### `FAST_COMMIT(n)`

判断给定的节点 $n$ 是否收到了至少 $2f+1$ 个弱投票，如果是就返回提交。

```pseudo
17: procedure FAST_COMMIT(n: Node)
18:   while true do
19:     if weak_votes[n.round][n.source] ≥ 2f+1 then
20:        return n
```

---

## 📩 **事件处理器**

### 当收到普通节点时

处理一个尚未认证的 proposal。

```pseudo
21: upon receiving node n do
22:   r ← n.round
23:   s ← n.source
24:   if n 是 s 在 r 轮的第一个节点 then
25:     for each parent ∈ n.parents do
26:       weak_votes[r][parent]++
27:   dag.PROCESS_NODE(n)
```

* 对 $n$ 的父节点分别计数弱投票。
* 然后调用 `PROCESS_NODE()` 检查并投票。

---

### 当收到认证节点时

处理已经通过认证的节点。

```pseudo
28: upon receiving certified node cn do
29:   dag.PROCESS_CERTIFIED_NODE(cn)
```

---

# 🔷 总结流程

1️⃣ **主循环**：
不断找出下一个可提交的 anchor 节点及其因果历史。

2️⃣ **锚点选择 & 检查**：
获取当前轮的候选锚点，并通过弱投票判断能否直接提交。

3️⃣ **Bullshark 提交逻辑**：
调用 Bullshark 逻辑完成 anchor 及其因果历史的递归提交。

4️⃣ **事件驱动的节点接收**：
每收到一个节点或认证节点都更新 DAG 状态和投票。

---

# 🌟 关键思想

✅ 用 `weak_votes` 统计每个节点是否被足够多后继节点引用。
✅ 每轮动态选择最合适的 anchor 进行提交。
✅ 如果当前 anchor 不适合提交（可能因网络异步或投票不够），可以 `SKIP_TO` 下一个。
✅ 基于 DAG 和锚点的因果关系保证一致性。
✅ 确保提交有序且高效。


你贴的这段是对 **Shoal++ 对 Bullshark Direct Commit 规则的优化** 的详细解释，核心是提出了 **Fast Direct Commit** 规则。
我帮你总结提炼出来，逻辑清晰地讲解：

---

# 🚀 Fast Direct Commit in Shoal++

---

## 📝 背景：Bullshark 的 Direct Commit

在 Bullshark 里，提交一个 anchor（锚点）至少需要两个轮次（= 6 个消息延迟）：

* **第 1 轮**：

  * 认证 anchor 本身。
  * 消耗 $3md$（proposal → vote → certificate）。
* **第 2 轮**：

  * 收集到至少 $f+1$ 个后续节点的认证，它们链接到该 anchor。
  * 这意味着需要等到这些节点的认证完成。

**总共最少需要 $6md$。**

> 为什么要等这么久？
> 因为只有当后续节点认证完成（而且链接到 anchor），才能确保共识安全。

---

## 🔍 Shoal++ 的观察

Shoal++ 发现：

> **如果已经观察到至少 $2f+1$ 个提案（未认证）链接到某个 anchor，那么这个 anchor 的命运已经“板上钉钉”。**

原因：

* 虽然这些提案还没有认证，但至少其中 $f+1$ 必须来自诚实节点，而诚实节点不会双重提议。
* 所以最终一定会有 $f+1$ 个经过认证的节点链接到这个 anchor。

---

# ⚡ Fast Direct Commit 规则

* 当一个副本在 DAG 中观察到至少 $2f+1$ 个提案节点引用一个 anchor 时，就可以直接提交该 anchor。
* 不必等这些提案经过完整的认证过程。
* **延迟降低为：**

  * 第 1 轮：anchor 认证 $3md$
  * 加上下一轮的提案广播 $1md$（不等认证）
  * **总共 $4md$**

---

## 📉 为什么还能保证安全？

因为：
✅ 至少 $f+1$ 个提案来自诚实节点，它们一定会继续认证这些提案并链接到 anchor。
✅ 所以最终一定会形成满足 Bullshark 原始安全性的 $f+1$ 个经过认证的链接。

---

## 🔄 两种规则共存

* Shoal++ 同时保留了 **原始的 Direct Commit 规则** 作为兜底：

  * 如果网络很不稳定，或者 $2f+1$ 个提案进展缓慢，而最快的 $f+1$ 个节点认证完成得更快，这种情况下原始规则更有优势。
  * 在实践里，这种情况很少见。
* 默认优先使用 **Fast Direct Commit**。

---

## 🌟 为什么 Shoal++ 能更高效？

* Shoal++ 的声誉机制会优先选择网络连接好、响应快的节点作为 anchor 候选。
* 因此，未来的提案很可能快速链接到 anchor，使得 $2f+1$ 个提案很快被观察到。

---

## 📑 总结对比

| 特性   | Bullshark Direct Commit   | Shoal++ Fast Direct Commit  |
| ---- | ------------------------- | --------------------------- |
| 提交条件 | ≥ $f+1$ 个经过认证的提案引用 anchor | ≥ $2f+1$ 个（未认证的）提案引用 anchor |
| 延迟   | ≥ $6md$                   | ≥ $4md$                     |
| 安全性  | ✅                         | ✅                           |
| 可靠性  | 高（保守）                     | 高，大部分场景更快                   |

---

### 🔗 核心公式

Bullshark:

$$
\text{Commit latency} ≥ 6md
$$

Shoal++:

$$
\text{Fast Commit latency} ≥ 4md
$$

并且二者都安全，只是 Shoal++ 更积极地利用未认证的提案来推测未来的认证结果。


### **Shoal++ 的技术基础与核心机制**  

Shoal++ 的构建基于两个核心组件：  
1. **Narwhal 的 DAG 核心**（数据传播层）  
2. **Bullshark 的共识核心**（锚点提交逻辑）  

为简洁起见，论文未重复已有工作的伪代码（如节点广播、交易提交、垃圾回收等），而是聚焦于 Shoal++ 的创新点。以下为关键设计抽象：  

---

#### **1. DAG 接口与节点结构（Algorithm 1）**  
Shoal++ 的 DAG 层提供以下核心操作：  
- **`PROCESS_NODE`**：验证节点提案的有效性并投票。  
- **`PROCESS_CERTIFIED_NODE`**：将已认证的节点加入本地 DAG，并触发提交规则。  
- **`RUN_BULLSHARK`**：基于 Bullshark 的逻辑，从给定锚点出发确定首个可提交的锚点（§3.1.1）。  

**提交后的处理**：  
- 一旦锚点被提交，其**因果历史（causal history）**中的所有未排序节点将按确定性规则排序。  
- 辅助函数 **`CAUSAL_HISTORY`** 用于追溯节点的依赖关系，**`GET_ANCHORS`** 则动态生成每轮的锚点候选列表（继承并扩展了 Shoal 的声誉机制）。  

---

#### **2. 对已有组件的改进**  
Shoal++ 在 Narwhal 和 Bullshark 的基础上进行了关键增强：  
- **动态锚点调度**：通过 `GET_ANCHORS` 动态调整锚点候选，而非固定轮次分配。  
- **快速提交优化**：在 `RUN_BULLSHARK` 中引入 **Fast Direct Commit Rule**，结合声誉机制加速锚点确认。  
- **因果历史处理**：提交锚点时，通过 `CAUSAL_HISTORY` 确保依赖节点的高效排序，避免冗余通信。  

---

#### **3. 模块化设计优势**  
- **可扩展性**：DAG 层与共识层解耦，便于单独优化（如替换 Bullshark 为其他共识逻辑）。  
- **兼容性**：保留 Narwhal 的认证广播（RB）和 Bullshark 的安全保证，确保正确性。  
- **灵活性**：通过 `GET_ANCHORS` 等接口支持动态策略（如故障时调整锚点候选）。  

---

### **总结**  
Shoal++ 并非从零构建，而是通过**组合现有最佳组件（Narwhal + Bullshark）**并针对性优化（动态锚点、快速提交、多 DAG），实现了低延迟与高吞吐的平衡。其模块化设计既复用前人工作的正确性证明，又通过创新接口（如声誉驱动的 `GET_ANCHORS`）提升性能，为 DAG-BFT 协议提供了可扩展的改进框架。



你贴的是 **§5.2 Increasing anchor frequency** 部分的完整说明，是 Shoal++ 为减少 *Anchoring Latency* 而做出的更激进设计的详细讨论。
这段很长，我帮你总结并梳理核心思想、设计目标、问题和解决方案。

---

# 🚀 Shoal++：提高锚点频率

## 🎯 背景和动机

* **目标**：减少 *Anchoring Latency*

  > 如果每个节点都可以被当作一个 anchor 且能被提交，那么理论上 *Anchoring Latency* 就可以趋近于 0。

* **问题**：在 Bullshark/Shoal 中，每轮只选择一个（或少量）节点作为 anchor，其他节点要等到后续 anchor 触达自己才算被“捡起”，产生延迟。

* **直观想法**：如果每轮的所有节点都当作 anchor 并行推进，就可以消除这种等待时间。

---

## 🔄 如何做：并行化锚点

* 把一轮中的所有节点都当作 anchor，并为每个节点启动一个独立的共识实例。
* 每个实例运行一个 one-shot Bullshark + Fast Direct Commit。
* 这些实例可以 *并行运行*，但最终必须按照预定义的顺序提交，以确保一致性。
* 类比于 PBFT 中的多个 slot 并行提议。

---

## ⚠️ 遇到的问题：慢节点阻塞

* 锚点需要按照顺序提交或跳过。
* 如果前面的某个 anchor 太慢、无法提交或丢失，后面的 anchor 虽然已经满足条件，却必须等前面解决才能提交。
* 这种情况并不少见，因为 DAG 只要求边数 ≥ $2f+1$，剩下的节点可能根本没有连上，前面的 anchor 很容易失联或被拖慢。
* 这种情况下，即使绿色、粉色节点已经 ready 了，也得等红色节点提交或被跳过。

---

## 🪄 Shoal++ 的优化

### 1️⃣ 利用声誉机制

* 优先选择历史上表现好的、连通性强的副本作为锚点候选，减少慢节点入选的概率。
* 问题：声誉只能部分预测未来性能，尤其是在动态、地理分布的网络里，“慢节点”很容易变化。

---

### 2️⃣ 引入 Round Timeout

* 声誉有时过于保守，只挑最快的 $2f+1$ 个节点。
* 实际上很多节点只是略慢一些、但很快就能到。
* 所以 Shoal++ 让每轮在观察到最快的 $2f+1$ 个节点后，多等一个小的 timeout，再把新到的节点也算进 DAG。
* 这样 DAG 更密集，允许更多节点作为锚点候选。
* timeout 非必需，仅是性能优化，且非常短（相比于认证开销可忽略）。

---

### 3️⃣ 动态跳过锚点候选

* 即便有 timeout，有的 anchor 还是无法提交（如节点宕机、网络丢失、拜占庭作恶）。
* 当某个 anchor A1 在当前轮无法直接提交，可以通过后续轮的 anchor A2 把 A1 间接提交（Indirect Commit）。
* 如果 A1 甚至不能通过 A2 的因果历史间接提交（即被“跳过”），则 Shoal++ 直接把它标记为 skip 并用 no-op 填充它的 slot。
* 而且，发现 A1 被 skip 后，后续虚拟的（未 materialize 的）anchor 也一并跳过，直接跳到下一个合理的 anchor。

---

## 💡 动态 materialization

* 不是一开始就把每轮的所有节点都确定为 anchor，而是先虚拟化，只 materialize 一个当前的 anchor。
* 当当前的 anchor 提交或被 skip 后，Shoal++ 再决定下一个 anchor 应该放在哪里。
* 所有副本都会在本地得出一致的 anchor 排序和提交/跳过结果。

---

## 📝 总结

| 机制                   | 目的                                 | 优点           | 问题           |
| -------------------- | ---------------------------------- | ------------ | ------------ |
| 🚀 并行锚点              | 提升 anchor 覆盖率，降低 Anchoring Latency | 所有节点都能及时被覆盖  | 慢节点阻塞整个顺序    |
| 🪪 声誉                | 减少慢节点当选概率                          | 提高锚点质量       | 难预测动态网络的慢节点  |
| ⏳ Round Timeout      | 等一小会儿收集更多节点                        | DAG 更密集，更多候选 | 稍微增加每轮时延     |
| ⛔ Skipping & Dynamic | 遇到坏 anchor 时跳过并重排                  | 不被坏节点拖慢      | 需要一致地判断 skip |

---

## 📊 核心思想

✅ 并行推进但有序提交；
✅ 声誉 + timeout 提高候选质量；
✅ 动态跳过无效 anchor 保证进度；
✅ 动态 materialize 锚点避免浪费资源和被坏节点拖慢。

---

你贴的这一段是 **§5.3 Operating multiple DAGs in parallel**，讲的是 Shoal++ 为了降低 *Queuing latency* 而引入多 DAG 并行运行的设计。
下面帮你系统地总结这部分的核心思路、问题与解决方案。

---

# 🚀 Shoal++：并行运行多个 DAG 实例

---

## 🎯 背景与目标

* 目标：降低 **Queuing latency**
  （即交易从到达节点到被打包进 proposal 的等待时间）

* 问题：
  在单个 DAG 里：

  * 每个轮次的 proposal 要等一个完整的 $3md$ 认证周期后才开始下一个轮次。
  * 如果某个交易刚刚错过当前轮的打包，需要等到下轮开始，平均延迟约为 $1.5md$。

---

## 💡 Shoal++ 的解决方案：多 DAG 并行

* 核心思想：

  > 运行 $k$ 个并发的、相互隔离的 DAG 实例，把它们的输出交错（interleave）成一条全序日志。

* 类比：
  就像 *pipeline*，多条流水线同时推进，每个流水线错开一点点，整体吞吐更高、延迟更低。

---

## 📋 具体实现

* 每个 DAG 独立运行自己的 proposal、认证和提交逻辑（作为黑盒）。
* 每当某个 DAG 提交一个 anchor，就输出一个 **log segment**（一段有序日志）。
* Shoal++ 以轮转（round-robin）方式从可用的 DAG 里取出一个 log segment 并拼接到总日志。
* 如果某个 DAG 提交得更快，超出的 log segment 需要等待其他 DAG 赶上，保证轮转顺序。
* 注意：

  * 各个 DAG 互不阻塞；
  * log segment 大小可以不一样，因为锚点的因果历史长短不同。

---

## 📈 性能提升

* 为什么延迟更低？

  * 在单个 DAG 中，每个轮次耗时 $3md$，但多 DAG 里每个 DAG 交错启动，这样每隔 $1md$ 就有一个 DAG 在提交 proposal。
  * 平均 Queuing latency 从 $1.5md$ 降低到 $1.5/k \cdot md$。

* 实际部署：

  * 作者实验里用 $k=3$ 个 DAG，发现是性能的甜点：

    * 延迟显著下降；
    * batch 规模合理；
    * DAG 太多会导致 batch 太小、开销上升、收益递减。

---

## ⚙️ 消息优化？

* 理论上可以把不同 DAG 的消息重叠，比如把 DAG $i$ 的 proposal 消息和 DAG $i+1$ 的认证消息合并（共用签名）。
* 但实际上没必要：

  * 签名验证不是瓶颈；
  * 各消息阶段延迟不对称，强行对齐反而增加了 Queuing latency。

---

## 📑 总结对比

| 特性                 | 单个 DAG     | 多个 DAG ($k=3$) |
| ------------------ | ---------- | -------------- |
| 平均 Queuing latency | $1.5md$    | $0.5md$        |
| 提案频率               | 每 $3md$ 一轮 | 每 $1md$ 一轮     |
| log 结构             | 一条         | 多条交错           |
| 开销                 | 低          | 略高，但可接受        |
| 吞吐                 | 高          | 更高             |
| batch 大小           | 大          | 适中             |

---

## 🔷 核心思想

✅ 多个并行 DAG 提供多个“入口”，交易可以更快被打包；
✅ DAG 互相独立运行，保证不会互相拖慢；
✅ 输出在全局上交错组合成一致的有序日志；
✅ 轮转顺序保证一致性。

---

## 🔗 总结一句话

> Shoal++ 通过并行运行 $k$ 个 DAG 并交错其输出，将 Queuing latency 从 $1.5md$ 降到 $1.5/k \cdot md$，同时保持一致性和高吞吐。

最后这段是 **§5.4 Discussion**，是对 Shoal++ 全文优化效果的总结与讨论。帮你提炼如下：

---

# 📝 Shoal++ 讨论

## 🚀 总结性能提升

* 在常见、无故障的情况下：

  * 平均 **e2e 共识延迟** ≈ **4.5md**

    * $4md$：通过 Fast Direct Commit 提交 anchor
    * $0.5md$：通过多 DAG 并行降低 queuing latency
* 当存在故障或网络异常时：

  * 需要更多 anchoring 和 commit 延迟去通过 **Indirect Commit** 或 **Skip Rule** 来解决异常锚点。

---

## 🌐 通信模式

* Shoal++ 采用的是一种 **线性 star-based** 通信模式：

  * 节点之间的认证采用 $O(n)$ 的复杂度。
  * 相比之下，PBFT 等经典 BFT 协议使用的是 **all-to-all** （全互连）通信，复杂度 $O(n^2)$。
* 如果愿意，也可以切换成 all-to-all：

  * 延迟降低约 $1md$，总延迟可达 **3.5md**
  * 代价是更高的消息复杂度。

---

## 👥 客户端视角

* Shoal++ 中：

  * 每个副本都可以作为 proposer。
  * 客户端只需联系本地副本即可。
  * 不需要远程联系某个 leader。
* 对比：

  * 单 leader 设计（如 PBFT）要求客户端联系可能很远的 leader，增加了客户端的 queuing latency。

---

## 📑 小结亮点

| 优化点             | Shoal++                                          |
| --------------- | ------------------------------------------------ |
| 提交锚点延迟          | $4md$                                            |
| Queuing latency | $0.5md$                                          |
| 总 e2e 延迟        | $4.5md$                                          |
| 容错机制            | Indirect Commit / Skip Rule                      |
| 通信模式            | 默认 star-based ($O(n)$)，可切换 all-to-all ($O(n^2)$) |
| 客户端延迟           | 本地 replica 即可，无需远程 leader                        |

---

## ✨ 关键观点

✅ 在最优场景下，比 Bullshark 等现有方案更快、更鲁棒；
✅ 线性通信代价低，且在需要时可以选择更快但更重的 all-to-all；
✅ 客户端友好，避免单 leader 带来的远程延迟。

---

📌 **一句话总结：**

> Shoal++ 在无故障时可实现 $4.5md$ 的超低延迟，通过线性通信、高并发 DAG 和无 leader 设计，实现高效且鲁棒的共识。




# 📝 Shoal++ 正确性证明总结


## 🎯 目标

证明 Shoal++ 相对于 Shoal 和 Bullshark 的三个增强：

* Fast Direct Commit
* Multiple Anchors per Round
* Multiple parallel DAGs
  都 **保留了安全性（Safety）和活性（Liveness）**。

---

## 🔗 依赖的三个性质

Shoal++ 的正确性依赖于以下三个关键性质：
1️⃣ **Property 1**

> 对于任意节点 $n$，调用 $\text{CAUSAL_HISTORY}(n)$ 返回的节点序列在所有副本上是一致的。
> （来自 Narwhal：因为 DAG 中所有节点都是经过认证的。）

2️⃣ **Property 2**

> 所有副本以相同顺序提交相同的 anchors。
> （来自 Bullshark 的安全性证明。）

3️⃣ **Property 3**

> 对于任意轮次 $r$，leader 声誉机制保证 $\text{GET_ANCHORS}(r)$ 在所有副本上返回相同的节点序列。
> （来自 Shoal 的安全性证明。）

---

## 🔷 Fast Direct Commit Rule

* **活性**：沿用 Bullshark 的 Direct Commit Rule，所以活性直接成立。
* **安全性**：

  * **引理 1**：

    > 对于任意 anchor $a$，两个过程 $\text{FAST_COMMIT}(a)$ 和 $\text{RUN_BULLSHARK}(a)$ 返回的值不会矛盾。
  * **证明要点**：

    * FAST\_COMMIT(a) 需要收到 $2f+1$ 个引用 $a$ 的提案。
    * 至少 $f+1$ 个是正确副本发出的，这些提案最终都会被认证。
    * 因此 Bullshark 也会认定 $a$ 为可提交，返回相同的结果。

---

## 🔷 Multiple Anchors per Round

* **活性**：沿用 Bullshark 的机制，活性成立。
* **安全性**：

  * **引理 2**：

    > 假设所有正确副本初始状态一致（round 和 anchors），那么所有正确副本对 $\text{NEXT_ORDERED_NODES}$（算法 2）的调用返回相同结果。
  * **证明要点**：

    * 根据 Property 3 和引理 1，每次 while 循环中所有副本选择的 anchor 相同。
    * 因为 $\text{NEXT_ORDERED_NODES}$ 是确定性的，且 causal history（Property 1）一致，所以每次调用返回相同节点序列。

---

## 🔷 Multiple Parallel DAGs

* 每个 DAG 的 Safety 和 Liveness 已在上面分别论证。
* 因为多个 DAG 的输出是按照确定性顺序交错的，所以合成系统的 Safety 和 Liveness 也成立。

---

## 📜 最终定理

**Theorem 1**

> Shoal++ 协议（算法 3）满足 Safety 和 Liveness。

**理由：**

* 每个增强特性单独证明了安全性和活性。
* 多个 DAG 组合后，输出的顺序是确定的，安全性和活性自然继承。

---

### 📌 总结：

| 增强特性               | 安全性      | 活性               |
| ------------------ | -------- | ---------------- |
| Fast Direct Commit | ✓（见引理 1） | ✓（继承 Bullshark）  |
| Multiple Anchors   | ✓（见引理 2） | ✓（继承 Bullshark）  |
| Multiple DAGs      | ✓（输出可组合） | ✓（每个 DAG 独立满足活性） |

