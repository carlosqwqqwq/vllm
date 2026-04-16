# TwinPathServe Minimal Architecture

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责锁定 `FalseNegativeKV` 的 **P1 最小系统形态**。

它不再讨论：

- 要不要做单 worker 动态切 backend
- 要不要把更多 mismatch 一起并入第一版
- 要不要顺手扩展成 distributed cache pool

从现在开始，`FalseNegativeKV` 的第一版系统固定叫：

- `TwinPathServe`

并且系统形态固定为：

- `reuse_optimized_pool`
- `throughput_optimized_pool`

这是双池系统，不是单 worker 内动态换 backend。

## 2. 系统一句话定义

`TwinPathServe` 的核心定义固定为：

> 一个以 `reuse contract` 为中心的双池 serving runtime。
> 它先识别 request 的 `logical hit` 风险，再把请求路由到 `reuse_optimized_pool` 或 `throughput_optimized_pool`，
> 并只用两个 `bridge operator` 恢复第一批 false-negative hit。

## 3. 固定系统边界

### 3.1 系统目标

`TwinPathServe` 只解决下面两类 false-negative miss：

1. `prompt_logprobs` 造成的 observation mismatch
2. batch-invariant fast-path 造成的 backend mismatch

### 3.2 系统不做什么

第一版明确不做：

- carrier mismatch 的在线恢复
- pooling / partial prefill 的在线恢复
- 跨池共享 KV block
- 执行中途的 request migration
- 单请求内部动态切 backend

## 4. 固定组件

`TwinPathServe` 只有下面五个固定组件：

1. `route key builder`
   - 从请求派生 `prefix family`、`API contract class`、`backend compatibility class`、`false-negative risk class`
2. `prefix family table`
   - 维护 family 与 pool 的绑定关系，以及最近的 hit 证据
3. `reuse contract registry`
   - 记录 producer / consumer 的复用能力边界
4. `bridge operator dispatcher`
   - 只分发第一版允许的两个 bridge
5. 双执行池
   - `reuse_optimized_pool`
   - `throughput_optimized_pool`

除了这五块，不再引入第六个高层控制面。

## 5. 双池职责锁定

## 5.1 `reuse_optimized_pool`

它的职责固定为：

- 优先保证 `physical hit`
- 使用 APC-friendly backend / config
- 承担 `prompt_logprobs` selective replay
- 承担高 false-negative 风险 family 的稳定归属

它不负责：

- 追求所有 request 的最高裸 throughput
- 覆盖所有异构 API 特化

## 5.2 `throughput_optimized_pool`

它的职责固定为：

- 优先保证通用吞吐
- 保留最激进的 fast-path / backend 选择
- 承担低 false-negative 风险或冷 family 的默认执行

它不负责：

- 执行 `prompt_logprobs` selective replay
- 为高 reuse 风险 request 强行保留 APC 友好路径

## 6. 固定 route key

从现在开始，`TwinPathServe` 的 router 只允许读下面四个 key，不再添加第五个主 key：

## 6.1 `prefix family`

固定定义：

- 由同一稳定 prefix 指纹导出的 family 标识
- 指纹必须包含当前 cache key 已考虑的 carrier 相关信息
- P1 只需要 family 级一致性，不需要 block-level 细粒度决策

P1 的固定语义是：

- `known family`
  - 在系统中已经出现过，并留下过非零 `physical_hit_tokens` 或明确的高风险归属证据
- `unknown family`
  - 系统从未见过，或尚无可信 hit 证据

## 6.2 `API contract class`

固定枚举直接复用 instrumentation：

- `generation`
- `prompt_logprobs`
- `pooling`

## 6.3 `backend compatibility class`

P1 固定只允许下面四类：

- `dual_compatible`
  - 两个 pool 都可执行，且不会天然丢失 exact-prefix reuse
- `reuse_preferred`
  - 在 `reuse_optimized_pool` 中更可能保住 `physical hit`
- `throughput_preferred`
  - 在 `throughput_optimized_pool` 中更适合跑通用吞吐路径
- `unknown`
  - 当前无法可靠分类

## 6.4 `false-negative risk class`

P1 固定只允许下面五类：

- `low`
- `observation_mismatch`
- `backend_mismatch`
- `carrier_mismatch`
- `unknown`

解释：

- `observation_mismatch`
  - 主要对应 `prompt_logprobs_skip`
- `backend_mismatch`
  - 主要对应 `backend_incompatibility`
- `carrier_mismatch`
  - 主要对应 `prompt_embeds` / multimodal 等测量面

## 7. 固定路由规则

`TwinPathServe` 的路由流程固定为四步：

1. 构建 route key
2. 查询 `prefix family table`
3. 先按 `false-negative risk class` 判定主池
4. 再用当前 pool 负载做同类池内负载均衡

### 7.1 默认规则

默认情况下：

- `unknown family` + `low` risk
  - 进入 `throughput_optimized_pool`
- `known family` + `observation_mismatch`
  - 进入 `reuse_optimized_pool`
