# ExactPrefixKV 论文叙事骨架

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档不负责排重，也不负责实验细节。

它只做一件事：

> 把 `ExactPrefixKV` 收束成一个 reviewer 能快速理解、且不容易误判成 patch collection 的论文故事。

因此，这份文档服务于：

- 标题拟定
- 摘要起草
- 引言结构
- 贡献表述
- 图表规划
- 实验叙事顺序

## 2. 一句话 thesis

最推荐的一句话 thesis 是：

> 现有 exact-prefix serving 把 prefix cache 当作隐式优化能力，但在真实 heterogeneous LLM workloads 中，
> 系统缺少从输入稳定性、runtime exactness 到命中后执行兑现的统一契约；
> `ExactPrefixKV` 将其升级为一个 contract-guided exact-prefix serving substrate。

再短一点，可以写成：

> `ExactPrefixKV` makes exact-prefix reuse explicit, typed, and performance-realizable in vLLM.

## 3. 论文最核心的三句话

### 3.1 问题句

> 现有 prefix caching 系统通常只关心“有没有命中”，却没有系统地回答：
> exact prefix 如何被稳定生成、如何被统一判定、以及命中后如何稳定兑现收益。

### 3.2 方法句

> `ExactPrefixKV` 用三层结构解决这一缺口：
> `Cacheability ABI`、`Typed Exact Prefix Contract`、`Post-Hit Execution Selection`。

### 3.3 结论句

> 在 tool-rich、multimodal 和 prompt-embed workloads 上，只有同时解决这三层问题，
> exact-prefix serving 才会从“偶尔命中的 cache 技巧”变成一个稳定可用的系统能力。

## 4. 最推荐的标题模板

标题不应写得过宽，也不应写成 feature list。

### 4.1 推荐标题

1. `ExactPrefixKV: Contract-Guided Exact-Prefix Serving for Heterogeneous vLLM Workloads`
2. `ExactPrefixKV: Making Exact-Prefix Reuse Explicit, Typed, and Performance-Realizable in vLLM`
3. `Beyond Cache Hits: A Contract-Guided Substrate for Exact-Prefix Serving in vLLM`

### 4.2 不推荐标题

1. `A Better Prefix Cache for vLLM`
2. `Typed Prefix Caching for Multimodal Inputs`
3. `Prompt ABI for Prompt Caching`
4. `A Prefix-Aware Scheduler for vLLM`

这些标题要么太像 patch，要么直接撞近邻。

## 5. 摘要骨架

下面是最推荐的摘要骨架，不是最终文本，但结构应尽量保持。

### 5.1 第一段：问题定义

- 先讲 exact-prefix reuse 已成为主流 serving 优化；
- 再指出真实 workload 不再是纯 token prompt；
- 然后指出现有系统缺少统一 contract。

### 5.2 第二段：问题分裂的三种表现

这里建议用三句并列：

1. 应用持续制造 prefix drift；
2. runtime 对异构 carrier 的 exactness 判断碎片化；
3. 命中后系统仍常沿用非最佳执行模式。

### 5.3 第三段：方法总览

明确三层：

1. `Cacheability ABI`
2. `Typed Exact Prefix Contract`
3. `Post-Hit Execution Selection`

不要在摘要里塞太多实现细节。

### 5.4 第四段：结果和意义

结果应强调三种收益，而不是只写 throughput：

1. 更稳定的 hit / 更高的 cache coverage
2. 更低的 TTFT / p99
3. 更强的解释性和系统可诊断性

最后一句强调：

- 这是一种系统 substrate，而不是又一个 isolated optimization。

## 6. 引言结构建议

引言最容易写散，因此建议固定为六段。

### 6.1 第一段：背景

讲 prefix caching 已成为标准能力，但 workloads 越来越 heterogeneous。

### 6.2 第二段：现状缺口

点出三类系统裂缝：

- input instability
- runtime exactness fragmentation
- post-hit under-realization

### 6.3 第三段：失败例子

最好有一个贯穿全文的 motivating example：

- 长 system prompt
- 固定 tools
- 多轮 responses
- 图像或 prompt embeds

说明为什么“看起来共享前缀”，但系统仍 miss 或收益没有兑现。

### 6.4 第四段：论文 insight

明确说：

- 这不是单点 cache 技巧问题；
- 而是 exact-prefix serving 缺失端到端 contract。

### 6.5 第五段：方法总览

用三层结构快速概括系统。

### 6.6 第六段：贡献

贡献建议保持三条，不要散成五六条。

## 7. 最推荐的贡献写法

### 7.1 贡献一：问题重定义

> We identify that exact-prefix serving is limited not merely by cache lookup,
> but by the lack of an end-to-end contract spanning input stability, runtime exactness, and post-hit execution.

### 7.2 贡献二：系统设计

> We design `ExactPrefixKV`, a three-layer substrate that exposes cacheability ABI,
> unifies typed exact-prefix semantics for heterogeneous inputs, and selects execution modes after cache hits.

