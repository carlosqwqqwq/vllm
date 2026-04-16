# ExactPrefixKV Measurement Plan

更新日期：2026-04-15

## 1. 目标

这份文档的目标不是直接设计完整系统，而是回答：

> `ExactPrefixKV` 是否值得进入实现阶段？

因此它服务于一个 **go / no-go 决策**。

如果 measurement 结果不支持 thesis，就应及时止损；
如果结果支持，再进入最小系统原型。

## 2. 需要验证的核心假设

### H1：真实 workload 中存在大量“可避免的 prefix drift”

也就是：

- 请求之间本来存在高共享前缀；
- 但由于 tool schema 排布、message rendering、reasoning/history 组织、multimodal placeholder 布局等差异，
- 这些请求没有形成稳定的 exact prefix。

如果 H1 不成立，`Cacheability ABI` 不足以支撑一层贡献。

### H2：异构 carrier 的 exactness 仍然缺少统一观测与归因

也就是：

- vLLM 虽然已经支持部分 `mm_hashes`、`cache_salt`、`prompt_embeds`；
- 但对用户和开发者来说，仍缺少统一的：
  - exactness 语义
  - cache breaker 归因
  - typed hit / miss 观测

如果 H2 不成立，`TypedPrefixKV` 会退化成已有实现的文档化整理。

### H3：APC 命中之后仍有明显 residual inefficiency

也就是：

- 即便 prefix hit 已经很高；
- residual prefill 的 shape 仍频繁进入非最佳路径；
- 导致：
  - graph replay 不稳定
  - padding waste
  - cascade 未被最好利用
  - TTFT / p99 未兑现理论收益

如果 H3 不成立，`ResidualModeKV` 只能降级成次要优化。

### H4：三类问题在同一 workload 上可以形成闭环

也就是：

- 不是三类问题各自只在不同 workload 上零散出现；
- 而是在同一类生产相关 workload 中，确实存在：
  - prefix drift
  - typed exactness 复杂性
  - post-hit inefficiency

只有这样，组合 thesis 才成立。

## 3. 决策标准

measurement 结束后，用下面规则做 go / no-go。

### 3.1 Go 条件

满足下面三条中的两条，就可以进入 P1 最小系统实现。

1. 在至少两个 workload family 中，检测到 **>= 15% 的可避免 miss / prefix drift**。
2. 在至少两个 heterogeneous carrier workload 中，typed exactness 或 breaker attribution 缺失导致明显不可解释 miss，且现有 metrics 无法定位。
3. 在 APC 高命中场景下，post-hit inefficiency 至少带来：
   - `>= 8%` 的 p95/p99 TTFT 改善空间，或
   - `>= 10%` 的 graph replay / padding / cascade utilization 改善空间。

### 3.2 No-Go 条件

满足下面任意两条，就不建议继续做完整系统。

1. drift-induced miss 很少，且集中在不重要 workload。
2. typed carrier 问题大多已被现有 vLLM 路径覆盖。
3. residual inefficiency 的改善空间在真实 workload 上很小。
4. 三类现象无法在同一论文故事中串起来。

## 4. 测量范围

为避免和分布式 routing / workflow OS 方向纠缠，measurement scope 固定为：

- **单机或 colocated vLLM**
- **exact-prefix serving**
- **不修改模型**
- **不先改 kernel**

不纳入本轮 measurement 的内容：

- 分布式 cache affinity
- program / workflow-aware runtime
- approximate reuse / semantic cache
- 新 attention kernel

## 5. Workload 设计

至少覆盖四类 workload，每类都要有：

- baseline trace
- prefix-sharing trace
- 有控制变量的 drift 版本

### 5.1 W1：长 system prompt + 多轮 chat

目标：

- 观察 chat rendering、历史拼接和 reasoning/history 过滤对 exact prefix 的影响。

建议特征：

- 长 system prompt
- 多轮 user / assistant 历史
- 变动信息只在 suffix

优先原因：

- 最接近 provider prompt caching 的基础场景；
- 是 `Cacheability ABI` 的最低门槛测试。

### 5.2 W2：tool-rich / agentic workload

目标：

- 观察 tool schema、tool_choice、tool result 注入、responses API 轮转对前缀稳定性的影响。

建议特征：

- 固定 tools 定义
- 变动的 tool results
- 多轮 responses / function calling

优先原因：

- 最能检验 `PromptABIKV` 是否只是经验帖；
- 也是最接近 `Don't Break the Cache` 的对照场景。

### 5.3 W3：prompt embeds workload

目标：

- 观察 prompt 以 `prompt_embeds` 进入时，typed exactness 与 cache observability 是否仍有缺口。

建议特征：

