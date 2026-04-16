# ExactPrefixKV 可发表性审计

更新日期：2026-04-15

## 1. 这份文档是干什么的

这不是一份技术设计文档，而是一份**论文可发表性审计**。

它回答四个问题：

1. 这条线现在还能不能继续做；
2. 如果继续，论文到底能 claim 什么；
3. 哪些 claim 一写出来就会和现有工作高重合；
4. 接下来该用什么证据判断 go / no-go。

这份文档默认讨论的是当前组合主线：

- `PromptABIKV`
- `TypedPrefixKV`
- `ResidualModeKV`

统一后的 thesis：

- `ExactPrefixKV`

## 2. 执行判断

当前最准确的判断不是：

- “这条线已经很新”

而是：

- **这条线仍可继续，但必须以组合版系统 thesis 的形式推进。**

更具体地说：

1. 三条单线分别推进，顶会风险偏高；
2. 组合成一个 `contract-guided exact-prefix serving` 的系统，还有可防守空间；
3. 但必须承认：
   - 截至 `2026-04-15`，我们只是**没有找到完全同构公开论文**；
   - 不能写成“确保没人做过”；
   - 也不能写成“首次提出 prompt caching / typed prefix / prefix-aware scheduling”。

## 3. 论文想活下来，核心 claim 必须是什么

### 3.1 当前最稳的一句话 thesis

> `ExactPrefixKV` 将 vLLM 的 exact-prefix reuse 从隐式缓存特性提升为一个
> 面向异构输入、具备显式契约、统一精确语义与命中后执行选择的系统底座。

英文版最稳的写法是：

> `ExactPrefixKV` is a contract-guided exact-prefix serving substrate for
> heterogeneous vLLM workloads.

### 3.2 审稿人最终应该记住什么

如果论文成功，审稿人记住的不能是：

- “又一个 prefix cache 优化”
- “支持了更多 carrier”
- “又加了一个 scheduler heuristic”

而应该是：

> 现有 exact-prefix serving 缺少从输入稳定性、runtime exactness 到命中后执行兑现的统一契约，
> 因而在 tool-rich / multimodal / embed-rich workload 中命中不稳定、语义不统一、收益不兑现；
> `ExactPrefixKV` 把这三段闭环系统化了。

只有这个层级，才像系统论文，而不是 patch 拼盘。

## 4. Claim 分级

下面把当前所有可能出现的 claim 分成三类。

### 4.1 绿色 claim：可以作为主张

这些是目前仍然可防守的。

1. **我们研究的是 exact-prefix serving 的系统闭环，而不是单点缓存技巧。**
2. **我们把应用输入稳定性、runtime typed exactness、命中后 execution selection 统一到同一个系统里。**
3. **我们面向的是 heterogeneous vLLM inputs，而不是纯文本 token prompt。**
4. **我们的重点是 contract / observability / execution realization，而不是更灵活的 approximate reuse。**
5. **我们关注的是 colocated / single-engine vLLM 中 exact-prefix serving 的系统语义，不是 workflow OS，也不是 distributed routing。**

### 4.2 黄色 claim：可以写，但必须非常克制

这些可以写，但需要限定语境，否则容易被最近邻工作击穿。

1. **typed exact-prefix semantics**
   - 可以写，但要明确：
   - 不是声称第一次支持 multimodal / prompt embeds；
   - 而是第一次把它们收束成统一 contract。
2. **cacheability ABI**
   - 可以写，但要明确：
   - 不是 prompt engineering 指南；
   - 而是 runtime-observable、可归因、可 enforced 的接口层。
3. **post-hit execution selection**
   - 可以写，但要明确：
   - 不是一般性的 short-prefill batching；
   - 而是 exact hit 后 residual execution 的再选择。

### 4.3 红色 claim：不要写

这些表述一旦写出来，就会直接增加拒稿概率。

1. “首次提出 prompt caching”
2. “首次研究 exact prefix match”
3. “首次支持 multimodal prefix cache”
4. “首次支持 prompt embeds APC”
5. “首次提出 prefix-aware scheduling”
6. “首次把 tool-aware caching 带入 serving”
7. “确保无人做过”
8. “全面领先现有工作”

