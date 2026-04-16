# RealizeKV 在 vLLM 中的只读落点图

更新时间：2026-04-15

说明：

- 本文档只做代码结构分析，不改代码。
- 目标是回答一个问题：
  - 如果以后要把 `RealizeKV / FalseNegativeKV` 真正做进 `vLLM`，当前仓库哪些位置最适合承载这件事？

## 1. 当前代码里已经存在的关键事实

## 1.1 request 级别已经有“跳过读 prefix cache”的总开关

相关位置：

- `vllm/v1/request.py`
- `vllm/sampling_params.py`
- `vllm/pooling_params.py`

当前可确认的事实：

- `Request.get_skip_reading_prefix_cache()` 会读取 sampling / pooling 参数。
- `sampling_params.py` 中，`prompt_logprobs` 默认会把 `skip_reading_prefix_cache` 置为真。
- `pooling_params.py` 里也存在默认跳过 APC 的路径。

这意味着：

- 现在系统不是“查不到 hit”。
- 而是很多请求在进入查表前，就已经被策略性地挡在 prefix cache 之外。

这正是 `counterfactual hit` 的自然测量入口。

## 1.2 KVCacheManager 是 physical hit 的总入口

相关位置：

- `vllm/v1/core/kv_cache_manager.py`

关键事实：

- `get_computed_blocks()` 在最前面就检查：
  - `enable_caching`
  - `request.skip_reading_prefix_cache`
- 如果任一条件不满足，直接返回空 hit。

这意味着：

- `KVCacheManager` 天然适合做：
  - physical hit 统计
  - counterfactual probe
  - realization gap 度量

如果以后实现 `RealizeKV`，这里是最自然的 profiler 落点。

## 1.3 exact-prefix 的真正命中逻辑在 coordinator / block pool

相关位置：

- `vllm/v1/core/kv_cache_coordinator.py`
- `vllm/v1/core/single_type_kv_cache_manager.py`
- `vllm/v1/core/block_pool.py`

当前代码说明了三件事：

### A. `block_pool.get_cached_block()` 是 exact-hash 读口

- 它按 `block_hash + group_id` 查找 block。
- 缺任一 group 就判 miss。

### B. `FullAttentionManager.find_longest_cache_hit()` 是线性前缀扫描

- 对 full attention，它按 block hash 从左到右扫描。
- 一旦 miss 就停止。

### C. `HybridKVCacheCoordinator.find_longest_cache_hit()` 用 fixed-point 求跨 attention type 的共同命中长度

- 这意味着 hybrid / mixed attention 模型里，hit 本身就可能受到跨组约束。
- 因而 `counterfactual hit` 不应该只看单一路径，而应看统一 coordinator 层。

这给我们两个实现启发：

1. `counterfactual probe` 最终应落在 coordinator 视角，而不是散落在单一 feature patch 里。
2. `bridge planner` 不能破坏现有 exact-hash + fixed-point 的结构，而应在它之上增加 typed contract 层。

## 1.4 attention backend 已经会在某些条件下整体关闭 APC

相关位置：

- `vllm/model_executor/layers/attention/attention.py`
- `vllm/model_executor/layers/attention/mla_attention.py`

当前代码事实：

- 当 `VLLM_BATCH_INVARIANT` 开启，且后端为 `FLASHINFER` 或 `TRITON_MLA` 时，prefix caching 会被直接禁用。
- `cache_config.prefix_caching_disable_reason` 会被置为 `backend_incompatibility`。

这很重要，因为它说明：

- backend mismatch 并不是抽象推测，而是当前代码里的明确现实。
- 如果后续做 `RealizeKV`，backend mismatch 完全可以成为第一类正式论文对象，而不是扩展项。

## 1.5 当前仓库里已经有这条线的脚手架

相关位置：

- `vllm/v1/core/false_negative_route.py`
- `vllm/v1/core/prompt_logprobs_sidecar.py`
- `vllm/v1/core/twin_path_policy.py`
- `vllm/v1/engine/input_processor.py`

从只读分析可见：

### A. 已经有 prefix family / route key 的概念

- `false_negative_route.py` 会构造：
  - `prefix_family`
  - `api_contract_class`
  - `backend_compatibility_class`
  - `false_negative_risk_class`

这几乎就是 `Typed Reuse Contract` 的雏形。

### B. 已经有 prompt_logprobs selective replay / sidecar 配置与规划

- `prompt_logprobs_sidecar.py` 里已经有：
  - sidecar key
  - selective replay config
  - boundary replay plan

这意味着 observation mismatch 的 bridge 不是空想，而是已有早期机制原型。

