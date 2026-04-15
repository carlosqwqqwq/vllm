# 原始证据索引

这份文档专门保存本轮调研用到的**一手证据入口**，方便后续读者快速回溯，而不需要重新翻整轮对话。

## 1. 使用原则

这里优先保留四类证据：

1. **本地源码**
2. **本地官方文档**
3. **官方 GitHub release / PR / issue**
4. **production-stack 的官方路线与 RFC**

这里不主动收录二手博客、转述文章或未经验证的社区说法。

## 2. 本地源码证据

### 2.1 `V1` 已经是主线

- `vllm/engine/llm_engine.py`
  - 关键意义：该文件直接将 `LLMEngine` 指向 `vllm.v1.engine.llm_engine.LLMEngine`
  - 支撑结论：当前研究不应该再把 `V1` 当成实验分支，而应把它视作主线运行时

### 2.2 KV connector 已经很多，说明社区已进入多级/外部 KV 阶段

- `vllm/distributed/kv_transfer/kv_connector/factory.py`
  - 关键意义：统一注册了 `LMCacheConnectorV1`、`LMCacheMPConnector`、`NixlConnector`、`MooncakeConnector`、`FlexKVConnectorV1`、`SimpleCPUOffloadConnector` 等
  - 支撑结论：问题不再是“有没有 connector”，而是“这些 connector 有没有统一成系统”

### 2.3 当前 KV block 抽象仍存在异构性边界

- `vllm/v1/core/kv_cache_manager.py`
  - 关键意义：本地注释明确说，当前表示虽然现在可行，但如果未来不同 KV group 要不同 block size，这个假设会被打破
  - 支撑结论：`HeteroKV` 不是凭空想象，而是当前代码已经承认的未来结构缺口

### 2.4 scheduler 已经能接入 connector，说明多级系统化是自然方向

- `vllm/v1/core/sched/scheduler.py`
  - 关键意义：scheduler 侧已经与 `kv_transfer_config` / connector 发生接缝
  - 支撑结论：`TierKV` 这类 scheduler-coordinated 设计具备自然接入点

### 2.5 当前已经有 scheduler-side CPU offload 设施

- `vllm/v1/simple_kv_offload/*`
  - 关键意义：CPU offload 并不是纯 worker hack，而是已经有成体系的 scheduler-side 设施
  - 支撑结论：多级 tier 方向具备现实基础

## 3. 本地官方文档证据

### 3.1 Prefix caching 的当前语义与限制

- `docs/design/prefix_caching.md`
  - 关键意义：明确 APC 是 hash-based exact prefix reuse，且只缓存完整 block
  - 支撑结论：当前 APC 是精确语义，许多改进应建立在 exact reuse 之上

### 3.2 Hybrid KV 当前仍有结构限制

- `docs/design/hybrid_kv_cache_manager.md`
  - 关键意义：明确 hybrid KV 的 group 抽象、页共享假设与一些 WIP 场景
  - 支撑结论：hybrid KV 已经进入主线，但仍未完全成熟

### 3.3 Disaggregated prefill 已经进入官方文档

- `docs/features/disagg_prefill.md`
  - 关键意义：说明外部 KV / 分离式 prefill 已经是官方认可能力
  - 支撑结论：`TierKV` 和 `AffinityKV` 都不是偏离主线的题

### 3.4 官方明确承认 KV-aware routing 的价值

- `docs/serving/data_parallel_deployment.md`
  - 关键意义：
    - 每个 DP engine 有独立 KV cache
    - intelligent routing 可最大化 prefix caching 收益
    - 当前 internal DP load balancing 未来可纳入 KV-cache-aware logic
  - 支撑结论：`AffinityKV` 是官方已经承认的重要方向

### 3.5 production-stack 自己把 routing 与 KV reuse 绑定起来

- `docs/deployment/integrations/production-stack.md`
  - 关键意义：文档中明确写了 prefix-aware routing、KV cache offloading、model-aware routing 等关键词
  - 支撑结论：路由不再只是外围工程问题，而是 vLLM 生态中的核心性能杠杆

## 4. 官方 release 证据

### 4.1 `v0.18.0`（2026-03-20）

- 关键信号：
  - smart CPU offloading
  - FlexKV offloading backend
  - multiple KV groups
  - HMA + NIXL connector
- 支撑结论：社区正把 KV 从单层本地缓存推向多级、外部化与 hybrid-aware

### 4.2 `v0.19.0`（2026-04-03）

- 关键信号：
  - general CPU KV cache offloading
  - GPU-side KV events
  - KV connector metadata extensibility
  - hybrid 相关改进
- 支撑结论：KV 已经不是单点 feature，而是 release 级主线能力

## 5. 关键 PR 证据

### 5.1 `#16625` `LMCacheConnectorV1`

- 时间：2025-04-26 merged
- 关键信号：
  - LMCache 被正式接入 V1
  - PR 描述给出 2x H100 场景下吞吐与 tail ITL 收益
- 支撑结论：外部 KV / PD 路径是被社区实测认可的主赛道

### 5.2 `#31707` `multiple KV cache groups in Hybrid KV Coordinator`

