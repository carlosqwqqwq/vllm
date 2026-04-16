# FalseNegativeKV 全方法准备度审计

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责回答一个问题：

- 在继续跑公开数据集主实验之前，`FalseNegativeKV / TwinPathServe` 这条线到底有没有把“论文方法”实现完整。

它不负责：

- 重新发散新 thesis
- 重写已有 paper narrative
- 用措辞代替尚未完成的系统实现

从现在开始，这份文档的固定用途是：

1. 复核当前 active thesis 是否仍然正确
2. 复核历史 idea 中哪些应被吸收，哪些应被排除
3. 明确当前代码里哪些模块已经是方法本体，哪些还只是原型或 measurement
4. 给出“先补方法，再跑公开 benchmark”的硬性执行顺序

## 2. 先给严格结论

先给不绕弯的结论。

### 2.1 当前 thesis 方向是对的，但系统还没闭环

截至 `2026-04-15`，我认为当前 active thesis 继续保留为：

- 问题名：`FalseNegativeKV`
- 系统名：`TwinPathServe`

这个方向本身是正确的，原因是：

1. 它把前面多轮 idea 中最有学术性的那一部分收敛成了统一问题：
   - 已存在的可复用状态，为什么在 runtime 中没有兑现成实际收益。
2. 它比单点 feature patch 更有论文势能：
   - 不再是“支持某个 API”
   - 而是“解释并恢复 false-negative exact hit”
3. 它比继续发散新方向更值得做厚：
   - 现在最稀缺的不是新点子，而是把现有 thesis 做成完整系统。

但同样必须明确：

- **当前主线的方法层文档已经补齐到了 `measurement -> reuse contract -> realization policy -> TwinPathServe` 四层。**
- **当前仓库里真正 runtime 落地完整的，仍然只有 measurement 基座和第一个 bridge。**
- **`TwinPathServe` 作为统一系统还没有完成。**

因此，今天不能诚实地说：

- “论文方法已经完整实现”

更准确的判断是：

- `FalseNegativeKV` 的问题定义已经站住；
- `reuse contract` 和 `realization policy` 已经有正式规格；
- 第一个 bridge 已经跑通；
- 但 unified runtime 还没有闭环，所以现在还不该把公开 benchmark 主实验当成论文最终主结果。

### 2.2 如果现在直接按完整系统投稿，风险仍然偏高

如果按当前代码状态直接把论文写成“完整 `TwinPathServe` 系统已经成立”，风险很高。

最主要的原因不是问题不对，而是方法没有完全落地：

1. 第二类主机制还没有在 runtime 中完成统一实现；
2. 双池路由还停留在 architecture 文档；
3. ad hoc baseline 与 unified baseline 还没有在同一 runner 中闭合；
4. 公开 benchmark runner 仍主要是 measurement-first，不是 full-method runner。

因此，当前最合理的学术判断是：

- 这条线**有潜力冲 CCF-A**；
- 但前提是先把“方法”做完整，而不是继续在 measurement runner 上累积更多公开 benchmark 数字。

## 3. 为什么最终主线应当是 FalseNegativeKV，而不是回退到旧 idea

这里不重开选题，只讨论与当前主线直接相关的五条历史脉络。

### 3.1 `ObservationKV / ObservationSidecarKV`

它们留下来的正确部分是：

- `prompt_logprobs`、pooling、token-level observation 这类 mixed API 请求会制造真实的 observation mismatch；
- “已有 KV 不等于能直接返回 observation” 是一个真实系统缺口；
- `minimal observation sidecar` 是可以成立的 bridge 思路。

它们不应继续保留为独立 thesis 的原因是：

- 单独写容易退化成 feature patch；
- 很容易被 reviewer 理解成：
  - “支持更多 API”
  - “支持 `prompt_logprobs` + APC”

所以它在当前主线里的正确位置应该固定为：

- `FalseNegativeKV` 下的第一类 mismatch
- 也就是 `observation mismatch`

### 3.2 `InvariantKV`

它留下来的正确部分是：

- 执行后端、batch invariance 和 fast path 会导致 exact prefix reuse 语义漂移；
- 这不是单后端 bug，而是 runtime capability boundary 问题；
- “更快后端”与“更强复用”之间的张力是真实的。

它不应继续独立成 thesis 的原因是：

- 单独写容易变成 backend engineering paper；
- 很容易被打成“修 fast backend 的 APC 兼容”。

所以它在当前主线里的正确位置应该固定为：

- `FalseNegativeKV` 下的第二类 mismatch
- 也就是 `backend mismatch`

### 3.3 `EligibilityKV`

它留下来的正确部分是：

- 现有系统对 APC 的启停仍然过于保守；
- “能不能复用”不该只是黑名单和硬编码；
- 需要一个显式的、可解释的 `cacheability / bridgeability` 判断层。

它不应继续独立成 thesis 的原因是：