这些都不符合当前证据。

## 5. 最近邻防守矩阵

这一节的目标不是证明“我们和谁都不一样”，而是找出**最小可防守差异**。

### 5.1 Prompt Cache

- 近邻工作：`Prompt Cache`
- 重合点：
  - prompt reuse
  - structured / modular input reuse
- 我们不能说的：
  - 我们比它更会复用 prompt
- 我们能说的：
  - 我们坚持 **exact reuse**
  - 我们研究的是 vLLM runtime 内的 exact-prefix contract
  - 我们关注异构 carrier 的统一 exactness 与命中后执行兑现

### 5.2 OpenAI / Anthropic / Google prompt caching

- 近邻来源：
  - OpenAI Prompt Caching
  - Anthropic Prompt Caching
  - Google Gemini Context Caching
- 重合点：
  - exact prefix
  - 工具、图像、schema 会影响缓存
  - 输入组织影响命中
- 我们不能说的：
  - 我们首次发现输入稳定性重要
- 我们能说的：
  - provider 文档说明了现象；
  - 我们把这种现象变成 vLLM 内部的显式 contract、typed semantics 和 post-hit policy。

### 5.3 Don’t Break the Cache

- 近邻工作：`Don't Break the Cache`
- 重合点：
  - agentic workload 下缓存很脆弱
  - tool / schema / thinking / block layout 会打碎 cache
- 我们不能说的：
  - 我们首次发现 agent workload 会破坏缓存
- 我们能说的：
  - 该工作主要是分析与实践层；
  - 我们做的是 runtime 内部的接口语义和执行层设计。

### 5.4 Serve Programs, Not Prompts / Pie / ThunderAgent

- 近邻工作：
  - `Serve Programs, Not Prompts`
  - `Pie`
  - `ThunderAgent`
- 重合点：
  - 应用层结构进入 serving
  - KV 管理变成显式对象
  - workflow / inferlet / program 感知
- 我们不能说的：
  - 我们是更轻量的 program-serving runtime
- 我们能说的：
  - 我们明确**不做** program-aware runtime；
  - 我们只收缩在 exact-prefix serving 的 contract 层。

### 5.5 Preble / DualMap

- 近邻工作：
  - `Preble`
  - `DualMap`
- 重合点：
  - prefix-aware scheduling
  - cache affinity
- 我们不能说的：
  - 我们是新的 prefix-aware scheduler
- 我们能说的：
  - 我们主要研究单机 / colocated exact-prefix semantics；
  - 调度只是命中后执行模式选择，而不是分布式路由。

### 5.6 LAPS / InferCept

- 近邻工作：
  - `LAPS`
  - `InferCept`
- 重合点：
  - short re-prefill
  - graph / residual handling
  - pause-resume / recompute
- 我们不能说的：
  - 我们解决了 short residual batching
- 我们能说的：
  - 我们的切口是 exact hit 之后的 residual mode selection；
  - 决策对象是 reuse 本身，而不是一般性短序列分桶。

## 6. 这条线必须回答的四个 reviewer 问题

如果这四个问题答不出来，继续做的意义不大。

### 6.1 你们和最近邻工作的“一句话 delta”是什么？

现在最好的回答是：

> 现有工作分别研究了 prompt 组织、typed carrier 支持或 prefix-aware scheduling，
> 但没有把它们收束成一个面向 heterogeneous vLLM inputs 的 exact-prefix contract，
> 并把命中后的执行兑现纳入同一系统。

### 6.2 你们有没有最近邻工作做不了的 killer experiment？

这条线至少需要一个下面这种级别的实验：

- 在 tool-rich / multimodal / prompt-embed workloads 中，
- 证明 baseline vLLM 的 miss 主要不是模型算子问题，而是：
  - 输入稳定性差
  - typed exactness 分裂
  - post-hit residual 执行模式不佳
- 并证明组合系统能同时修复这三类问题中的至少两类。