- `known family` + `backend_mismatch`
  - 进入 `reuse_optimized_pool`
- `carrier_mismatch`
  - 进入 `throughput_optimized_pool`
  - 只做 measurement，不做在线恢复
- `pooling`
  - 进入 `throughput_optimized_pool`
  - 第一版不扩面

### 7.2 family 绑定规则

P1 固定采用“family 绑定”而不是“每请求重新猜测”：

- 一个 `prefix family` 一旦被判定为高 false-negative 风险并成功进入 `reuse_optimized_pool`，
  后续同 family 请求默认继续进 `reuse_optimized_pool`
- 只有在 pool 不可用或过载时才允许 fallback

这条规则的目的不是最优调度，而是让第一版先获得可重复、可解释的命中行为。

### 7.3 冷启动规则

P1 冷启动固定采用保守策略：

- 新 family 默认先去 `throughput_optimized_pool`
- 但如果当前请求 `API contract class = prompt_logprobs`，则允许直接进入 `reuse_optimized_pool`

原因很简单：

- 对 `prompt_logprobs` 请求，错过第一次 family pinning 的代价太高
- 对其他未知 request，P1 优先避免把所有流量都吸入 reuse pool

## 8. 固定 bridge operator

P1 只允许两个 bridge operator，不再加第三个。

## 8.1 bridge 1：`prompt_logprobs` selective replay

### 适用条件

同时满足下面四条时才允许触发：

- `API contract class = prompt_logprobs`
- `false-negative risk class = observation_mismatch`
- request 已被路由到 `reuse_optimized_pool`
- 存在可用 exact prefix 命中证据

### 固定动作

动作固定为：

- 尽量复用已有 prefix KV
- 只 replay 满足 `prompt_logprobs` 所需 observation 的最小 tail
- 不做整段 full-prompt replay，除非 selective replay 前提失效

### 固定 fallback

如果 selective replay 不能安全执行，则：

- 留在 `reuse_optimized_pool`
- 退回该 pool 的 full prefill
- 不允许中途迁移到 `throughput_optimized_pool`

## 8.2 bridge 2：batch-invariant fast-path twin-pool route

### 适用条件

同时满足下面三条时才允许触发：

- `false-negative risk class = backend_mismatch`
- `backend compatibility class = reuse_preferred`
- `prefix family = known family`

### 固定动作

动作固定为：

- 让高 reuse 风险 family 进入 `reuse_optimized_pool`
- 让低风险或未知 family 留在 `throughput_optimized_pool`
- 不在单个 worker 内动态切换 backend

### 固定 fallback

如果 `reuse_optimized_pool` 不可用或超载，则：

- 请求退回 `throughput_optimized_pool`
- 记录 `route_fallback_reason`
- 该次请求不尝试后台二次迁移

## 9. 固定 failure mode 与 fallback

这部分从现在开始视为定案，不给实现者二次拍板空间。

| failure mode | 固定处理 |
| --- | --- |
| `reuse_optimized_pool` 过载 | 单次回退到 `throughput_optimized_pool`，记录 `route_fallback_reason=pool_overload` |
| `reuse_optimized_pool` 故障 | family 绑定暂时失效，退回 `throughput_optimized_pool`，记录 `route_fallback_reason=pool_unavailable` |
| selective replay 前提不满足 | 留在 `reuse_optimized_pool` 做 full prefill，记录 `bridge_fallback_reason=replay_precondition_failed` |
| route key 无法分类 | 进入 `throughput_optimized_pool`，记录 `false-negative risk class=unknown` |
| family 绑定过时 | 不迁移历史请求；只在下一次 admission 时更新 family 归属 |

固定原则：

- 所有 fallback 都发生在 request admission 或 prefill 起点
- 一旦 prefill 开始，P1 不做 request migration

## 10. 与当前 `vLLM V1` 的实现映射

`TwinPathServe` 不要求重写 `vLLM` 核心调度器；它的最小实现应挂在当前已有入口上。

### 10.1 route key 构建

落点固定在请求进入 `Request` 和 input processor 的边界：

- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:59)
- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:257)

这里负责产出 route 所需的 typed tags，而不是直接做池间调度。

### 10.2 request 预处理

`EngineCore` 当前会在预处理阶段把 `EngineCoreRequest` 转成 `Request`：

- [core.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/core.py:739)

`TwinPathServe` 不应把复杂路由逻辑塞进这里；这里最多只消费已经写好的 route key。

### 10.3 多 engine 路由

当前多 engine 入口已经存在 `get_core_engine_for_request`：

- [core_client.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/core_client.py:1296)
- [core_client.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/core_client.py:1350)

P1 的固定做法是：

- 把 `reuse_optimized_pool` 和 `throughput_optimized_pool` 实现成两组 engine
- 在 `get_core_engine_for_request` 中先按 route key 选池，再在池内做负载均衡

### 10.4 pool 内调度

pool 内部继续复用现有 `Scheduler`：

- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:67)

P1 不改 scheduler 语义，只改跨 pool admission。

## 11. 固定 out-of-scope

