# 论文 Idea 与创新点

## 1. 选题标准

如果目标是系统类 CCF-A，而且最终必须能作用到 `vLLM` 主线，那么我认为候选工作至少要满足五条标准：

1. **轻量级**：不能强依赖重新训练模型或改写大量 kernel。
2. **普适性**：不能只对单一模型、单一硬件、单一 attention 形式有效。
3. **高吞吐**：必须能在高并发、多请求场景下带来系统层收益。
4. **低延迟**：至少在 TTFT、TPOT、P99 ITL 中一项有稳定改善。
5. **可落地**：必须能自然地嵌入当前 `vLLM V1` 架构，而不是另起一套引擎。

按照这个标准，最不建议做的主线是“纯近似 KV 裁剪/压缩”。那类工作不是不能发，而是：

- 很容易变成模型近似推理论文
- 需要做精度 tradeoff
- 和 `vLLM` 当前最缺的系统闭环不完全对齐

## 2. 候选方向

### 方向 A：TierKV

**题目方向**

`TierKV: An Exact, Scheduler-Coordinated Multi-Tier KV Cache System for vLLM`

**核心思想**

把当前分散的：

- GPU prefix cache
- CPU KV offload
- external KV connector
- disaggregated KV transfer

统一成一个精确语义的多级 KV hierarchy，并由 scheduler 协同管理：

- block 放在哪一层
- 何时预取
- 何时下沉
- 何时提升
- 哪些块值得保留在 GPU

**为什么强**

- 与当前社区趋势完全对齐
- 不需要改模型语义
- 同时能打 throughput 和 latency
- 容易在 `vLLM` 中形成系统级贡献，而不是单点 patch

**风险**

- 需要改 scheduler、KV manager、offload、connector 多个层次
- 实验矩阵较大

### 方向 B：HeteroKV

**题目方向**

`HeteroKV: Heterogeneity-Aware KV Virtual Memory for Hybrid LLM Inference`

**核心思想**

解决 hybrid model 在 KV 上的异构性问题：

- layer-specific dtype
- layer-specific layout
- layer-specific block size / page size
- layer-aware cache policy

**为什么强**

- 新意比较足
- 对 hybrid model 的痛点切得很准

**风险**

- 实现更重
- 更依赖 hybrid 模型 benchmark 才能充分体现优势
- 可能需要触碰更多 backend 假设

### 方向 C：RouteKV

**题目方向**

`RouteKV: Cache-Aware Request Routing for Prefix-Heavy LLM Serving`

**核心思想**

把 prefix hit 问题从“单实例内的缓存命中”扩展到“多实例/多 DP rank 的路由优化”：

- 会话黏性
- prefix-aware routing
- cache pressure aware balancing

**为什么强**

- 比较容易做出来
- 对线上多轮对话 workload 很实用

**风险**

- 更像 serving optimization，系统新意略弱
- 单独成文时容易显得窄

## 3. 首选方向：TierKV

如果让我只选一条最值得投入的主线，我建议选 **TierKV**。

### 3.1 为什么它最适合系统论文

当前 `vLLM` 最明显的矛盾是：

- 它已经有很多 KV 相关能力
- 但这些能力还没有统一成一个系统

这类问题非常适合系统论文，因为它有天然的“problem gap”：

- 现有框架支持了很多机制，但机制之间没有统一语义
- 结果就是收益不稳定、实现复杂、兼容性脆弱
- 新系统的价值正好是把这些分裂机制统一起来

### 3.2 建议的三大创新点

#### 创新点 1：统一的多级精确 KV 抽象

把所有 KV block 统一建模为：

- `location`: GPU / CPU / Remote
- `state`: resident / in-flight / pinned / evictable
- `reuse_score`: 未来复用收益估计
- `reload_cost`: 取回代价估计

关键点是：它仍然是 **exact block reuse**，而不是近似压缩。

#### 创新点 2：调度器协同的收益感知放置与预取

当前很多 offload / connector 更像“缓存命中就拿、空间不足就下沉”。

