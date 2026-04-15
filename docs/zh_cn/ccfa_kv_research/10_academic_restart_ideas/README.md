# 第六轮：按学术问题重开的三个方向

## 1. 本轮目标

这一轮不再从 `vLLM` 的局部工程痛点出发，而是从更高层的系统问题出发，重新定义候选方向。

更具体地说，本轮先明确排除上一轮偏工程切口的三个候选：

- `AliasStableKV`
- `ComposablePrefixKV`
- `NeighborhoodKV`

排除它们的原因不是这些点没有价值，而是它们更像：

- 实现层 hardening
- 数据结构局部优化
- cache policy 或 memory policy 的细化

它们还不够像一篇 `OSDI/CCF-A` 论文应有的“问题类”。

因此，这一轮的筛选标准被重置为：

1. 研究对象必须是 `LLM serving` 的共性问题，而不是 `vLLM` 私有实现细节。
2. 贡献里必须包含新的系统抽象，而不是单点 heuristic。
3. 论文命题应当能脱离 `vLLM` 叙事成立，`vLLM` 只是验证载体。
4. 方向必须同时兼顾：轻量级、普适性、高吞吐、低延迟。
5. 必须诚实标注重合风险，而不是宣称“确保没人做过”。

## 2. 本轮使用的学术母题

本轮把相关工作重新聚成三类命题空间：

### 2.1 状态表示

代表工作：

- `HCache`
- `Apt-Serve`
- `ShadowKV`

它们研究的是：是否可以用 `KV` 之外的状态表示来获得更好的恢复效率、吞吐或内存占用。

### 2.2 状态生命周期

代表工作：

- `INFERCEPT`
- `Tokencake`
- `KVFlow`
- `TensorRT-LLM Early Reuse`

它们研究的是：可复用状态何时产生、何时可见、何时暂停、何时迁移或恢复。

### 2.3 状态组合语义

代表工作：

- `Prompt Cache`
- `Serve Programs, Not Prompts`
- `EPIC / MEPIC`
- `StepCache`
- `Don't Break the Cache`

它们研究的是：可复用状态如何切分、组合、复用或验证。

## 3. 本轮保留下来的三个方向

| 方向 | 核心学术命题 | 当前角色 | OSDI 潜力 | vLLM 可验证性 | 最大风险 |
| --- | --- | --- | --- | --- | --- |
| `PolyStateKV` | 可复用状态是否应被抽象为多表示对象，而非固定为单一 KV | 主线候选 | 高 | 中高 | 范围容易过大，必须限制 prototype |
| `TxnStateKV` | 可复用状态是否应具备统一事务语义，而不是靠 ad hoc lifecycle 规则 | 主线候选 | 中高 | 高 | 如果 workload 不够 agentic，可能显得边界过窄 |
| `CapsuleKV` | exact reusable state 的最小可组合对象是什么 | 强备选 / 主线候选 | 中高 | 中高 | 容易被误解成 hashing 或 ABI patch |

当前推荐顺序：

1. `PolyStateKV`
2. `TxnStateKV`
3. `CapsuleKV`

如果只看“新颖度天花板”，则更接近：

1. `PolyStateKV`
2. `CapsuleKV`
3. `TxnStateKV`

## 4. 三个方向之间的关系

这三条线不是同一个点的三个别名，而是三种不同层级的系统命题：

- `PolyStateKV`
  - 讨论 **可复用状态应以什么形式存在**
- `TxnStateKV`
  - 讨论 **可复用状态应以什么生命周期语义运行**
- `CapsuleKV`
  - 讨论 **可复用状态应以什么组合对象被表达和验证**

因此，它们更像三种互相竞争的论文视角，而不是一个线性依赖栈。

## 5. 推荐阅读顺序

1. [PolyStateKV](polystatekv/README.md)
   - 先读。它最像一个新的系统抽象，研究“state form polymorphism”。
2. [TxnStateKV](txnstatekv/README.md)
   - 第二读。它把 reusable state 从静态 cache entry 提升为事务对象。
3. [CapsuleKV](capsulekv/README.md)
   - 第三读。它研究 exact reusable state 的最小可组合单位。

## 6. 当前最稳妥的总判断

截至 `2026-04-15` 这一轮检索，我没有发现与这三个命题完全等价的公开系统论文；但也**不能写成“确保没人做过”**。

更稳妥的表述是：

> Existing work has separately optimized state representation, state restoration, state publication, modular prompt reuse, or backend-agnostic output reuse. A missing systems question is whether reusable state in modern LLM serving should be treated as a first-class object with polymorphic forms, transactional lifecycle, or composable exact-state semantics.

这比直接写 `first` 更安全，也更符合当前证据强度。

## 7. 本轮关键资料

- HCache: <https://arxiv.org/abs/2410.05004>
- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- ShadowKV: <https://arxiv.org/abs/2410.21465>
- INFERCEPT: <https://arxiv.org/pdf/2402.01869>
- Tokencake: <https://arxiv.org/abs/2510.18586>
- KVFlow: <https://arxiv.org/abs/2507.07400>
- TensorRT-LLM Early Reuse: <https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/>
- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Serve Programs, Not Prompts: <https://arxiv.org/abs/2510.25412>
- EPIC: <https://arxiv.org/abs/2410.15332>
- MEPIC: <https://arxiv.org/abs/2512.16822>
- StepCache: <https://arxiv.org/abs/2603.28795>
- Don't Break the Cache: <https://arxiv.org/abs/2601.06007>