- 单独写会显得过于协议化；
- 如果没有强实例机制支撑，容易像 configuration cleanup。

所以它在当前主线里的正确位置应该固定为：

- `FalseNegativeKV` 的统一抽象层
- 也就是：
  - `reuse contract`
  - `bridgeable miss`
  - `route key`
  - `false_negative_reason taxonomy`

### 3.4 `WritePathKV`

`WritePathKV` 的问题本身有学术性，但它不应该被并入当前主方法。

原因很明确：

1. 它讨论的是写路径、发布路径和多层状态流转；
2. 它会把当前 thesis 从“恢复 false-negative exact hit”拉向“跨层 publication policy”；
3. 这会显著扩大发论文所需的系统面，直接拖慢当前主线。

因此它当前的正确位置应该是：

- 保留为未来可能的独立方向
- **不并入 `FalseNegativeKV` P1/P2 方法闭环**

### 3.5 结论

前面这些 idea 里，真正应该被吸收到当前主线里的只有三层：

1. `ObservationKV / ObservationSidecarKV`
   - 贡献 `observation mismatch`
2. `InvariantKV`
   - 贡献 `backend mismatch`
3. `EligibilityKV`
   - 贡献统一的 contract / bridgeability 抽象

这三层合在一起，才是当前 thesis 最稳的学术内核。

## 4. 当前代码里，哪些已经是“论文方法”，哪些还不是

下面的判断只基于当前仓库代码与已落地实验，不基于未来计划。

## 4.1 已经完成的方法组成

### 4.1.1 measurement 基座：已完成第一版

已经具备：

- request-level JSONL trace
- `logical_hit_tokens / physical_hit_tokens / false_negative_tokens` 等字段
- `false_negative_reason` 归因
- `engine_*_lookup.jsonl`
- `engine_*_replay.jsonl`

这意味着：

- 论文的 measurement layer 已经有可靠的实现基础；
- 后续所有机制实验都不需要重新搭观测层。

### 4.1.2 抽象方法层：已完成文档冻结

目前已经补齐下面两份正式规格：

- `reuse_contract_spec.md`
- `realization_policy_spec.md`

这两份规格的意义是：

1. 已把旧主线最有价值的抽象正式回流到当前 thesis；
2. 已把当前主线固定成：
   - `measurement surface`
   - `reuse contract`
   - `realization policy`
   - `TwinPathServe`
   这条清晰的方法链；
3. 已把后续 runtime 和评测矩阵应该服务什么写死。

这意味着：

- 当前主线已经不再缺“抽象层名字”；
- 当前真正缺的是 runtime 落地，而不是继续发明新的方法名。

### 4.1.3 第一个 bridge：已完成最小可运行实现

已经具备：

- `prompt_logprobs_sidecar_key`
- `prompt_logprobs_use_sidecar`
- `replay_start_token`
- `effective_reuse_tokens`
- worker sidecar cache
- scheduler selective replay planner
- correctness smoke / regression
- `off / on / sidecar-only / replay-budget` 消融

并且已经有真实结果证明：

- 不是只改了 trace；
- 而是真的恢复了 `effective reuse` 和端到端时间收益。

这意味着：

- `observation mismatch` 这一类已经从概念变成了真实方法。

## 4.2 还没有完成的方法组成

### 4.2.1 `TwinPathServe` 统一 runtime：未实现

目前仓库里虽然已经有：

- `enable_twin_path_serve`
- `enable_family_binding`
- `enable_backend_mismatch_route`

这些 config 字段，

但当前它们还没有真正接到 runtime 主路径。

当前缺失的真实模块包括：

1. `route key builder`
2. `prefix family table`
3. `reuse contract registry`
4. `bridge operator dispatcher`
5. `realization policy` 到 runtime 决策面的真实映射
6. `reuse_optimized_pool / throughput_optimized_pool` 的真实 admission / route

也就是说：

- **当前的 `TwinPathServe` 仍然是 architecture 文档，不是已经实现的系统。**

### 4.2.2 第二个主机制：未实现

当前 `backend mismatch` 还停留在：

- measurement evidence
- synthetic / semi-controlled preexperiment
- twin-pool 逻辑证据

但还没有形成：

- 在线 route
- family binding
- fallback trace
- 统一 bridge operator
- `realization_mode` 的统一 runtime 决策

因此当前只能说：

- `backend mismatch` 已经被测出来；

还不能说：

- `TwinPathServe` 已经把它恢复掉了。

### 4.2.3 ad hoc baseline：未实现为统一可切换系统

论文评测矩阵要求同时比较：

- vanilla
- status quo skip
- ad hoc per-feature fix
- unified P1

但当前仓库还没有把这四类 baseline 收敛成：

- 同一 runner
- 同一 config schema
- 同一输出结构

这意味着：

- 当前实验虽然能看趋势；
- 但还不能支撑干净的论文主表。