- 多请求共享相同 embedding prefix
- suffix 长度可控变化

优先原因：

- 最能检验 `TypedPrefixKV` 是否还有论文价值；
- 也能暴露 typed carrier 观测是否不足。

### 5.4 W4：multimodal workload

目标：

- 观察 `mm_hashes + placeholder layout + text prefix` 共同作用下的 exact prefix 稳定性。

建议特征：

- 单图、多图
- 图像不变、文本 suffix 变化
- 图像顺序变化
- placeholder 布局变化

优先原因：

- 最能测试 typed contract 的必要性；
- 也是工业 prompt caching 文档明确强调的场景。

### 5.5 W5：APC 高命中但 residual 碎片化 workload

目标：

- 专门测 post-hit execution inefficiency。

建议特征：

- 大量共享长前缀
- residual 只剩很短、但长度不规则的补算
- batch shape 波动较大

优先原因：

- 这是 `ResidualModeKV` 是否成立的核心场景。

## 6. 插桩原则

本轮 measurement 只做 **P0 instrumentation**：

- 只加观测
- 不改策略
- 不改语义

原则如下：

1. 每个新指标都要能落到明确代码位置。
2. 先记录 request-level 事件，再做聚合。
3. 优先在已有 metrics 管道上扩展，不另起一套复杂日志框架。

## 7. 代码插桩点

下面这些位置是当前最值得下手的插桩点。

### 7.1 输入进入 engine 的 typed 边界

- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:255)
- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:286)
- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:321)

这里可以记录：

- carrier type
  - `tokens`
  - `embeds`
  - `multimodal`
- `cache_salt` 是否存在
- multimodal item 数量
- placeholder layout 摘要
- prompt length

### 7.2 responses / tool-rich 请求构造边界

- [responses/utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/utils.py:95)
- [responses/serving.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/serving.py:715)

这里可以记录：

- 请求是否带 tool
- 是否带前一轮 reasoning / output
- 旧 system message 是否被过滤
- cache salt 是否启用

这组指标用于定位 `Cacheability ABI` 层的 breaker。

### 7.3 prefix hit 计算边界

- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:176)
- [metrics/stats.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/stats.py:115)
- [metrics/loggers.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/loggers.py:172)
- [metrics/loggers.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/loggers.py:1083)

这里可以记录：

- request-level queried tokens
- prefix-hit tokens
- preempted / non-preempted
- typed carrier 标签
- breaker reason

当前已有 prefix cache queries/hits 计数，但还缺：

- typed breakdown
- breaker attribution
- drift category

### 7.4 typed extra keys 生成边界

- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:369)
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:471)
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:497)

这里可以记录：

- 当前 block 使用了哪些 extra keys
  - `mm`
  - `lora`
  - `cache_salt`
  - `prompt_embeds`
- block 级 typed composition

这组指标是 `Typed Exact Prefix Contract` 的观测核心。

### 7.5 命中后的 scheduler / residual 边界

- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:919)
- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:993)

这里可以记录：

- 每个请求本轮 `num_scheduled_tokens`
- residual token 数
- common prefix blocks
- 是否 resumed / preempted

### 7.6 graph / cascade 决策边界

- [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:3593)
- [flash_attn.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/attention/backends/flash_attn.py:1043)

这里可以记录：

- 实际 `CUDAGraphMode`
  - `FULL`
  - `PIECEWISE`
  - `NONE`
- `use_cascade_attn` 是否启用
- disabled reason
  - prefix 太短
  - query 太少
  - DCP
  - local/sliding/alibi
- batch padding 比例

这组指标是 `ResidualModeKV` 的 measurement 核心。

## 8. Trace Schema

建议至少定义三张逻辑表。

### 8.1 Request-level trace

字段建议：

- `request_id`
- `workload_family`
- `carrier_type`
- `has_cache_salt`
- `has_tools`
- `num_mm_items`
- `prompt_tokens`
- `shared_prefix_tokens_expected`
- `prefix_hit_tokens_actual`
- `breaker_reason`
- `preempted`
- `ttft_ms`
- `e2e_ms`

### 8.2 Block-level typed trace

字段建议：

- `request_id`
- `block_idx`
- `extra_keys_bitmap`
- `has_mm`
- `has_prompt_embeds`
- `has_cache_salt`
- `block_hit`
- `parent_hit`

### 8.3 Post-hit execution trace

字段建议：

- `request_id`
- `residual_tokens`
- `num_common_prefix_blocks`
- `cudagraph_mode`
- `graph_replay_success`
- `cascade_eligible`
- `cascade_enabled`
- `cascade_disabled_reason`
- `padding_tokens`
- `padding_ratio`

