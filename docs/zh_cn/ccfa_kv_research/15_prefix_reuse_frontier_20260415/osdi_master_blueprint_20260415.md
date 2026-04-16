# OSDI 主线联动蓝图

更新时间：2026-04-15

## 1. 严格结论

先把判断说死，不绕弯。

### 1.1 `14_beyond_current_ideas` 的方向是对的

当前 active thesis：

- 问题名：`FalseNegativeKV`
- 系统名：`TwinPathServe`

它抓住了这条研究线最有价值的部分：

> 系统里明明已经存在 exact 或 logical-exact reuse 条件，但 runtime 并没有把它兑现成 `physical hit`。

这个问题本身是成立的，而且相比“再做一个更灵活的 prefix cache”更有边界，也更不容易直接撞到 `EPIC / MPIC / KVLink / KVShare / RelayCaching` 这些近邻路线。

### 1.2 但 `14` 单独拿来，仍然偏薄

就当前文档和实现状态看，`14` 更像：

- 一个强问题定义
- 一套 measurement-first 基座
- 一个已经跑通的第一个 bridge
- 一个尚未闭环的第二个 bridge 计划

如果直接拿它去支撑 `CCF-A / OSDI`，最大风险不是题目错，而是 reviewer 会把它理解成：

- `prompt_logprobs` selective replay
- backend mismatch route
- 再加一套解释语言

也就是：

> 有统一问题名，但还没有足够厚的统一系统层。

### 1.3 正确动作不是换题，而是做纵向联动

最合理的动作不是推翻 `14`，而是把下面三套材料联动起来：

- `12_combined_main_thesis`
  - 提供抽象层
- `14_beyond_current_ideas`
  - 提供 active 问题、主实现、主实验
- `15_prefix_reuse_frontier_20260415`
  - 提供前沿定位和 paper-facing 抬升

只有这样，当前主线才有机会从：

- `measurement thesis`

抬升为：

- `runtime substrate thesis`

---

## 2. 三套目录各自应该负责什么

## 2.1 `12_combined_main_thesis`：抽象层来源，不再是主线

`12` 当前不该被恢复成 active thesis，但它里面最有价值的东西必须回流。

它最值得吸收的不是旧题名，而是三层结构：

1. `Cacheability ABI`
2. `Typed Exact Prefix Contract`
3. `Post-Hit Execution Selection`

这些内容现在不该以旧 thesis 身份存在，而应被翻译成当前主线语言：

- `Cacheability ABI`
  -> `reuse contract / bridgeability contract`
- `Typed Exact Prefix Contract`
  -> `false_negative_reason / prefix_family / route_key`
- `Post-Hit Execution Selection`
  -> `TwinPathServe / realization-aware selection`

因此，`12` 的定位应该固定为：

> 抽象层归档库，而不是候选题库。

## 2.2 `14_beyond_current_ideas`：唯一 active execution branch

`14` 必须继续保持唯一主线。

它应该承担下面四件事：

1. 定义问题与边界
2. 给出 measurement 证据
3. 落地最小系统实现
4. 提供主实验结果

也就是说：

- 所有代码相关推进都应服务 `14`
- 所有 benchmark 结果都应回落到 `14`
- 不再允许在别的目录平行开第二套主系统

## 2.3 `15_prefix_reuse_frontier_20260415`：paper-facing overlay

`15` 不负责替代 `14`，而负责：

1. 解释为什么当前方向没有明显撞车
2. 把内部问题名抬升成更强的论文叙事
3. 帮助决定题目、摘要、贡献表述

其中最关键的抬升是：

- 对内：`FalseNegativeKV`
- 对外：`RealizeKV`

这不是改名游戏，而是 thesis 视角变化：

- `FalseNegativeKV`
  - 更像问题诊断
- `RealizeKV`
  - 更像系统目标
  - 强调把 `counterfactual exact hits` 兑现成 `physical hits`

---

## 3. OSDI 级 thesis 应该长什么样

## 3.1 一句话 thesis

我建议把最终 paper-facing thesis 固定成：

> Modern LLM serving runtimes fail not only because reuse is absent, but because many counterfactual exact hits are never realized under runtime contract mismatch; `RealizeKV` turns these latent hits into physical reuse through contract-aware bridging and realization-aware execution.