下面这些从现在开始明确写成 out-of-scope：

- PIC / arbitrary-position reuse
- hidden cache / activation restoration
- workflow-aware KV retention
- cross-model prefill sharing
- distributed cache pool

补充说明：

- `prompt_embeds` 与 multimodal 在 P0 中必须测量，但不属于 P1 在线恢复面
- pooling / partial prefill mismatch 继续保留为后续扩展，不进入当前最小系统

## 12. 论文层面的作用

`TwinPathServe` 的作用不是证明“所有 false-negative miss 都能修”。

它的作用固定为：

1. 证明 `FalseNegativeKV` 不是只会做 measurement 的问题命名
2. 证明同一 `reuse contract` / `bridge operator` 框架能容纳两类不同 mismatch
3. 证明双池系统比 scattered backend / feature patch 更像一个完整 runtime 设计

如果这三条无法成立，就不应把 `TwinPathServe` 当成 `OSDI/CCF-A` 级系统贡献。

## 13. admission 状态机与系统不变量

为了避免 `TwinPathServe` 被实现成“若干路由特判”，P1 从现在开始固定采用 admission-first 状态机。

### 13.1 admission 状态机

每个请求只允许经历下面五个状态：

1. `classify`
   - 生成 route key
2. `family_lookup`
   - 查询 `prefix family table`
3. `pool_select`
   - 选择 `reuse_optimized_pool` 或 `throughput_optimized_pool`
4. `bridge_plan`
   - 仅在选中 `reuse_optimized_pool` 时尝试 bridge 预案
5. `execute`
   - 一旦 prefill 开始，状态机结束，不允许跨池迁移

这条状态机的意义是：把所有复杂性都收敛到 prefill 之前，避免 P1 一边做 measurement，一边偷偷变成 request migration 系统。

### 13.2 系统不变量

P1 必须始终满足下面四条不变量：

1. 一个 request 在一次 admission 中只能属于一个 pool。
2. pool 选择必须能够被 route key 与 family table 解释，不能出现“人工特判但 trace 中无证据”的情况。
3. 所有 fallback 都必须在 trace 中带原因标签，不能出现 silent fallback。
4. `reuse_optimized_pool` 的存在目的是保护复用收益，而不是承载所有特殊请求；若大量无关流量进入该 pool，则说明路由规则已经失效。

## 14. 最小原型推进顺序

`TwinPathServe` 不应直接从双池完整上线开始，而应按下面三步推进。

### 14.1 Step 1：离线路由模拟

目标：

- 基于已有 traces 离线重放 route key 与 family binding
- 验证哪些 request 会被送入 `reuse_optimized_pool`

验收：

- `W1` family 能稳定进入 reuse pool
- `W5` family 能在 reference/probe 条件下呈现可解释分流

### 14.2 Step 2：双 engine admission 原型

目标：

- 在 `core_client.py` 的多 engine 入口完成 pool 选择
- 不改变 pool 内 scheduler 语义

验收：

- route decision 可追踪
- fallback 可追踪
- 无 pool 间循环重试

### 14.3 Step 3：与 selective replay 联调

目标：

- `reuse_optimized_pool` 内启用 `prompt_logprobs selective replay`
- `throughput_optimized_pool` 保持高吞吐默认路径

验收：

- `W1` 在 reuse pool 上有收益
- `W5` 至少表现出 route 带来的 logical false-negative 缓解趋势

## 15. 评测与验收要求

`TwinPathServe` 的评测不能只看“路由是否成功”，而要固定回答下面四个问题。

### 15.1 路由是否真的减少了假阴性

核心指标：

- pool 维度的 `physical_hit_tokens`
- family 维度的命中保持率
- `false-negative risk class` 与实际收益是否一致

### 15.2 路由是否引入新的系统成本

核心指标：

- `TTFT`
- `p95/p99 TTFT`
- throughput
- pool overload rate
- fallback rate

### 15.3 统一 substrate 是否优于 scattered fixes

固定对比：

- vanilla `vLLM V1`
- 单独 `prompt_logprobs` patch
- 单独 backend route patch
- `TwinPathServe`

若 `TwinPathServe` 无法在解释力或收益上优于分别修补，则系统贡献不成立。

### 15.4 family binding 是否稳定

需要回答：

- family 归属是否频繁抖动
- 冷启动误判代价有多大
- pool 不可用时 fallback 是否可控

## 16. kill criteria 与降级路径

`TwinPathServe` 是否还能作为主 thesis 的系统部分，从现在开始取决于下面三条。

1. 若双池路由只能改善单一 `prompt_logprobs` workload，而不能给第二类 mismatch 带来任何正收益，则它只能降级为单 bridge 配套机制。
2. 若 family binding 的运行时开销或不稳定性过高，以至于需要引入 request migration 才能维持收益，则 P1 边界失守，系统应降级重做。
3. 若 `TwinPathServe` 相比 ad hoc fixes 既没有明显更强的解释力，也没有更好的 end-to-end 指标，则不应继续把它包装为完整 runtime 设计。