## 9. Breaker 分类

如果不先定义 breaker taxonomy，`Cacheability ABI` 很容易沦为空话。

建议至少分成下面几类：

1. `MESSAGE_ORDER_DRIFT`
2. `SYSTEM_PROMPT_DRIFT`
3. `TOOL_SCHEMA_DRIFT`
4. `TOOL_CHOICE_DRIFT`
5. `TOOL_RESULT_DRIFT`
6. `REASONING_HISTORY_DRIFT`
7. `MM_LAYOUT_DRIFT`
8. `MM_CONTENT_DRIFT`
9. `CACHE_SCOPE_DRIFT`
10. `PROMPT_EMBEDS_DRIFT`
11. `UNKNOWN`

P0 阶段即使先做近似归因，也比没有归因强得多。

## 10. Baseline 与对照组

本轮 measurement 至少要有下面四组。

### B0：当前 vLLM baseline

- 不加任何新策略
- 仅记录已有公开指标

### B1：vLLM + P0 instrumentation

- 不改变运行结果
- 只增加 typed / breaker / post-hit trace

### B2：理想化 stable-prefix replay

- 人工控制输入组织，尽可能保证 exact prefix 稳定
- 用于估计“ABI 层的上界收益”

### B3：理想化 residual-oracle analysis

- 不立即实现 selector
- 只离线分析：
  - 如果换 graph mode
  - 如果允许 bounded recompute
  - 如果以更优 bucket 对齐
  - 理论上还能省多少

这样可以在不写完整策略前，先判断 `ResidualModeKV` 是否值得做。

## 11. 核心指标

### 11.1 命中相关

- prefix hit tokens
- prefix hit ratio
- typed carrier hit ratio
- breaker 分布
- drift-induced avoidable miss ratio

### 11.2 延迟相关

- TTFT
- p95 TTFT
- p99 TTFT
- end-to-end latency

### 11.3 吞吐相关

- requests/s
- output tokens/s

### 11.4 执行路径相关

- cudagraph mode 分布
- graph replay 成功率
- cascade eligible 率
- cascade enabled 率
- padding ratio

### 11.5 解释性相关

- miss 中有多少能被 breaker taxonomy 解释
- typed miss 中有多少能定位到特定 carrier

这一类指标很关键，因为它决定 `ABI / contract` 是否是系统贡献，而不只是性能优化。

## 12. 预期结果与论文价值映射

### 12.1 如果 W1/W2 上 drift 很强

说明：

- `Cacheability ABI` 不是空话；
- 有机会把问题定义抬到“输入稳定性是 serving 接口问题”。

### 12.2 如果 W3/W4 上 typed miss 与解释缺口很强

说明：

- `TypedPrefixKV` 仍有价值；
- 论文可以强调统一 exactness semantics，而不是单点 feature 支持。

### 12.3 如果 W5 上 post-hit inefficiency 很强

说明：

- `ResidualModeKV` 值得进入最小策略实现；
- 论文可以把 execution realization 作为第三层，而不是附录优化。

### 12.4 如果三者在同一 workload 家族共同出现

这才是最理想结果。

它意味着：

- 论文可以讲“闭环系统”；
- 而不是三个 loosely related optimization。

## 13. Kill Criteria

P0 做完后，出现下面任意两条，应考虑终止该 thesis。

1. tool-rich / multimodal / embed-rich workload 中，可避免 miss 比例仍然很低。
2. typed carrier 相关问题主要已经被现有实现解决，缺少显著缺口。
3. post-hit inefficiency 带来的理论上界收益很小。
4. breaker taxonomy 无法稳定归因大部分 miss。
5. 最终数据只能支撑一条单线，支撑不了闭环 thesis。

## 14. P0 之后的最小推进顺序

如果 measurement 支持继续，建议按下面顺序走。

### P1：只做 observability + lightweight enforcement

目标：

- 补齐 breaker attribution
- 暴露 typed hit/miss metrics
- 建最小 cacheability ABI

### P2：收紧 typed exact-prefix contract

目标：

- 统一 carrier canonicalization
- 统一 typed extra key 语义
- 明确 publication / invalidation / observability

### P3：实现 post-hit selector

目标：

- 在不改 kernel 的前提下，
- 先做基于 residual、graph mode、cascade eligibility 的轻量 selector。

## 15. 当前建议

当前建议非常明确：

- **先做 P0 instrumentation，不要直接写完整机制。**

因为现在缺的不是好听的设计，而是：

- 这条 thesis 到底有没有足够强的数据支撑。

如果 measurement 结果弱，越早知道越好；
如果 measurement 结果强，再进入实现阶段也不晚。

## 16. 附：当前最有价值的文件抓手

