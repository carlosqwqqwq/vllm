# 方案对比与推荐

## 1. 先给结论

如果目标是“**尽量高概率做成一篇系统类 CCF-A 论文，并且尽快在当前 vLLM 上做出第一版结果**”，我现在最推荐的顺序是：

1. **TierKV**
2. **AffinityKV**
3. **QoSKV**
4. **SegmentKV**
5. **HeteroKV**

这个排序综合考虑了：

- 论文问题是否够大
- 与当前 `vLLM` 是否顺向兼容
- 社区证据是否充分
- 第一版原型是否可做
- 是否容易形成完整实验矩阵

## 2. 横向对比

| 方案 | 主要问题 | 系统新意 | 与当前 vLLM 兼容性 | 实现成本 | 最容易打出的收益 | 主要风险 |
| --- | --- | --- | --- | --- | --- | --- |
| `TierKV` | 多级 KV 能力分裂，缺少统一系统 | 很强 | 高 | 高 | 吞吐、TTFT、P99 ITL | scope 偏大 |
| `AffinityKV` | 多实例/DP 下缓存复用被路由浪费 | 强 | 高到中高 | 中 | TTFT、hit rate、端到端吞吐 | 需要多实例环境 |
| `QoSKV` | APC 存在 HoL、污染、不可观测 | 中强 | 很高 | 中 | P99 TTFT、稳定 hit | 若只做 patch 会显得窄 |
| `SegmentKV` | RAG/Agent 的非严格前缀复用缺失 | 强 | 中等偏高 | 中高 | TTFT、prefill saved | workload 依赖较强 |
| `HeteroKV` | Hybrid 模型 KV 抽象过于刚性 | 很强 | 中等 | 很高 | hybrid 模型兼容性与内存效率 | 实现和验证成本最高 |

## 3. 如果你只有一条主线可选

### 3.1 选 TierKV 的条件

如果你希望：

- 问题足够大
- 贡献最像系统论文
- 能同时打吞吐与时延
- 与 vLLM 当前社区趋势最一致

那就选 `TierKV`。

它最大优势是“系统闭环”最完整，很容易形成：

- 问题缺口
- 核心机制
- 多组 baseline
- 多维收益

### 3.2 选 AffinityKV 的条件

如果你们具备多实例或 DP 集群环境，而且更关注：

- gateway / router / serving fabric
- 端到端部署
- 会话/agent workflow

那 `AffinityKV` 很可能比 `TierKV` 更快做出漂亮结果。

### 3.3 选 QoSKV 的条件

如果你更重视：

- 低延迟
- P99 问题
- prefix cache 的稳定收益
- reasoning / interactive workload

那 `QoSKV` 会是很务实的一条线。

### 3.4 选 SegmentKV 的条件

如果你们的 workload 明显偏：

- RAG
- Agent
- tool-use
- 模板化上下文

那 `SegmentKV` 会非常有针对性。

### 3.5 选 HeteroKV 的条件

如果你们：

- 想打 harder problem
- 有 hybrid 模型资源
- 能接受更长实现周期

那才考虑 `HeteroKV`。

## 4. 从研究策略上，我更建议怎么做

我更建议采用 **主线 + 子机制** 的方式，而不是把多个方向并行铺开。

### 4.1 最稳组合

把 `TierKV` 作为主线，然后吸收：

- `QoSKV` 的一部分机制
  - 例如 hot prefix retention、prefetch / admission 协同、prefix state metrics
- `AffinityKV` 的一部分外部扩展
  - 例如在多实例部署里加入 KV-aware routing 作为扩展实验

这样会让论文更完整。

### 4.2 另一种强组合

如果你们更偏线上部署和 cluster serving，可以把 `AffinityKV` 作为主线，再把：

- `QoSKV` 的状态指标
- `SegmentKV` 的 workflow / context 结构化提示

作为增强项。

## 5. 我当前最真实的建议

如果让我现在帮你押一条方向，我会建议：

### 第一选择：TierKV

因为它最像“vLLM 当前最缺的一块拼图”。

### 第二选择：AffinityKV

因为官方文档和 production-stack roadmap 已经把它写得很明显了，而且系统论文味道也很强。

### 第三选择：QoSKV

因为它更容易快速打出低延迟结果，是一条很务实的攻坚线。

## 6. 下一步最合理的动作

如果你认同上面的排序，我建议下一步不要继续发散，而是收敛成下面两步：

1. 在 `TierKV / AffinityKV / QoSKV` 中选一条主线
2. 我再把它展开成一份正式的论文提案

正式提案应包含：

- Problem -> Gap -> Method -> Result -> Impact 的一句话定位
- 2 到 4 条明确贡献
- 系统设计草图
- 实验矩阵与 baseline
- 写作时的 related work 站位

## 7. 最终判断

如果你的目标是“**先做出最有胜算的一版**”，就选 `TierKV`。  
如果你的目标是“**直接切线上多实例部署痛点**”，就选 `AffinityKV`。  
如果你的目标是“**集中火力解决 prefix cache 的线上时延问题**”，就选 `QoSKV`。
