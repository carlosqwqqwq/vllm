# FalseNegativeKV Paper Storyline

更新日期：2026-04-16

## 1. 这份文档的作用

这份文档只负责把 `FalseNegativeKV` 收束成一个 reviewer 能快速理解的论文故事。

它不负责：

- 再次排重
- 再次发散新 idea
- 再次扩展 P1 机制范围

从现在开始，`FalseNegativeKV` 的 paper-ready 包装只围绕下面这条 thesis 展开，不允许继续抽象漂移。

补充说明：

- 这份文档锁定的是目标 `full-thesis` 版本的 paper storyline
- 当前实际写作应优先遵守 `paper/` 子目录中的 measurement-first 口径
- 当前可强写的主证据是 `W1 prompt_logprobs`
- 当前 `W5` 在 vanilla runtime 侧仍只能稳妥写成 cross-backend logical false-negative evidence，但在 `TwinPathServe` 原型侧已经有公开 workload 恢复证据；两者不能混写，更不能提前写成在线原生 `backend_mismatch` 分类已经完成
- 当前正式方法层以
  [reuse_contract_spec.md](../../implementation/reuse_contract_spec.md)
  为准；paper storyline 必须与这份方法规格保持一致

## 2. 一句 thesis

固定 thesis 句为：

> `FalseNegativeKV` studies **contract-aware recovery of false-negative exact hits**:
> many requests in modern LLM serving are logical hits but not physical hits, because existing runtimes lack a reusable contract between producer state and consumer requirements.

写成更适合引言里的中文表达就是：

> 现有 exact-prefix serving 系统只显式优化 `physical hit`，却没有系统地建模那些“逻辑上应命中、物理上却未命中”的 `false-negative hit`；
> `FalseNegativeKV` 将其提升为一个可测、可解释、可恢复的 runtime 问题，并用 `TwinPathServe` 给出第一版系统化恢复方案。

这句 thesis 在 paper 中还必须补上一条判定边界：

> 本文讨论的不是所有 prefix miss，而是那些已经有参考执行条件证明“可复用”，却没有在当前请求中兑现为收益的 miss。

## 3. 标题风格锁定

标题风格固定为：

- `问题名 + 方法主张`
- 或 `系统名 + 问题主张`

不允许使用下面这种写法：

- `A Better Prefix Cache for vLLM`
- `Supporting Prompt Logprobs in Prefix Cache`
- `A Patch for Mixed vLLM APIs`

### 3.1 主标题

最推荐主标题固定为：

- `FalseNegativeKV: Contract-Aware Recovery of False-Negative Exact Hits in vLLM`

### 3.2 备选标题

只保留一个系统导向备选：

- `TwinPathServe: Contract-Aware Recovery of False-Negative Exact Hits for vLLM Serving`

## 4. 摘要骨架锁定

摘要固定为五句结构，不允许再拆成 feature list。

### 4.1 第一句：背景

先讲：

- exact-prefix reuse 已经是主流 LLM serving runtime 的核心优化
- 但 workload 已经从纯文本 prompt 走向 tool-rich、multimodal、mixed API

### 4.2 第二句：问题

明确指出：

- 许多请求在逻辑上是 `logical hit`
- 却没有变成 `physical hit`
- 原因不是 state 不存在，而是 `reuse contract` 不匹配

### 4.3 第三句：问题表征

必须出现下面四个概念中的至少三个：

- `logical hit`
- `physical hit`
- `false-negative hit`
- `bridgeable miss`

并说明我们提供 measurement surface，而不是只给 anecdotal bugs。

### 4.4 第四句：方法

这里必须同时出现：

- `reuse contract`
- `realization policy`
- `bridge operator`
- `TwinPathServe`

并明确当前写作主线是一个四层链条：

1. measurement surface
2. `reuse contract`
3. `realization policy`
4. `TwinPathServe`

并明确第一版只有两个实例化：

- `prompt_logprobs` selective replay
- batch-invariant fast-path twin-pool route

### 4.5 第五句：结果与意义

结果句固定强调三件事：

1. false-negative miss 在真实 workload 中足够普遍
2. 至少两类 mismatch 可被同一框架恢复
3. 恢复后能带来显著 `TTFT` 改善，且 throughput 回退受控

最后一句意义必须落在：

- exact reuse 应被视为 contract-aware runtime capability
- 而不是 scattered compatibility patches

## 5. 引言骨架锁定

引言固定写成六段，不再自由发挥。

### 5.1 第一段：背景