- [responses/utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/utils.py:95)
- [responses/serving.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/serving.py:715)
- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:255)
- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:286)
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:369)
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:471)
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:497)
- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:176)
- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:919)
- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:993)
- [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:3593)
- [flash_attn.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/attention/backends/flash_attn.py:1043)
- [metrics/stats.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/stats.py:115)
- [metrics/loggers.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/loggers.py:172)
- [metrics/loggers.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/loggers.py:1083)

## 17. 实验矩阵建议

为了避免 measurement 最后变成零散日志，建议从一开始就固定一个二维实验矩阵。

### 17.1 横轴：workload family

- `W1` 长 system prompt + 多轮 chat
- `W2` tool-rich / agentic workload
- `W3` prompt embeds workload
- `W4` multimodal workload
- `W5` APC 高命中但 residual 碎片化 workload

### 17.2 纵轴：要回答的问题

- `Q1` drift 是否存在
- `Q2` typed exactness 是否存在解释缺口
- `Q3` post-hit inefficiency 是否明显
- `Q4` 三类问题是否能在同一 story 中闭环

### 17.3 期望输出

每个 workload family 至少给出：

1. 一个 baseline trace
2. 一个 controlled-drift 版本
3. 一个 typed breakdown
4. 一个 post-hit breakdown

最终形成一张总表，回答：

- 哪些 workload 支撑 ABI
- 哪些 workload 支撑 typed contract
- 哪些 workload 支撑 post-hit selector

## 18. P0 插桩任务拆解

为了让 measurement 可以直接执行，建议把 P0 拆成四个最小任务。

### T1：输入侧 typed trace

目标：

- 在请求进入 engine 时记录 carrier 形态与 cache 相关元数据。

建议文件：

- `vllm/v1/engine/input_processor.py`
- `vllm/entrypoints/openai/responses/utils.py`
- `vllm/entrypoints/openai/responses/serving.py`

建议产出：

- request-level typed trace
- tool / responses 边界 trace

### T2：prefix-hit 与 breaker trace

目标：

- 把“命中了多少”扩展为“为什么没命中”。

建议文件：

- `vllm/v1/core/kv_cache_manager.py`
- `vllm/v1/metrics/stats.py`
- `vllm/v1/metrics/loggers.py`

建议产出：

- typed hit/miss breakdown
- breaker taxonomy 统计

### T3：block-level typed composition trace

目标：

- 记录 block hash 到底由哪些 typed extra keys 组成。

建议文件：

- `vllm/v1/core/kv_cache_utils.py`

建议产出：

- block-level extra keys bitmap
- carrier composition 分布

### T4：post-hit execution trace

目标：

- 记录 APC 命中后 residual 如何影响 graph/cascade/padding。

建议文件：

- `vllm/v1/core/sched/scheduler.py`
- `vllm/v1/worker/gpu_model_runner.py`
- `vllm/v1/attention/backends/flash_attn.py`

建议产出：

- residual-length histogram
- graph mode 分布
- cascade disabled reason
- padding ratio

## 19. Measurement 产出物

P0 完成后，建议至少产出下面五个文件或图表集合。

1. `prefix_drift_breakdown.csv`
   - 每类 breaker 的请求数、token 数、占比。
2. `typed_hit_breakdown.csv`
   - 按 carrier 分类的 queries / hits / misses。
3. `post_hit_execution_breakdown.csv`
   - residual、graph mode、cascade、padding 的联合统计。
4. `measurement_summary.md`
   - 用论文语言总结每个 workload family 的结论。
5. `go_no_go_decision.md`
   - 直接对照本计划里的 go / no-go 条件给结论。

## 20. 当前最推荐的执行顺序

如果下一步开始真正做 P0，我建议按下面顺序落地：

1. 先做 `T1 + T2`
   - 因为先要知道 drift 和 typed miss 是否真的存在。
2. 再做 `T4`
   - 因为 post-hit inefficiency 是第三层是否成立的关键。
3. 最后再补 `T3`
   - 它最偏解释性，不是最早期 blocker。

这个顺序的目的很明确：

- **先验证 thesis 是否值得继续，再补更细的内部解释。**

## 21. 与 `p0_instrumentation_spec.md` 的关系

这份文档回答的是：

- 为什么要做 P0
- 要测什么
- 用什么 workload 和指标做 go / no-go

而 [p0_instrumentation_spec.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/p0_instrumentation_spec.md) 回答的是：

- 第一批代码具体改什么
- 新增字段放哪里
- JSONL 怎么长
- 哪些文件需要最小 diff

两者的关系可以理解为：

- `measurement_plan.md` 定目标
- `p0_instrumentation_spec.md` 定实现
