# FalseNegativeKV Paper Writing Strategy

更新日期：2026-04-16

## 1. 为什么现在要单独写这份文档

`FalseNegativeKV` 现在已经进入一个危险阶段：

- 问题已经足够有意思
- 预实验结果也足够强
- 但系统恢复结果还没有完整闭环

这时最容易犯的错误有两个：

1. 写成 feature patch paper
2. 提前把论文写成“完整系统已经证明胜出”

这份文档的任务，就是把当前最稳的 paper 写法锁住。

## 2. 当前最稳的论文定位

当前最稳的定位不是下面两种：

### 2.1 不能退化成 patch paper

不应写成：

- `prompt_logprobs` 的 prefix cache patch
- batch invariant 的 backend patch
- 若干 vLLM 兼容性修补

这种写法的问题在于：

- 贡献会被打成 patch collection
- 很难支撑 OSDI 级 narrative
- 审稿人会直接追问为什么不分别修

### 2.2 也不能提前写成完整系统闭环

当前也不应写成：

- 我们已经统一恢复了多类 false-negative hit
- 我们已经在完整系统上稳定获得端到端显著收益

因为截至当前：

- `W1` 是直接证据
- `W5` 是强 twin-pool 逻辑证据
- 但 recovery prototype 还没有形成 `H3/H4` 的完整结果链

### 2.3 当前推荐定位

当前最稳的写法是：

> `FalseNegativeKV` is a measurement-backed runtime thesis:
> it identifies and formalizes false-negative exact hits, shows that they are
> not isolated bugs, and argues that `TwinPathServe` is the right contract-aware
> architecture to recover them.

翻成更适合内部统一口径的中文就是：

> 先把 `false-negative exact hit` 写成一个被系统性忽视的 runtime pathology，
> 再把 `TwinPathServe` 写成解决这类问题的统一系统方向，
> 但在 recovery 闭环做完之前，不提前声称完整端到端胜利。

## 2.4 当前应如何提标而不降格

面对高强度系统 reviewer，当前最危险的反应不是“claim 全收回”，
而是把论文主动降成：

- measurement paper
- 加两个 prototype patch

更合理的提标方式是：

1. 收紧结果型 claim
2. 抬高结构型 claim

因此，当前更高、也更稳的写法应当把工作抬成：

- `correctness-aware reuse control plane`

而不是继续停留在：

- `prompt_logprobs bridge + twin-pool route`

更具体地说：

- `prompt_logprobs selective replay`
  应写成一种 `bridge operator`
- `TwinPathServe`
  应写成一个 correctness-aware reuse substrate 的系统实例
- `logical_hit_tokens`
  应写成 counterfactual reuse measurement 所需的 upper-bound oracle

这样做不会把贡献说小，反而会把贡献从“修两个点”
提升成“提出一个此前缺失的 runtime capability layer”。

## 3. 论文一句话主张

当前最稳的一句话 thesis 建议写成：

> Modern LLM serving runtimes optimize physical exact-prefix hits, but fail to
> model logical hits that are lost before computation begins; `FalseNegativeKV`
> turns these false-negative exact hits into a measurable, contract-aware
> systems problem.

这句话比“我们做了一个更好的 prefix cache”更强，因为它：

- 先定义问题
- 再说明为什么现有 runtime 不足
- 最后给出我们的系统切口

## 4. 标题策略

### 4.1 当前最稳标题

如果近期论文仍然只以 `vLLM` 为主实现与证据系统，最稳标题是：

- `FalseNegativeKV: Contract-Aware Recovery of False-Negative Exact Hits in vLLM`
- 中文可对应写成：
  - `FalseNegativeKV：大模型推理中前缀复用假阴性的测量、表征与契约感知恢复`

原因：

- 不会因为范围过大而被抓住泛化问题
- 与当前实现边界一致
- 仍然保留了“问题 + 方法主张”的顶会风格

### 4.2 更大范围标题

如果后续我们能补出更强的跨系统讨论或复现实证，再考虑升级为：

- `FalseNegativeKV: Contract-Aware Recovery of False-Negative Exact Hits in LLM Serving`

当前不建议直接用这个更大标题，因为证据还主要来自 `vLLM`。

## 5. 当前 claim 边界