### 6.3 你们是闭环系统，还是三个 patch 拼接？

如果实验结果最后呈现成这样：

- ABI 带来一点 hit 率提升
- typed hash 修掉一些特例
- selector 再优化一点 TTFT

但三者之间没有强耦合因果链，那这篇论文会显得很散。

因此，必须证明：

1. 没有 ABI，就没有稳定 exact prefix；
2. 没有 typed contract，就没有可靠的 runtime 判断；
3. 没有 post-hit selection，就没有稳定的收益兑现。

### 6.4 这个系统是不是只是工程 hygiene？

这是最危险的问题。

如果 reviewer 认为这只是：

- 一套更规范的输入组织；
- 一套更完整的 hash；
- 一套更细的 scheduling heuristic；

那高度就不够。

因此必须用实验说明：

- 这些不是 hygiene，而是 exact-prefix serving 的**缺失系统语义**；
- 没有它们，系统会持续在 correctness、hit stability 和 latency realization 上付出代价。

## 7. 现在最需要的数据，不是更多想法

继续推进前，必须先用 measurement 判断下面三件事是否真实存在。

### 7.1 Prefix drift 是否足够严重

我们需要知道：

- 在真实 tool-rich / multi-turn / multimodal / prompt-embed workload 中，
- 有多少 miss 是因为“本来应当共享 exact prefix，但被输入组织打碎了”。

如果这个比例很低，`Cacheability ABI` 就撑不起一层。

### 7.2 Typed carrier 分裂是否足够严重

我们需要知道：

- 现有 vLLM 的 carrier 语义是否已经足够统一；
- 如果已经统一得差不多，那 `TypedPrefixKV` 就不再是论文核心，只能做背景。

### 7.3 Post-hit inefficiency 是否足够严重

我们需要知道：

- 在 APC 已高命中的情况下，
- residual shape 是否仍频繁导致：
  - graph fallback
  - padding waste
  - cascade 非最佳使用

如果这一点很弱，`ResidualModeKV` 只能退化为附属优化。

## 8. 可以发表的条件

继续往 OSDI/CCF-A 级别推进，至少要满足下面四条中的三条。

1. **有清晰的一句话 delta**
   - reviewer 一眼能分清你不是 Prompt Cache / LAPS / ThunderAgent 的变体。
2. **有强因果闭环**
   - 三层不是独立 patch，而是顺序依赖。
3. **有两个以上 workload family 的稳定收益**
   - 不能只在一个 niche synthetic benchmark 上好看。
4. **有一个最近邻工作不自然覆盖的 killer experiment**
   - 否则论文边界会非常弱。

## 9. 应当停止的条件

只要下面任意两条成立，就应考虑 pivot，而不是继续硬做。

1. 我们写不出一句不依赖长解释的 delta claim。
2. 我们发现主要收益其实只是更规范地组织 prompt。
3. 我们的 typed contract 贡献最后只剩“把已有 extra hash 收一收”。
4. 我们的 execution selection 收益在非合成 workload 上很小。
5. 我们最好的实验结果也很容易被解释成现有工作的小延伸。

## 10. 当前推荐的论文定位

### 10.1 建议采用的定位

推荐定位：

- `contract-guided exact-prefix serving`
- `heterogeneous exact-prefix semantics in vLLM`
- `realizing exact-prefix reuse beyond cache hits`

### 10.2 不建议采用的定位

不建议定位成：

- 新 prompt caching 算法
- 新 prompt engineering 方案
- 新 multimodal cache 机制
- 新 distributed prefix scheduler
- 新 agent runtime

这些方向的近邻都更成熟，正面对打不划算。

## 11. 接下来的正确动作

从可发表性角度，接下来最合理的不是继续想名字，而是：

1. 做 measurement plan
   - 先看这条线有没有足够强的数据支撑。
2. 做 P0 instrumentation
   - 只加观测，不加策略。
3. 用数据决定是否进入 P1
   - 如果数据不强，尽早止损；
   - 如果数据强，再做最小闭环原型。

## 12. 最终判断

这条线现在的状态可以概括为：

