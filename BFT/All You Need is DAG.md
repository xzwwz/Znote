## 📄 论文：《All You Need is DAG》

### 核心贡献

这篇论文提出了一种新的基于 DAG（有向无环图）的异步拜占庭原子广播协议，称为 **DAG-Rider**。它是首个同时满足以下三个最优性的异步协议：

* 最优的拜占庭容错能力（最多容忍 ⌊n/3⌋ 个拜占庭节点）
* 最优的摊销通信复杂度
* 最优的时间复杂度

此外，它还具有：
✅ 后量子安全（不依赖传统签名）
✅ 保证所有正确进程提出的值最终都会被有序提交
✅ 所有正确进程最终观察到相同的全序

---

### 🦈 协议设计

**DAG-Rider** 由两个层次组成：
1️⃣ **通信层**

* 每个进程不断地通过可靠广播发送消息，每个消息是一个 vertex（顶点）。
* 所有 vertex 形成一个 DAG：每个顶点引用之前的顶点（强引用和弱引用），从而形成拓扑关系。
* 可靠广播确保拜占庭节点不能欺骗、篡改历史。

2️⃣ **本地排序层**

* 每个进程只需要观察自己的本地 DAG（不需要额外通信）。
* 使用一个全局随机 coin（阈值签名实现）来选择 leader，然后基于 leader 和 DAG 结构决定交易顺序。

---

### 📊 主要性质

* **通信复杂度**：与状态最优的 Dumbo 协议相当，可摊销到线性。
* **时间复杂度**：常数期望时间内达成共识。
* **安全性**：由于不需要签名，安全性可以实现后量子安全。
* **公平性**：所有正确进程提出的值都最终被包含，而不是丢弃。

---

### 📋 与其他协议对比

| 协议        | 通信复杂度   | 时间复杂度    | 后量子安全 | 保证公平性 |
| --------- | ------- | -------- | ----- | ----- |
| VABA      | O(n²)   | O(log n) | ❌     | ❌     |
| Dumbo     | 摊销 O(n) | O(log n) | ❌     | ❌     |
| DAG-Rider | 摊销 O(n) | O(1)     | ✅     | ✅     |

---

### 🔧 技术细节

* 每个 DAG 顶点由一个进程在某一轮广播。
* 强引用保证协议安全性，弱引用保证有效性（不遗漏慢节点的提议）。
* 使用可靠广播和一个全局 coin 保证共识一致性与随机性。
* 波浪式推进 DAG，每个 wave 确定一个 leader 并提交 leader 的因果历史。

---

### 总结

这篇论文展示了通过精心设计的 DAG 构建和本地排序逻辑，可以在异步、拜占庭环境下高效地解决原子广播问题，满足分布式账本/区块链等系统的高性能和高安全性需求。

---




### 📡 什么是可靠广播？

可靠广播（Reliable Broadcast）是一个通信抽象，用来解决异步、拜占庭环境下消息传递的问题。它保证：

* 如果一个正确进程广播了一条消息，那么所有正确进程最终都会接收到相同的消息。
* 每条消息最多被接收一次。
* 拜占庭进程没法让不同的正确进程收到不同的版本。

---

### 🚀 r\_bcast

全称：**reliable broadcast**

```text
r_bcast_k(m, r)
```

* 由进程 $p_k$ 调用，广播一条消息 $m$（比如一个交易块），属于某个“轮次” $r$。
* 意义就是「我宣布：这是我在第 r 轮的提议，请大家都记下来」。

每个正确进程在某轮里最多调用一次 r\_bcast（每轮一个 vertex）。

---

### 📬 r\_deliver

全称：**reliable deliver**

```text
r_deliver_i(m, r, k)
```

* 在某个正确进程 $p_i$ 本地触发：表示它已经确认接收到了 $p_k$ 在第 $r$ 轮广播的 $m$。
* 意义就是「我收到了 $p_k$ 在第 r 轮广播的消息 $m$，我可以放心把它加入我的 DAG 里」。

---

### 📜 保证的性质

这两个操作配合起来实现了可靠广播的三大性质：
✅ **一致性 (Agreement)**
如果一个正确进程 $i$ 收到并输出了 $r_deliver(m, r, k)$，那么最终所有正确进程都会输出相同的 $m$。

✅ **完整性 (Integrity)**
每个 $m$ 在每个正确进程里最多被 deliver 一次。

✅ **有效性 (Validity)**
如果 $p_k$ 是正确的，并且它调用了 $r_bcast_k(m, r)$，那么所有正确进程最终都会输出 $r_deliver(m, r, k)$。

---

### 🗂️ 在 DAG 中的角色

在 DAG-Rider 协议里，每个 DAG 顶点其实就是由某个 $r_bcast$ 发出去的消息创建的，其他进程通过 $r_deliver$ 接收消息后，把它放到本地 DAG 里，从而构建出一致的 DAG。

---

📌 总结：

| 操作             | 谁调用   | 作用         |
| -------------- | ----- | ---------- |
| **r\_bcast**   | 发起者进程 | 广播一条消息     |
| **r\_deliver** | 接收者进程 | 确认收到并接受该消息 |

---

DAG中的每个顶点表示来自某个进程的可靠广播消息，每个消息中包含对以前广播顶点的引用等数据