讲 prefix caching / exact-prefix reuse 已成为 serving 基础能力。

### 5.2 第二段：缺口

讲现有系统默认只看 `physical hit`，缺少 `logical hit -> physical hit` 的 contract 视角。

### 5.3 第三段：motivation example

必须给一个统一例子，推荐固定为：

- 长 system prompt
- 多轮 chat / tool-rich 请求
- 同一 family 上既有普通 generation，又有 `prompt_logprobs`
- 后端 fast-path 与 cache-friendly path 不一致

### 5.4 第四段：insight

明确指出：

- 这不是几个 feature patch
- 而是一类 `false-negative exact hit` runtime pathology

### 5.5 第五段：方法总览

必须按下面顺序出现：

1. measurement
2. `reuse contract`
3. `realization policy`
4. `TwinPathServe`

其中：

- measurement 负责把 `logical hit -> physical hit` 的落差量化出来
- `reuse contract` 负责解释为什么“状态存在”不等于“可直接消费”
- `realization policy` 负责解释为什么“理论上可恢复”也不等于“当前值得恢复”
- `TwinPathServe` 负责在 runtime 中兑现 bridge 与 route

### 5.6 第六段：贡献

贡献只允许写三条，不得展开成五条以上。

## 6. 三条贡献锁定

下面三条贡献从现在开始视为固定版本。

### 6.1 贡献一：问题表征

> We identify false-negative exact hits as a previously unstructured runtime pathology in LLM serving, and formalize `logical hit`, `physical hit`, `false-negative hit`, and `bridgeable miss` with a measurement-first methodology.

### 6.2 贡献二：统一 substrate

> We design a four-layer contract-aware reuse substrate spanning measurement, `reuse contract`, `realization policy`, and runtime realization, centered on `false_negative_reason`, `bridgeability`, and `bridge operator`, showing why a unified framework is more explanatory than per-feature fixes.

### 6.3 贡献三：双实例系统与评测

> We implement `TwinPathServe` in vLLM with two bridge operators, `prompt_logprobs` selective replay and batch-invariant twin-pool routing, and present first prototype evidence that both observation mismatch and backend mismatch can be recovered within a `soundness-first` boundary.

## 6.4 贡献抬升的固定讲法

如果 reviewer 质疑“贡献是否太小”，正文与 rebuttal 都必须把贡献写成下面这条三层链，而不是写成两个 patch。

1. 第一层是问题贡献：
   我们把“已计算但未兑现”的复用损失从普通 miss 中分离出来，给出一套可判定、可测量的问题边界。
2. 第二层是抽象贡献：
   我们说明 hash 只能解决前缀身份定位，不能保证当前请求合法消费既有状态；因此系统需要显式的 `reuse contract` 来桥接“状态存在”与“收益兑现”，并给出 `producer state -> consumer requirement -> realization decision` 的统一语义链。
3. 第三层是系统贡献：
   我们用 `TwinPathServe` 证明 observation mismatch 与 backend mismatch 不是彼此无关的 feature gap，而能被同一 runtime substrate 容纳与恢复。

如果这三层在 paper 中没有同时出现，reviewer 就会自然把论文降级理解为 patch collection。

## 6.5 现象发生条件必须写死

从现在开始，paper 的前半部分必须明确写出前缀复用假阴性的成立条件，而不是只给直觉描述。

固定条件为：

1. 存在参考执行条件，已证明某个 prefix family 可以稳定复用。
2. 当前请求与参考请求共享相同的可复用前缀。
3. 当前请求变化的是消费条件，而不是前缀本身是否存在。
4. 当前请求的实际命中显著低于参考请求，因此既有计算没有兑现为当前收益。

固定非条件为：

- 前缀此前从未被计算过
- family 对齐失败
- 上下文被截断或 prompt 已实质变化
- 普通哈希 miss

不把这组条件写清楚，reviewer 会默认把你们的问题理解成“cache key 没打通”。

## 6.6 hash 质疑的固定回答

当 reviewer 提出“KV cache 用 hash，为什么会 already exists but not reused”时，paper 中必须坚持下面这条回答口径：

> hash 解决的是前缀身份定位，而不是当前请求在其返回语义、执行路径和正确性约束下是否合法消费这份状态。哈希命中是复用成立的必要条件，而不是充分条件。

进一步的固定扩展句为：

> 本文所说的“已经存在”并不是指当前请求在当前缓存表中必然已有一个可直接读取的物理 entry，而是指语义等价的前缀计算已经在参考执行条件下被证明可复用，但当前请求没有把这部分既有计算兑现为自己的命中收益。