### C. 已经有 twin-path policy

- `twin_path_policy.py` 已经把 route 决策拆成：
  - `reuse_optimized_pool`
  - `throughput_optimized_pool`
  - `family binding`

这正好对应 `RealizeKV` 的 route / scheduler 一层。

结论：

- 这条研究线在本仓库中已经不再是“idea-only”状态。
- 更准确地说，它已经具备：
  - typed route key
  - sidecar/replay bridge 雏形
  - twin-path policy 雏形

真正缺的，是把这些离散机制统一成 paper-grade substrate。

## 2. 最推荐的实现分期

## P0：measurement-first

目标：

- 不急着恢复所有 miss。
- 先证明问题规模。

最小动作：

- 记录 physical hit
- 记录 counterfactual hit
- 记录 realization gap
- 按 route key 分桶

为什么先做这个：

- 没有 measurement，reviewer 很容易说这是 isolated engineering issue。

## P1：Observation Mismatch Bridge

最推荐先打：

- `prompt_logprobs`
- 部分 `pooling / token-level observation`

最小机制：

- boundary replay
- bounded sidecar

为什么先做这个：

- 这类请求的 gap 往往直接、可解释、易复现。
- 同时又不会马上撞到 PIC 或 cross-model reuse。

## P2：Backend Mismatch Route

最推荐对象：

- `FLASHINFER / TRITON_MLA + batch invariance`

最小机制：

- typed route
- family binding
- compatible pool selection

为什么做它：

- 这类问题在代码里已经是显式禁用 APC。
- 非常适合写成“runtime contract mismatch”。

## P3：统一 Realization-Aware Scheduler

条件：

- 只有 P1/P2 的 gap 和收益都足够大时才值得做。

最小目标：

- 引入 bridge cost
- 引入 realized gain
- 在 `schedule()` 中做 realization-aware 决策

如果没有充足数据，不建议过早做这一层。

## 3. 实验设计建议

## 3.1 微基准

目标：

- 证明 gap 存在且不小。

建议指标：

- `physical_hit_tokens`
- `counterfactual_hit_tokens`
- `realization_gap`
- `gap ratio = gap / counterfactual_hit_tokens`

分桶维度：

- API contract
- backend
- carrier type
- prompt length
- repeated prefix family

## 3.2 系统基准

建议工作负载：

### A. 长 system prompt + 多轮 agentic session

- 这是现实场景，也和最新评测论文一致。

### B. 长文档 QA / RAG

- 可直接复用 `benchmark_prefix_caching.py` 和现有 long-doc 基准思路。

### C. prompt_logprobs / pooling / classification 请求

- 这是最能直接打出 realization gap 的负载。

### D. hybrid / backend mismatch 场景

- 用于证明 route 的必要性。

## 3.3 论文主指标

除了 TTFT / throughput 以外，建议新增三类指标：

### A. Latent-Reuse 指标

- `counterfactual_hit_ratio`
- `realized_hit_ratio`

### B. Realization 指标

- `realization_efficiency = realized_hit / counterfactual_hit`
- `bridge_cost_per_realized_token`

### C. 系统指标

- TTFT
- p95 / p99 latency
- throughput
- HBM footprint
- extra replay tokens

这组指标能把论文从“工程 patch”拉成“系统问题 + substrate”。

## 4. 当前最不建议做的事

### 不建议把 scope 一口气做宽

特别不建议同时上：

- PIC
- multimodal arbitrary-position reuse
- cross-model / cross-adapter reuse
- cluster-level pool

这样会立刻失控，也容易撞近邻论文。

### 不建议把 sidecar 扩成第二套 hidden cache 系统

原因很直接：

- 一旦这么做，就会明显靠近 hidden cache / hybrid cache 路线。
- 第一版必须把 sidecar 收窄为最小 bridge state。

### 不建议直接把故事写成 `TwinPathServe`

`TwinPathServe` 更像实现名，不像 thesis。

更好的关系是：

- thesis：`RealizeKV`
- 内部问题名：`FalseNegativeKV`
- 当前最小实现：`TwinPathServe`

## 5. 最后判断

就当前仓库状态而言，最强的事实不是“我们已经把系统做完了”，而是：

1. `vLLM` 当前 prefix caching 已经是核心基础设施。
2. 当前代码里已经存在明确的 skip / disable / route / sidecar 入口。
3. 因此 `counterfactual exact hit` 在这个仓库里不是抽象概念，而是一个可以被测、被桥接、被调度的问题。

这正是后续最值得继续推进的地方。