- **还没死，但不能再靠“idea 本身听起来很新”往前走。**

它能不能发，不取决于“和现有 idea 像不像”，而取决于：

- 我们是否能把它变成一个 reviewer 无法轻易降格成 patch collection 的系统 thesis。

当前我的建议是：

- **继续，但只做审计和 measurement，不直接重实现。**

这不是保守，而是当前最学术、也最省成本的推进方式。

## 13. 推荐题目与摘要起草骨架

### 13.1 推荐题目模板

当前最推荐的标题方向有三类：

1. `ExactPrefixKV: Contract-Guided Exact-Prefix Serving for Heterogeneous vLLM Workloads`
2. `ExactPrefixKV: Making Exact-Prefix Reuse Explicit, Typed, and Performance-Realizable in vLLM`
3. `Beyond Cache Hits: A Contract-Guided Substrate for Exact-Prefix Serving in vLLM`

这三类标题的共同特点是：

- 不把贡献写成单点 feature；
- 不把故事写成一般性的 scheduler；
- 直接突出 contract、typed、post-hit realization 这三个关键词。

### 13.2 摘要起草顺序

摘要建议固定成四段逻辑：

1. 背景与问题
   - exact-prefix reuse 普及，但 workload 越来越 heterogeneous。
2. 现有系统的三类缺口
   - 输入不稳定；
   - runtime exactness 分裂；
   - 命中后收益不稳定。
3. 我们的方法
   - ABI、typed contract、post-hit selector。
4. 我们的结果与意义
   - hit stability、coverage、TTFT/p99、系统解释性。

### 13.3 摘要里不建议出现的词

为了避免过早撞近邻，摘要里不建议高频出现：

- first
- novel prompt caching
- general scheduler
- workflow-aware serving
- agent runtime

更安全的表述是：

- contract-guided
- exact-prefix serving
- heterogeneous inputs
- performance realization after hits

## 14. 贡献检查清单

在真正开始写论文前，建议用下面清单自查。

### 14.1 问题定义层

- 是否能用一句话定义系统缺口，而不是 feature 缺口？
- 是否能解释为什么现有工作没有解决同一个闭环？
- 是否明确区分了 exact reuse 与 modular / workflow / approximate reuse？

### 14.2 设计层

- 三层是否真的是顺序依赖，而不是并列 patch？
- 如果拿掉任意一层，故事是否明显不成立？
- 设计是否仍然保持轻量级和 vLLM 可实现性？

### 14.3 评测层

- 是否至少覆盖两个以上 workload family？
- 是否至少有一个 killer experiment 最近邻工作不能自然覆盖？
- 是否同时报告性能和解释性指标？

### 14.4 写作层

- 标题是否仍然是“系统 substrate”，而不是“某个新技巧”？
- 摘要是否避免了过强 novelty claim？
- 引言是否先讲问题闭环，再讲三层方法？

## 15. 论文推进顺序

如果文档和 measurement 结果都支持继续，推荐按下面顺序推进正文写作。

1. 先写 problem statement
   - 不写方案细节，先把系统缺口写稳。
2. 再写 motivating example
   - 选择一个能同时包含 tools、typed carrier 和 residual path 的例子。
3. 再写方法总图
   - 三层结构一张图收束。
4. 再写评测提纲
   - 先确定实验顺序，再决定具体实现展示粒度。
5. 最后再写实现细节
   - 避免实现细节反客为主。

## 16. 参考资料

- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Preble: <https://arxiv.org/abs/2407.00023>
- InferCept: <https://arxiv.org/abs/2402.01869>
- Serve Programs, Not Prompts: <https://yecl.org/publications/gim2025hotos.pdf>
- Don’t Break the Cache: <https://arxiv.org/abs/2601.06007>
- LAPS: <https://arxiv.org/abs/2601.11589>
- ThunderAgent: <https://arxiv.org/abs/2602.13692>
- DualMap: <https://arxiv.org/abs/2602.06502>
- OpenAI Prompt Caching: <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- Google Gemini Context Caching: <https://ai.google.dev/gemini-api/docs/caching/>