中文压缩版：

> 现代 LLM serving 中的大量损失，不是因为没有可复用状态，而是因为 runtime 没有把已经存在的 counterfactual exact hit 兑现成 physical hit；`RealizeKV` 用 contract-aware bridge 与 realization-aware execution 去恢复这部分损失。

## 3.2 三层系统骨架

如果想要达到 `OSDI` 的形状，我建议整篇论文必须固定成下面三层。

### 第一层：Measurement Surface

目标：

- 证明问题不是 anecdotal bug。

核心对象：

- `logical hit`
- `physical hit`
- `false-negative hit`
- `bridgeable miss`
- `counterfactual_hit_tokens`
- `realization_gap`

这一层现在主要由 `14` 已经具备的 instrumentation 提供支撑。

### 第二层：Unified Contract Layer

目标：

- 证明这不是两个无关 patch，而是统一 contract 问题。

核心对象：

- `reuse contract`
- `false_negative_reason`
- `prefix_family`
- `route_key`
- `bridge_operator`

这一层需要从 `12` 吸收抽象，并在 `14` 中正式立起来。

### 第三层：Realization Runtime

目标：

- 证明系统不仅能解释问题，还能以统一控制面恢复收益。

核心对象：

- `TwinPathServe`
- `selective replay`
- `observation sidecar`
- `family binding`
- `reuse-friendly pool / throughput-friendly pool`
- `realization-aware selection`

这一层是当前最缺的部分，也是后续必须补厚的部分。

---

## 4. 当前主线距离 OSDI 还差什么

## 4.1 差的不是更多 benchmark，而是方法闭环

当前最大短板不是：

- 公开 benchmark 数量不够多

而是：

- 方法还没从 “两个恢复实例” 上升为 “一个统一 runtime substrate”

因此，后续工作顺序必须是：

1. 先补方法闭环
2. 再补主实验

而不是反过来。

## 4.2 目前已经够强的部分

下面这些部分已经具备论文势能：

1. 问题定义已经站住
2. measurement-first 路线是对的
3. `prompt_logprobs` selective replay 已经是实质性 bridge，而不是纯 trace
4. backend mismatch 已经有明确 runtime 现实基础

## 4.3 目前还不够强的部分

下面这些部分是必须补齐的：

1. `reuse contract` 还没有成为正式方法层
2. `TwinPathServe` 还没有真正成为统一 runtime，而更像 architecture
3. `realization-aware selection` 还没有形成成本模型或统一决策面
4. `W1/W2/W5` 还没有被组织成一条强证据链
5. 当前叙事仍更偏 `FalseNegativeKV` 内部问题名，而不是更强的 paper-facing thesis

---

## 5. 三套材料如何联动

## 5.1 `12 -> 14`：抽象回流

这是最重要的一步。

固定吸收关系如下：

| `12` 里的内容 | 在当前主线里的新位置 |
| --- | --- |
| `Cacheability ABI` | `reuse contract` |
| `Typed Exact Prefix Contract` | `false_negative_reason + route key + prefix_family` |
| `Post-Hit Execution Selection` | `TwinPathServe` |

原则：

- 不恢复旧题名
- 只恢复旧抽象
- 所有回流后的内容都必须服务 `14`

## 5.2 `14 -> 15`：问题抬升

`14` 负责给出：

- 问题证据
- 机制实例
- 主实验

`15` 负责把它抬升成更适合投稿的叙事：

- 从 `false-negative exact hit`
  抬升成
- `counterfactual exact hit realization`

原则：

- 对内保留 `FalseNegativeKV`
- 对外逐步用 `RealizeKV`

## 5.3 `15 -> 14`：边界约束

`15` 不是只做包装，它还负责反向约束主线，防止扩题。

固定约束如下：

- 不并入 PIC
- 不并入 distributed cache pool
- 不并入 hidden cache substrate
- 不并入 cross-model sharing
- 不并入 arbitrary-position reuse

这样才能保住 exact / logical-exact 这条清晰边界。

---

## 6. 需要新增的四个工作包

如果目标是 `CCF-A / OSDI`，我建议后续只允许推进下面四个工作包。

## WP1：把 `reuse contract` 正式写成方法层

目标：

- 不再把 contract 只留在零散字段和 route key 里。

最小要求：