| 主题 | 当前证据状态 | 当前允许写法 | 当前不允许写法 |
| --- | --- | --- | --- |
| `W1 prompt_logprobs` | 强、直接，且已有 dedicated formal recovery evidence | `prompt_logprobs` can turn logical exact hits into physical misses via conservative reuse contracts | 我们已经统一恢复了所有 observation mismatch |
| `W5 backend mismatch` | vanilla runtime 侧是强逻辑证据，twin-path 侧已有公开 workload 恢复证据 | cross-backend compatibility classes can induce logical false-negative exact hits, and twin-pool routing can recover them on public workloads | runtime already classifies these misses online as `backend_mismatch` |
| `TwinPathServe` | 设计边界已锁定，且已有第一版 formal runtime evidence | we present `TwinPathServe` as the contract-aware runtime design with initial public-workload evidence | we fully demonstrate `TwinPathServe` end-to-end gains |
| 统一 substrate | 叙事上成立，实验上仍需补厚 | scattered fixes fail to explain the common cause of these misses | unified recovery is already empirically superior in all settings |
| 端到端收益 | 尚未闭环 | leave as target evaluation claim | we achieve large TTFT wins with controlled throughput regression |

## 6. 摘要该怎么写

当前摘要必须满足三条：

1. 强写问题，不弱写
2. 强写 measurement，不弱写
3. 谨慎写恢复结果，不提前越权

因此，当前摘要最好的句法顺序是：

1. 背景：exact-prefix reuse 已经成为核心 serving 优化
2. 缺口：现有系统只显式优化 `physical hit`
3. 问题：大量 `logical hit` 会在运行前被丢失
4. 方法：我们给出 measurement surface 和 contract-aware runtime design
5. 当前结论：我们在 `vLLM` 中测到两类强 false-negative 来源，并给出已有 formal evidence 的 `TwinPathServe` 原型恢复路径

当前摘要最后一句可以谨慎升级到：

- 我们已经在两类 mismatch 上给出 formal recovery evidence
- 但多 benchmark 端到端收益仍未完整闭环

## 7. 引言该怎么写

当前引言最稳的写法，不是先讲系统实现，而是先讲下面这条观察：

> 现有 serving 系统把 exact reuse 看成“命中了就赚，没命中就算了”的 opportunistic optimization，
> 但在现代 mixed-API workload 中，更关键的问题是：很多请求本来就应该命中，只是没有一个可复用 contract
> 去把 producer state 变成 consumer-visible reuse。

当前六段式引言应当承担的职责分别是：

1. prefix reuse 已成为 serving 基础设施
2. 现代 workload 让“逻辑命中”和“物理命中”脱钩
3. `prompt_logprobs` 和 backend mismatch 统一暴露出同一种 pathology
4. 我们把它形式化为 `false-negative exact hit`
5. `TwinPathServe` 是围绕 `reuse contract` 的系统化回应
6. 当前贡献是问题表征、测量、以及面向恢复的系统方向

## 8. 当前最应该避免的写法

下面这些句子现在都不应出现在摘要或引言里：

- `We solve false-negative exact hits in vLLM.`
- `We recover multiple mismatch classes with a unified runtime.`
- `Our system significantly improves TTFT while preserving throughput.`
- `FalseNegativeKV is a general solution for prefix cache misses.`

它们的问题分别是：

- 第一条写得像问题已经被完全解决
- 第二条默认 `H3` 已经成立
- 第三条默认 `H4` 已经成立
- 第四条把问题范围扩大成了“所有 miss”

## 9. 当前最应该主动强调的点

当前摘要和引言里最值得主动强调的点有四个：

1. `false-negative exact hit` 不是 prefix cache miss 的同义词
2. 问题的关键在 `logical hit -> physical hit` 之间缺少 runtime contract
3. 这不是单个 feature bug，而是现代 mixed-API serving 的统一症状
4. `TwinPathServe` 的意义首先在于统一 substrate，其次才是具体 bridge

## 9.1 必须主动写清的“发生条件”

如果论文前半部分不明确回答“什么条件下会发生前缀复用假阴性”，reviewer 会默认把问题理解成：

- cache key 没对齐
- prefix hash 没命中
- 或者某个已有特性暂时不支持

因此，背景、动机和测量章必须明确写出：

1. 参考请求已经证明该 prefix family 可以稳定复用。
2. 当前请求与参考请求共享相同的可复用前缀。
3. 当前变化的是消费条件，而不是前缀本身。
4. 当前请求的实际命中显著低于参考请求。