## 7. 五张主图锁定

主图数量固定为五张，不再加到七八张大图。

## 7.1 Figure 1：Motivating Example

固定内容：

- 同一 `prefix family` 上的 generation / `prompt_logprobs` / backend mismatch 请求
- 为什么它们是 `logical hit`
- 为什么当前系统却只得到低 `physical hit`

固定结论：

- current runtimes lose reuse before computation starts

## 7.2 Figure 2：Problem Decomposition

固定内容：

- `logical hit`
- `physical hit`
- `false-negative hit`
- `bridgeable miss`
- `reuse contract` 与 mismatch 的关系

固定结论：

- false-negative miss 不是单点 bug，而是统一问题类

## 7.3 Figure 3：Instrumentation Breakdown

固定内容：

- 五类 workload 的 prevalence
- `false_negative_reason` breakdown
- `bridgeable_tokens` 占比

固定结论：

- 问题足够普遍，并且不只出现在单一 API

## 7.4 Figure 4：`TwinPathServe` Architecture

固定内容：

- `reuse contract` layer
- `realization policy` layer
- `reuse_optimized_pool`
- `throughput_optimized_pool`
- route key
- 两个 `bridge operator`
- fallback path

固定结论：

- 这是一套 runtime design，而不是若干 feature patch

## 7.5 Figure 5：End-to-End Results

固定内容：

- `TTFT`
- `p95/p99 TTFT`
- throughput
- ad hoc per-feature fix 对比

固定结论：

- unified substrate + twin-path runtime 能恢复真实收益

## 8. reviewer attack -> rebuttal 锁定

下面这张表从现在开始视为标准 rebuttal 模板。

| reviewer attack | 我们必须给出的 rebuttal | 绝不能写的弱回应 |
| --- | --- | --- |
| `patch collection` | 论文核心不是两个 patch，而是 `logical hit / physical hit / false-negative hit / bridgeable miss` 的统一问题表征，以及共享的 `reuse contract` / `bridge operator` substrate | “我们修了两个很有用的 bug” |
| `hash already solves this` | hash 只解决 identity，不解决 consumption feasibility；本文研究的是“已证明可复用的既有计算为何没有兑现为当前请求收益” | “虽然有 hash，但实现比较复杂” |
| `vLLM`-specific | `vLLM` 只是实例系统；只要 runtime 同时存在 shared-state reuse、异构 API、异构 backend，就可能出现 false-negative exact hit | “因为 vLLM 很流行，所以这些 bug 值得写论文” |
| `why not fix separately` | scattered fixes 无法统一解释、统一测量、统一路由，也不能说明为什么 logical hit 会系统性丢失；`FalseNegativeKV` 的贡献是给出统一 contract | “统一做更优雅” |
| `niche workload only` | 我们必须拿出五类 workload 的 prevalence，并至少证明两类 mismatch 在不同 family 上成立；不能只靠一个 `prompt_logprobs` microbenchmark | “这个场景在我们内部很重要” |
| `too little contribution` | 贡献必须是三层包：问题表征、统一 substrate、双实例系统与闭环评测；少一层都不够 | “实现工作量其实很大” |

## 9. 论文结构锁定

正文结构固定为七段式：

1. Introduction
2. Background and Motivation
3. Measuring False-Negative Exact Hits
4. Contract-Aware Reuse Substrate
5. `TwinPathServe` Design and Implementation
6. Evaluation
7. Discussion and Limits

## 10. 讨论与不该声称的点

下面这些话从现在开始明确不能写：

- “我们首次提出 prompt caching”
- “我们首次支持 multimodal prefix cache”
- “我们解决了所有 prefix cache miss”
- “我们的系统一般优于所有 serving runtime”

可以写的、更稳的表达是：

- 我们识别并形式化了 `false-negative exact hit`
- 我们证明统一 substrate 比 scattered fixes 更能解释并恢复这类 miss
- 我们在 `vLLM` 中给出了第一版双实例系统

## 11. 这篇论文成立的最低条件

如果最后不能同时满足下面三条，这篇论文就不该按主 thesis 投稿：

1. prevalence 结果足够强，能支撑 `H1`
2. 两类 mismatch 都能被同一框架恢复，能支撑 `H3`
3. 至少一个强实例化达到 `TTFT >= 20%` 改善，能支撑 `H4`

也就是说，`paper_storyline_fnkv.md` 不是包装文档，它只是把前面 measurement 和 architecture 已经证明的东西锁成 reviewer 能读懂的故事。