- 时间：2026-01-09 merged
- 关键信号：Hybrid KV 已经不再局限于非常简单的 group 情况
- 支撑结论：hybrid model 兼容性已成为核心问题之一

### 5.3 `#34328` `FlexKVConnectorV1`

- 时间：2026-03-12 merged
- 关键信号：
  - FlexKV 被接入为 offloading backend
  - PR 描述给出 TTFT / TPOT / QPM 收益
- 支撑结论：多级 / 分布式 KV store 已经被官方主线吸收

### 5.4 `#37160` `Simple yet General CPU KV Cache Offloading`

- 时间：2026-04-01 merged
- 关键信号：
  - 复用 BlockPool / KVCacheCoordinator
  - 强调 simple、general、hybrid model support
- 支撑结论：社区正在主动把 KV offload 向轻量级、普适性方向收敛

## 6. 关键 issue / RFC 证据

### 6.1 指向 `TierKV / HeteroKV` 的证据

- `#23161`
  - 主题：layer-specific KV dtype & layout
  - 意义：当前统一表示对 hybrid 模型不够自然

- `#39702`
  - 主题：SimpleCPUOffloadScheduler TOCTOU race
  - 意义：多级 / 异步传输的一致性协议仍然是系统痛点

- `#36771`、`#38700`
  - 主题：LMCache 与 hybrid KV / 新模型兼容性问题
  - 意义：多级与 hybrid 的接缝仍未完全稳定

### 6.2 指向 `QoSKV` 的证据

- `#38194`
  - 主题：vLLM 的 prefix cache hit 低于其他 inference stack
  - 意义：APC 收益在真实 workload 中并不稳定

- `#37308`
  - 主题：Prefix caching 下 asymmetric batch 导致严重 HoL blocking
  - 意义：APC 可能带来显著低时延副作用

- `#39321`
  - 主题：reasoning model thinking tokens 污染 prefix cache
  - 意义：缓存污染与死缓存是现实问题

- `#33434`
  - 主题：prefix cache state metrics
  - 意义：prefix cache 缺少生产级观测能力

- `#23083`
  - 主题：persistent / pinned prefixes
  - 意义：热前缀保留是明确需求

### 6.3 指向 `AffinityKV` 的证据

- `production-stack #59`
  - 主题：prefix-cache-aware routing RFC
  - 意义：社区已经开始认真讨论 KV / prefix 感知路由

- `production-stack #590`
  - 主题：clarify prefix-aware routing vs KV-cache-aware routing
  - 意义：该问题已从“有没有”进入“怎么界定与落地”

- `production-stack #244`
  - 主题：agentic workflows via KV-cache reuse and context-aware routing
  - 意义：KV-aware routing 与 agent workflow 已被明确关联

- `production-stack #855`
  - 主题：2026 roadmap
  - 意义：公开路线图里明确写入 KV-cache-aware and prefix-aware routing、predictive routing、agent workload intelligent routing

### 6.4 指向 `SegmentKV` 的证据

- `production-stack #244`
  - 主题：agentic workflow / context-aware routing
  - 意义：现代 workload 已经明显偏向共享上下文、多阶段 workflow，而不是单轮独立请求

- 本地 `docs/serving/data_parallel_deployment.md`
  - 主题：应根据 KV cache state 做 intelligent routing
  - 意义：共享上下文的价值已被官方认可，只是还未延伸到 segment 级复用

## 7. production-stack 证据

### 7.1 Router README

- `src/vllm_router/README.md`
  - 关键信号：
    - 支持 session routing
    - `(WIP) prefix-aware routing`
  - 支撑结论：router 已经不是纯 round-robin，且 KV 复用已进入其设计空间

### 7.2 production-stack 仓库主页与 roadmap

- 关键信号：
  - request routing
  - KV cache offloading
  - future router improvements
  - predictive routing / intelligent routing
- 支撑结论：部署侧优化与 KV 复用的结合已成为官方生态路线

## 8. 仓库统计只作为辅助信号

本轮调研还看了仓库活跃度与 star 数，但这些只作为辅助，不作为结论依据。真正决定选题价值的仍然是：

- 代码结构
- 官方文档
- release
- PR / issue / roadmap

## 9. 如何使用这份索引

### 9.1 如果你要继续调研

优先顺序建议是：

1. 先看本地代码与文档证据
2. 再看 release 与关键 PR
3. 最后按 idea 查对应 issue / roadmap

### 9.2 如果你要写论文

可按下面方式引用思路：

- 用本地文档与源码说明系统现状
- 用 release / PR 说明社区主线演进
- 用 issue / RFC 说明仍未解决的问题

### 9.3 如果你要做实验

优先从与目标 idea 最相关的证据出发设计 workload：

- `TierKV`：offload / connector / hybrid / race
- `AffinityKV`：DP / router / prefix-aware routing
- `QoSKV`：HoL / pollution / metrics / pinned prefix
- `SegmentKV`：agent workflow / shared context
- `HeteroKV`：layer-specific dtype/layout/block size

## 10. 结论

这份索引的意义不在于列得多，而在于让后续读者能快速回答三件事：

1. 这轮调研看了哪些一手材料。
2. 每条材料在论证什么。
3. 它最终支持了哪条研究主线。