- 明确 contract 输入维度
- 明确 contract 输出维度
- 明确哪些 mismatch 属于 contract mismatch
- 明确 contract 如何决定 bridgeability

成功标准：

- reviewer 不能再把整篇论文理解成 patch list。

## WP2：把 `TwinPathServe` 从 architecture 变成 runtime

目标：

- 让 `TwinPathServe` 不再只是设计图，而是统一执行层。

最小要求：

- route 决策有真实 runtime 语义
- family binding 有真实生命周期
- bridge operator dispatcher 存在
- fallback / rollback 有统一 trace

成功标准：

- `prompt_logprobs` recovery 和 backend mismatch recovery 属于同一个 runtime。

## WP3：把 `W1/W2/W5` 组织成主证据链

目标：

- 不只是覆盖 workload family，而是形成“递进证明”。

推荐角色：

- `W1`
  - 证明最基础 logical hit 被 conservative contract 吃掉
- `W2`
  - 证明这不是 toy case，而是 agentic 主战场问题
- `W5`
  - 证明 realization 不只是 API mismatch，还涉及 backend choice

成功标准：

- 三者共同支撑“统一 runtime pathology”这个说法。

## WP4：引入 realization cost model

目标：

- 把论文从“恢复 hit”提升到“恢复收益”。

最小要求：

- 明确 `bridge_cost`
- 明确 `realized_gain`
- 明确何时值得 replay / sidecar / reroute

成功标准：

- 论文不再只是说：
  - “我们恢复了多少 hit”
- 而是能说：
  - “我们在受控代价下实现了更高 realization efficiency”

---

## 7. 最终 claim 应该如何分层

为了避免写过头，我建议把 claim 固定成三层递进。

## 7.1 当前安全 claim

这是当前最稳的说法：

> false-negative exact hits 在真实 workload 中存在，且至少两类 mismatch 已可被 measurement surface 识别。

## 7.2 下一阶段 claim

这是 `WP1 + WP2` 完成后的说法：

> observation mismatch 与 backend mismatch 可以被统一的 contract-aware runtime 容纳与恢复。

## 7.3 最终 OSDI claim

只有 `WP3 + WP4` 也站住以后，才建议写成：

> exact-prefix serving should be understood as a realization problem rather than a cache lookup problem; `RealizeKV` provides a substrate that measures, bridges, and schedules counterfactual exact hits into physical reuse.

---

## 8. 明确哪些事情不要做

为了守住主线，下面这些事情现在都不该做。

### 8.1 不要横向扩题

特别不要并入：

- PIC
- multimodal arbitrary-position stitching
- distributed prefix cache pool
- cross-model prefill sharing

这些内容只会让问题面变大，不会让 thesis 变厚。

### 8.2 不要继续发散新高层 idea

从现在开始，`ccfa_kv_research` 下不应该再新增一轮“新方向比较”文档。

原因很简单：

- 现在缺的不是新题
- 而是把现有主线闭环

### 8.3 不要把 sidecar 扩成第二套 hidden cache 系统

sidecar 当前的正确定位只能是：

- 最小 bridge state

一旦扩成通用 hidden-state cache，整条线就会滑向另一类论文。

---

## 9. 推荐推进顺序

后续最推荐的推进顺序如下。

1. 在文档层正式写清 `12/14/15` 的角色分工
2. 在 `14` 中补一份正式的 `reuse contract` 方法文档
3. 把 `TwinPathServe` 补成真实 runtime 闭环设计
4. 重写主 paper storyline，使其从“问题 + 两个实例”变成“三层 substrate + 两个实例”
5. 重构 evaluation matrix，让 `W1/W2/W5` 成为主证据链
6. 再决定是否扩展第三类 bridge

注意：

- 在第 4 步之前，不建议大量增加 benchmark 数量。
- 在第 5 步之前，不建议写过强的 full-system claim。

---

## 10. 一句话总判断

这份蓝图最终只想固定一句判断：

> `14_beyond_current_ideas` 是当前最正确的 active thesis，但它单独还不足以支撑 `CCF-A / OSDI`；只有把 `12` 的抽象层吸收进来，并借 `15` 完成 paper-facing 抬升，当前主线才会从 measurement thesis 变成 runtime substrate thesis。

这就是接下来所有工作的总原则。