没有这组条件，`FalseNegativeKV` 只是一个漂亮但边界模糊的名字。

## 9.2 必须主动写清的“hash 不是充分条件”

另一个很容易被 reviewer 质疑的点是：

- `KV cache` 既然用 hash，为什么还会“已经存在但没被复用”？

这里论文必须提前把措辞钉死：

- hash 只解决“是不是同一个前缀身份”
- 不解决“当前请求能不能合法消费这份状态”

所以我们不应写：

- `the cache entry already exists and is not reused`

更稳的写法是：

- `equivalent prefix computation has already been established under a reference execution condition, but the current request does not realize that reuse`

中文口径则固定为：

- “语义等价的前缀计算已经在参考执行条件下被证明可复用，但当前请求没有把这部分既有计算兑现为自己的命中收益。”

## 10. 从现在到 full-thesis 版本还差什么

如果我们要把当前 paper draft 升级成真正的 OSDI 主 thesis 版本，还至少差四样东西：

1. `prompt_logprobs selective replay` 的真实恢复实现
2. `TwinPathServe` 双池 route 的在线闭环实现
3. `H3/H4` 对应的端到端结果
4. 统一 substrate 相对 ad hoc fixes 的优势证据

## 10.3 当前已经补上的一条外部效度证据

截至 `2026-04-15`，`LongBench W1` 公开 prevalence runner 已经跑通，
并拿到第一轮正式结果。

这意味着当前文稿在问题表征层不再只依赖：

- `MT-Bench`
- dedicated validation

而已经开始具备：

- 公开长上下文 benchmark 的强问题证据

因此，后续引言和实验章可以更稳地写：

- 我们的问题不仅出现在短 prompt benchmark 上
- 也出现在真实长上下文 public workload 上

但仍然不能提前写成：

- recovery 在多 benchmark 上已经完整闭环

## 10.1 从“小贡献”抬到“系统贡献”还差哪一层

如果 reviewer 觉得这篇论文“贡献偏小”，根因通常不是工作量，而是 paper 没把贡献抬到下面这个层次：

1. 不只是指出两个现象，而是提出一个此前未被清晰识别的问题类。
2. 不只是做两个修补，而是说明为什么现有 runtime 缺少一个必要的 `reuse contract` 层。
3. 不只是修通一条路径，而是证明同一 substrate 能同时容纳两类不同 mismatch。
4. 不只是给出局部收益，而是说明它在端到端指标上有可复现的系统价值。

因此，后续所有写作都要服务于这条“贡献抬升链”。如果某个章节只是在解释实现细节，却没有帮助这四层站稳，那它对顶会写作价值有限。

## 10.2 理论包装的最低闭环

在当前 measurement-first 阶段，理论包装至少要闭环到下面这一步：

- 问题边界明确
- 发生条件明确
- hash objection 被提前消解
- patch-collection objection 被提前消解
- `TwinPathServe` 被写成 runtime capability，而不是方便实现的工程选择

如果这五点没有同时成立，paper 即使实验做得不错，也仍然会显得像 feature patch paper。

在这四项完成前，摘要和引言都应保持：

- 问题写强
- 设计写清
- 收益写克制

## 11. 当前判断

如果今天就必须开始写论文，最稳的路线是：

1. 先用 measurement-first 口径写摘要和引言
2. 先把 `W1` 作为 motivating example 写透
3. 把 `W5` 区分为 vanilla runtime 侧的 backend-class logical false-negative 强证据，与 twin-path 原型侧的公开 workload 恢复证据
4. 把 `TwinPathServe` 写成有明确接口和实例化边界的系统设计
5. 等多 benchmark、完整 baseline matrix 和 output-equivalence 审计补齐后，再升级摘要最后一句和贡献第三条

## 12. 当前包装纪律

既然当前阶段优先级已经切到“论文包装与写作”，后续文稿只应优先打磨下面五个部位：

1. 题目与一句 thesis
2. 摘要五到六句骨架
3. 引言前四段的问题 framing
4. 三条贡献的强弱边界
5. reviewer 最容易抓的三类措辞风险

不应在写作轮次里继续扩下面这些内容：

- 新实验设计文档
- 新实现推进文档
- 新 operator brainstorming
- 新 workload 家族发散

当前最值钱的写作工作，不是“再加材料”，而是把已经存在的材料组织成一个更像 systems paper、而不是 patch collection 的 package。