### 4.2.4 公开 benchmark full-method runner：未实现

目前已经完成的公开 benchmark 路线，主要还是：

- measurement-first prevalence runner

当前还没有完成：

- 带 unified route / bridge 的 full-method public runner
- 对应的 ad hoc baseline runner
- 对应的统一分析脚本

所以当前公开 benchmark 最多能证明：

- 问题存在

还不能作为最终主系统主实验来证明：

- 系统完整收益

## 5. 现在是否足够支撑 CCF-A

### 5.1 如果今天冻结在当前状态：不够

如果现在停止实现，只拿现有代码和结果投稿，

最合理的判断是：

- 可以支撑一篇很强的问题驱动型系统雏形；
- 但还不够稳地支撑完整 `OSDI/CCF-A` 级 unified system thesis。

缺口主要不是写作，而是方法闭环缺两块：

1. 第二类机制实例化
2. 统一 runtime substrate

### 5.2 如果完成下面两件事：有机会够

我认为这条线只要补齐下面两块，就有真实机会达到 CCF-A 级别：

1. 把 `backend mismatch` 做成第二个真实在线机制
2. 把 `TwinPathServe` 做成统一 route / bridge / fallback substrate

一旦这两块成立，论文贡献包就会比较完整：

1. 问题贡献
   - `false-negative exact hit`
2. 抽象贡献
   - `reuse contract / bridgeable miss / route key`
3. 系统贡献
   - `TwinPathServe`
4. 机制贡献
   - `prompt_logprobs selective replay`
   - `backend mismatch twin-pool route`

这时它才更像一篇系统论文，而不是一个成功 patch 加一个设计图。

## 6. 在继续公开 benchmark 之前，必须先补齐哪些模块

从现在开始，公开 benchmark 主实验前的固定前置 gate 应该锁成下面五层。

### 6.1 Gate 1：主方法冻结

必须先冻结论文方法的边界：

- 只保留两类 mismatch
  - `observation mismatch`
  - `backend mismatch`
- 只保留两类机制
  - `prompt_logprobs selective replay`
  - twin-pool backend route
- 不再把 `prompt_embeds` / multimodal 在线恢复并入 P1 主方法

否则越写越大，方法永远收不住。

### 6.2 Gate 2：抽象层与决策层冻结

在跑公开 benchmark 主实验前，必须承认下面这件事已经完成：

1. `reuse contract` 规格冻结
2. `realization policy` 规格冻结
3. 方法层输出与评测矩阵指标一一对应

这一步现在已经基本完成，因此后续不应再把时间花在重新发明高层术语上。

### 6.3 Gate 3：统一 runtime substrate 落地

在跑公开 benchmark 主方法实验前，至少要把下面这些最小模块做出来：

1. `route key builder`
   - `prefix family`
   - `api contract class`
   - `backend compatibility class`
   - `false-negative risk class`
2. `family binding` 最小实现
3. `reuse_optimized_pool / throughput_optimized_pool` 的最小 admission
4. route / fallback / family pinning trace

这一层不需要一步做到最强，但必须真的存在。

### 6.4 Gate 4：第二个机制实例化落地

在跑公开 benchmark 主表前，至少要把下面其中之一做成真实在线系统：

- backend mismatch twin-pool route

这不是可选项。

因为如果只有第一个 bridge，整篇论文很容易被打回：

- “这其实还是 prompt_logprobs 一个 case”

### 6.5 Gate 5：统一 baseline / ablation matrix 落地

在跑公开 benchmark 主表前，必须统一收敛：

1. vanilla
2. status quo skip
3. ad hoc prompt-logprobs fix
4. ad hoc backend fix
5. unified P1

并且要求：

- 同一 runner
- 同一 config
- 同一 summary 口径

如果这一层没有完成，公开 benchmark 结果会越来越多，但论文主表仍然不干净。

## 7. 当前推荐的固定执行顺序

从现在开始，固定执行顺序应该改成下面这样：

1. **先补完整方法，不再优先扩公开 benchmark**
2. 冻结 `reuse contract + realization policy` 的方法层语义
3. 完成 `TwinPathServe` 最小 route substrate
4. 完成 `backend mismatch` 第二个在线机制
5. 完成 unified baseline / ablation runner
6. 最后才大规模跑 `LongBench / BFCL / MTEB` 主实验

更直接一点说：

- 现在最应该补的不是更多 benchmark 数字；
- 而是把“论文方法”从 `1.0 个机制` 补成 `2.0 个机制 + 统一 runtime`。

## 8. 一句最终判断

如果只保留一句最该执行的话，这句应该固定为：

> `FalseNegativeKV` 作为 thesis 是成立的，但当前真正实现完整的还只有 measurement 与第一个 bridge；在 `TwinPathServe` 统一路由和 `backend mismatch` 第二机制落地之前，不应该把公开 benchmark 主实验继续往前推成论文主结果。