### 7.3 贡献三：实现与评测

> We implement the design in vLLM and show that the unified contract improves cache stability,
> workload coverage, and latency realization on tool-rich, multimodal, and prompt-embed workloads.

## 8. 论文结构建议

最推荐的正文结构如下。

### 8.1 Section 1: Introduction

- 问题
- 动机例子
- 三层 insight
- 贡献

### 8.2 Section 2: Background and Motivation

- vLLM APC 现状
- exact-prefix serving 现状
- heterogeneous inputs
- motivating traces

### 8.3 Section 3: Why Exact-Prefix Serving Breaks Down

- input drift
- runtime typed fragmentation
- post-hit inefficiency

### 8.4 Section 4: ExactPrefixKV Design

- ABI
- typed contract
- post-hit selector

### 8.5 Section 5: Implementation

- 在 vLLM 哪些模块落地
- 为什么是轻量实现
- 哪些不做

### 8.6 Section 6: Evaluation

- workload family
- P0/P1 对照
- 主结果
- breakdown
- ablation

### 8.7 Section 7: Discussion

- 为什么不是 Prompt Cache / program-serving / distributed scheduling
- 局限性
- 未来工作

## 9. Figure 规划

一篇系统论文如果故事要稳，图的结构要提前定。

### 9.1 Figure 1：Motivating Example

同一应用请求如何在：

- 输入层看起来共享前缀；
- runtime 中却因为 drift / typed mismatch / post-hit path 而损失收益。

### 9.2 Figure 2：Problem Decomposition

三类问题并列：

- unstable prefix generation
- fragmented exactness semantics
- under-realized post-hit execution

### 9.3 Figure 3：ExactPrefixKV Architecture

三层系统总图：

- ABI
- typed contract
- selector

### 9.4 Figure 4：Measurement or Trace Breakdown

展示：

- drift 占比
- typed miss 占比
- post-hit inefficiency 占比

### 9.5 Figure 5：Main End-to-End Results

展示：

- TTFT
- p99
- throughput
- cache coverage

## 10. 实验叙事顺序

实验顺序不要按模块写，而要按问题闭环写。

### 10.1 第一组：为什么现有系统不够

先证明问题存在：

- drift
- typed miss
- post-hit inefficiency

### 10.2 第二组：三层组合的总体收益

展示完整系统带来的：

- hit stability
- TTFT / p99 改善
- coverage 提升

### 10.3 第三组：分层消融

消融顺序建议为：

1. 只有 ABI
2. ABI + typed contract
3. 全部三层

这样能直接体现闭环依赖。

### 10.4 第四组：边界讨论

说明：

- 为什么我们不是 modular prompt reuse
- 为什么我们不是 workflow runtime
- 为什么我们不是 distributed cache-affinity system

## 11. 最容易写崩的几个点

### 11.1 把论文写成 feature list

这是最大风险。

一旦贡献看起来像：

- 支持 prompt embeds
- 支持 multimodal
- 支持 cache_salt
- 再优化一点 residual

那 reviewer 会立刻降格成 patch collection。

### 11.2 把 ABI 写成 prompt engineering

如果 ABI 部分主要在讲：

- prompt 怎么摆
- tools 怎么放
- dynamic content 放哪

那会和工业文档以及 `Don't Break the Cache` 强重合。

### 11.3 把 selector 写成 LAPS 变体

如果 selector 部分主要在讲：

- short prefill batching
- graph clustering
- queue reordering

那就会失去自己的边界。

## 12. 当前最推荐的论文主张

当前最稳的论文主张可以写成：

> `ExactPrefixKV` identifies exact-prefix serving as a missing contract problem rather than a cache-hit problem,
> and shows that making exact-prefix reuse explicit, typed, and performance-realizable leads to more stable and broadly useful reuse in vLLM.

中文可以对应为：

> `ExactPrefixKV` 认为 exact-prefix serving 的核心缺口不在于“再提高一点 cache hit”，
> 而在于现有系统缺少一套从输入稳定性到执行兑现的统一契约；
> 只有把 reuse 变成显式、typed、可兑现的系统能力，prefix cache 才能在真实 workload 中稳定成立。

## 13. 与其他文档的关系

这份文档和同目录其他文档的关系如下：

1. [README.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/README.md)
   - 定义 thesis 本身。
2. [deep_novelty_assessment_20260415.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/deep_novelty_assessment_20260415.md)
   - 负责排重和实现可行性。
3. [publishability_audit.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/publishability_audit.md)
   - 负责 claim 与 stop 条件。
4. [measurement_plan.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/measurement_plan.md)
   - 负责用数据决定这条论文线是否值得继续实现。

## 14. 当前建议

这份文档服务的不是“现在就写正文”，而是：

- 在后续做 P0 instrumentation 时，始终用论文叙事来约束 measurement；
- 避免后面数据很多，但故事越来越散。