你可以进一步做成 scheduler-aware：

- prefill 热块优先驻留 GPU
- decode 敏感块提前预取
- 长尾低复用块下沉到 CPU 或 remote
- hybrid model 按 group 独立决策，但由统一 scheduler 协同

这会比简单 LRU / ARC 更像系统创新。

#### 创新点 3：一致性安全的零气泡异步传输协议

围绕 async load/store 设计统一协议，例如：

1. `probe`
2. `pin`
3. `reserve`
4. `transfer`
5. `commit`
6. `release`

这样可以系统性解决：

- TOCTOU race
- pending transfer 与 allocator 竞争
- multi-tier state 不一致

这部分非常适合写成论文里的关键机制。

## 4. 在 vLLM 里应该怎么改

### 4.1 需要动的核心模块

- `vllm/v1/core/sched/scheduler.py`
  - 插入 tier-aware scheduling
  - 为请求分配预取窗口和放置决策

- `vllm/v1/core/kv_cache_manager.py`
  - 扩展 block 元信息
  - 提供统一的 tier lookup / pin / release 接口

- `vllm/v1/core/kv_cache_coordinator.py`
  - 让不同 KV group 的命中与放置决策可独立又可合并

- `vllm/v1/simple_kv_offload/*`
  - 从“独立 CPU offload 路线”提升为“二级 cache 实现”

- `vllm/distributed/kv_transfer/kv_connector/*`
  - 将外部 connector 从“旁路能力”收束为“远端 tier”

- `vllm/v1/metrics/*`
  - 增加 tier-aware 指标

### 4.2 推荐的实现顺序

为了降低研究风险，我建议分三步推进：

1. **单机两级**
   - GPU + CPU
   - 不接远端 LMCache/FlexKV
   - 先证明统一抽象和 scheduler 协同有效

2. **远端三级**
   - GPU + CPU + LMCache/FlexKV
   - 做 remote tier 接入

3. **Hybrid model 扩展**
   - 把 multi-tier 策略扩展到 hybrid KV groups

这样既能快速出第一版结果，也能为论文后续增强留空间。

## 5. 实验设计建议

### 5.1 模型

- 稠密模型：Llama-3.1-8B、Qwen3.5
- Hybrid 模型：Gemma-3、GPT-OSS、Bamba/Minimax 一类

### 5.2 Workload

- 多轮对话
- 长上下文问答 / RAG
- prefix-heavy agent workload
- disaggregated prefill

### 5.3 Baseline

- 原生 `vLLM`
- `vLLM + SimpleCPUOffload`
- `vLLM + LMCache`
- `vLLM + FlexKV`

### 5.4 指标

- output throughput
- request throughput
- TTFT
- TPOT
- P99 ITL
- prefix cache hit ratio
- external transfer bytes
- prefetch hit ratio
- recomputation saved

### 5.5 Ablation

- 只用统一抽象
- 只用收益感知策略
- 只用异步预取协议
- hybrid-aware vs non-hybrid-aware

## 6. 论文写作上的差异化表述

和现有工作的差异，建议这样写：

- 相比 `vLLM` 本地 prefix caching：
  - 我们不是只做本地 block reuse，而是统一多级 KV 管理

- 相比 `LMCache / FlexKV`：
  - 我们不是单个外部缓存层，而是 scheduler-coordinated tiered system

- 相比 `KIVI / H2O / RocketKV`：
  - 我们不依赖近似压缩或 token eviction，保持 exact semantics

- 相比纯 routing 工作：
  - 我们把请求调度与 KV 放置/预取统一起来，而不是只做入口分流

## 7. 最终建议

如果你的目标真的是系统类 CCF-A，我建议主线就定成：

`TierKV: 面向 vLLM 的精确语义、多级、调度协同 KV Cache 系统`

它最符合当前社区趋势，也最像一篇完整系统论文应该回答的问题：

- 为什么现有机制不够
- 为什么必须统一
- 为什么统一后能同时提升吞吐和时延
- 为什么这种设计对主流模型和部署场景都有普适性
