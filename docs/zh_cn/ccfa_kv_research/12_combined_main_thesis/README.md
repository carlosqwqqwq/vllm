# ExactPrefixKV：组合三条线后的主 thesis

更新日期：2026-04-15

## 1. 执行结论

在连续多轮排重之后，单独推进 `PublicStateKV`、`ZeroLagKV`、`FastPathKV` 都已经不够稳。当前更值得继续的主线，不是再发明一个新的宽泛名词，而是把已有三条仍然存活的线：

- `PromptABIKV`
- `TypedPrefixKV`
- `ResidualModeKV`

组合成一个更完整、也更像系统顶会的 thesis：

> **ExactPrefixKV：把 exact-prefix serving 从“隐式缓存特性”提升为一个三层系统。**
>
> 第一层保证应用持续产生稳定、可命中的前缀；第二层在 runtime 内统一定义异构输入上的 exact prefix；第三层在命中后根据 residual work 与执行模式的真实代价做联合选择。

我当前的判断是：

1. 这个组合版故事比三条单线各自为战更强。
2. 截至 `2026-04-15`，我**没有找到与这个组合版 thesis 完全同构的公开系统论文**。
3. 但我**不能保证没人做过**，也不能把 novelty 写成“首次提出 prompt caching / typed prefix / residual scheduling”。
4. 它最可能成立的方式，是把贡献写成：
   - **contract-guided exact-prefix serving**
   - 而不是单点 cache 技巧。

### 1.1 文档导航

当前这一组文档的分工如下：

1. [README.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/README.md)
   - 主 thesis 定义、三层结构、与近邻工作的总体边界。
2. [deep_novelty_assessment_20260415.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/deep_novelty_assessment_20260415.md)
   - 更严格的排重、实现抓手和风险判断。
3. [publishability_audit.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/publishability_audit.md)
   - 从“是否还能发表”的角度审计 claim、边界和 stop 条件。
4. [measurement_plan.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/measurement_plan.md)
   - P0 measurement 方案、workload、指标、插桩点、kill criteria。
5. [paper_storyline.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/paper_storyline.md)
   - 论文叙事骨架、标题模板、摘要骨架和 figure 规划。
6. [p0_instrumentation_spec.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/p0_instrumentation_spec.md)
   - 第一批最小代码改动规格：新增字段、JSONL 格式、配置开关和文件级最小 diff。

### 1.2 推荐阅读顺序

如果是第一次进入这条主线，推荐按下面顺序阅读：

1. 先读本文
   - 搞清楚为什么是三条线组合，而不是单线继续发散。
2. 再读 `deep_novelty_assessment_20260415.md`
   - 看清楚和最近邻工作的真实重合边界。
3. 再读 `publishability_audit.md`
   - 看清楚能说什么、不能说什么。
4. 再读 `measurement_plan.md`
   - 看清楚下一步不是“想法扩写”，而是 P0 instrumentation。
5. 再读 `p0_instrumentation_spec.md`
   - 看清楚第一批到底改哪些字段和日志。
6. 最后读 `paper_storyline.md`
   - 用论文视角看如何把这条线收束成可投稿故事。

### 1.3 当前阶段判断

当前阶段还不是“开始写论文”或“开始做完整系统”，而是：

- **主 thesis 已收束完成；**
- **排重和可发表性审计已基本完成；**
- **下一步应进入 P0 instrumentation，而不是继续发散 idea。**

## 2. 为什么三条单线不够

三条单线各自都有价值，但单独作为主线都存在明显短板。

### 2.1 `PromptABIKV` 单独做，容易被降格成 prompt engineering

它解决的是：

- 应用层如何稳定地产生可命中的 exact prefix。

但如果单独成文，审稿人很容易把它理解成：

- best practice 总结
- prompt organization guideline
- provider prompt caching 经验帖

这会削弱论文高度。

### 2.2 `TypedPrefixKV` 单独做，容易被降格成 feature patch 集合

它解决的是：

- 异构输入下，什么才算同一个 exact prefix。

但如果单独成文，最容易滑向：

- 支持 `prompt_embeds` 的 APC
- 支持 multimodal prefix caching
- 把 `cache_salt` 放进 hash

这些点都真实，但单独看像 feature patch，不像完整系统。

### 2.3 `ResidualModeKV` 单独做，容易被压到 `LAPS` 一类近邻下面

它解决的是：

- exact hit 之后的 residual work 是否应该进入不同执行模式。

但如果单独成文，很容易和：

- short-prefill batching
- graph clustering
- prefix-aware scheduling

混在一起，导致边界不够干净。

因此，三条线更自然的关系不是三选一，而是三层协同：

1. `PromptABIKV`
   - 负责稳定输入。
2. `TypedPrefixKV`
   - 负责定义和验证 exact prefix。
3. `ResidualModeKV`
   - 负责命中之后的执行模式选择。

## 3. 组合后的统一 thesis

组合后的主张可以写成：

> 现有 LLM serving 把 prefix cache 当成一个“尽量命中”的附属能力，但在真实 chat、tool、multimodal 与 prompt-embed workload 中，系统缺少一套端到端的 exact-prefix contract。
>
> 结果是：应用层持续制造 prefix drift，runtime 内部对异构输入的 exactness 判断碎片化，而命中之后系统又默认沿用固定执行路径，无法兑现全部收益。
>
> `ExactPrefixKV` 通过一个三层系统，把 exact-prefix serving 变成：
>
> - 可稳定生成
> - 可统一判定
> - 可按代价执行

更具体地说，它解决三个连续问题：

1. **同一个 exact prefix 能否被稳定地产生出来？**
2. **runtime 能否在异构输入上对它做统一、精确、可观测的判断？**
3. **一旦命中，系统是否还能根据 residual work 重新选择更快的执行模式？**

只有三个问题一起解决，prefix cache 才会从“命中了就省一点 prefill”变成一个真正可泛化的 subsystem。

## 4. 三层结构

## 4.1 第一层：Cacheability ABI

这一层继承 `PromptABIKV`，但不把故事写成 prompt engineering。

它的任务是：

- 把会影响 exact-prefix 命中的输入组织方式变成显式接口；
- 让系统能诊断是谁打碎了 cache；
- 让 renderer / template / tool schema / multimodal wrapper 遵守可验证的 ABI。

最小输出包括：

- `stable prefix`
- `dynamic suffix`
- `cache scope`
- `breaker reason`

它回答的问题是：

> 上层应用到底有没有在持续生成“同一个 exact prefix”？

## 4.2 第二层：Typed Exact Prefix Contract

这一层继承 `TypedPrefixKV`，是组合 thesis 的中心。

它的任务是：

- 定义异构输入上的 exact prefix；
- 为不同 carrier 做 canonicalization；
- 统一 hash / invalidation / publication / observability 规则。

最小 carrier 建议只保留三类：

1. token span
2. prompt-embeds fingerprint
3. multimodal hash + placeholder layout + cache scope

这层回答的问题是：

> 如果输入不是纯文本 token，runtime 到底如何一致地判断“这是同一个 exact prefix”？

## 4.3 第三层：Post-Hit Execution Mode Selection

这一层继承 `ResidualModeKV`，负责把命中收益真正兑现成性能。

它的任务是：

- 在命中之后看 residual work，而不是只看原始 prompt 长度；
- 联合考虑：
  - bounded recompute
  - graph bucket 对齐
  - `FULL / PIECEWISE / NONE` cudagraph mode
  - cascade-attention eligibility
  - padding waste

它回答的问题是：

> 命中了 prefix cache，并不自动意味着“按最短 residual 继续跑”就是最快；那系统应该怎样重新做 mode selection？

## 5. 为什么这个组合更像顶会论文

它比单线更像顶会，有四个原因。

### 5.1 它是一个完整系统闭环

不是只研究：

- 命中前
- 或命中时
- 或命中后

而是把三段因果链接起来：

- 生成可命中前缀
- 判定可命中前缀
- 兑现可命中前缀收益

### 5.2 它不是单点 patch

它不会退化成：

- 再加一个 hash key
- 再加一个 prompt guideline
- 再加一个 scheduler heuristic

而是一个统一 contract。

### 5.3 它同时满足“轻量级”和“系统性”

它不要求：

- 新模型
- 新 attention 数学
- 新分布式架构
- 新 agent OS

但又比普通工程 patch 更高一层，因为它重构的是：

- exact-prefix serving 的接口和语义。

### 5.4 它更贴近 vLLM 的真实演进方向

`vLLM` 当前已经同时出现了：

- `prompt_embeds`
- multimodal hashes
- `cache_salt`
- chat / responses rendering 路径
- cudagraph dispatch
- cascade attention

说明系统正在被迫处理这三层问题，只是还没有把它们收束成一个 thesis。

## 6. 与现有工作的边界

组合版 thesis 的危险近邻主要有四组。

### 6.1 `Prompt Cache`

`Prompt Cache` 解决的是：

- modular / structured prompt reuse。

我们的边界是：

- 不做模块级灵活复用；
- 坚持 exact reuse；
- 关注 runtime-side exactness contract。

### 6.2 provider prompt caching 文档与 `Don't Break the Cache`

这些工作已经说明：

- tools、images、schema、tool results、thinking 都会影响缓存；
- 动态内容位置和顺序会影响收益；
- agentic workload 下 naive full-context caching 不稳。

我们的边界是：

- 不只是总结最佳实践；
- 而是把这些规则内化成 vLLM 的 ABI、typed exactness 和 mode selection。

### 6.3 `Serve Programs, Not Prompts` / `Pie` / `ThunderAgent`

这些工作把 serving 对象提升到：

- program
- inferlet
- workflow

我们的边界是：

- 不做 program-aware serving runtime；
- 不接管工具编排；
- 只聚焦现有 `vLLM` 内部的 exact-prefix contract。

### 6.4 `LAPS` 及 prefix-aware scheduling 系列

这些工作已经做了：

- short-prefill batching
- waiting window
- graph clustering
- prefix-aware routing / scheduling

我们的边界是：

- 不写成 length-aware batching；
- 而是写成 **exact hit 之后的 residual mode selection**。

## 7. 当前排重结论

截至 `2026-04-15`，我对组合 thesis 的排重判断是：

1. 三个组成部分分别都不是零重合。
2. 但我没有找到一篇公开系统论文，把下面三件事同时作为一个统一系统来写：
   - 输入稳定性 ABI
   - 异构输入上的 exact-prefix contract
   - 命中后的 residual execution-mode selection
3. 因此，**组合版 thesis 比任何一条单线都更可防守**。

更严谨的说法应该是：

> 没有找到完全同构的公开论文，但不能保证没人做过。

## 8. 在 vLLM 中为什么自然

这个 thesis 在 `vLLM` 中不是硬拼出来的，而是顺着已有代码长出来的。

### 8.1 第一层已有胚芽

`Responses API` 当前在构造下一轮输入时，已经表现出弱版本的公私分离：

- 旧 `system` 不自动继承；
- 旧 `reasoning output` 被显式跳过。

这说明：

- 上层“什么该进入下一轮前缀”已经不是空白问题。

### 8.2 第二层已有 typed carriers

`vLLM` 当前输入和请求路径已经同时带有：

- `prompt_embeds`
- `mm_hashes`
- `cache_salt`

说明系统已经不再是 token-only APC。

### 8.3 第三层已有 mode-choice tension

当前 `gpu_model_runner` 与 attention backend 已经存在非常具体的权衡：

- `dispatch_cudagraph(... disable_full=use_cascade_attn or has_encoder_output)`
- cascade attention 依赖启发式阈值
- `FULL` 与 `PIECEWISE` graph mode 的适用条件不同

说明命中之后到底怎样执行，并不是 trivial 的固定路径。

### 8.4 当前代码里的具体抓手

这条 thesis 在当前代码里至少有六个直接落点：

1. `vllm/entrypoints/openai/responses/utils.py`
   - 继续构造下一轮输入时，会过滤旧 `system`，并显式跳过旧 `reasoning output`。
2. `vllm/inputs/engine.py`
   - `MultiModalInput` 已把 `mm_hashes` 与 placeholder layout 作为一等输入字段。
3. `vllm/inputs/preprocess.py`
   - `cache_salt`、`prompt_embeds` 和文本 / 多模态路径在进入 engine 前就已经分流。
4. `vllm/v1/request.py`
   - request 生命周期里会持续更新 block hashes，说明 prefix 语义已经是 runtime 热路径的一部分。
5. `vllm/v1/worker/gpu_model_runner.py`
   - cudagraph dispatch 直接受 `use_cascade_attn` 与 `has_encoder_output` 影响，存在明确 mode-choice 张力。
6. `vllm/v1/attention/backends/flash_attn.py`
   - cascade attention 仍然依赖启发式阈值，而不是全局最优器。

这六个抓手共同说明：

- 第一层并不是“应用层空白”；
- 第二层并不是“typed carrier 不存在”；
- 第三层也不是“命中后只剩线性执行”。

因此，`ExactPrefixKV` 更像对这些分裂能力的系统化重组，而不是凭空引入一个新 subsystem。

## 9. 最小原型建议

最小原型不要一口吃成胖子，建议分四步。

### P0：统一观测

先补 trace 与统计：

- miss breaker 分类
- typed carrier 覆盖率
- typed miss / conservative disable 的比例
- APC high-hit 下 residual length histogram
- graph replay / eager fallback / cascade enable reason

### P1：实现 Cacheability ABI 诊断

在 renderer / preprocess 路径输出：

- stable prefix signature
- dynamic suffix signature
- breaker reason

先做 observability，不改策略。

### P2：实现 Typed Exact Prefix Contract

只先做三类 carrier：

1. token span
2. prompt embeds
3. multimodal hash + placeholder + cache scope

先统一 canonicalization、hash 与 invalidation。

### P3：实现 Post-Hit Mode Selection

当 typed prefix hit 成立后，再做轻量 mode selection：

- reuse as-is
- bounded recompute
- tiny delay for grouping

先做 rule-based，不上 learned policy。

## 10. 论文 claim 该怎么写

安全的 claim 应该是：

1. 现有 prefix cache 缺少端到端 exact-prefix contract。
2. 这一缺口同时存在于：
   - 输入稳定性
   - runtime exactness
   - post-hit execution
3. `ExactPrefixKV` 通过三层系统统一这三者，在 `vLLM` 上带来：
   - 更稳定的 hit
   - 更广的 workload coverage
   - 更低的 TTFT
   - 更好的吞吐/尾延迟折中

不安全的 claim 是：

- 首次提出 typed prefix caching
- 首次提出 cacheability ABI
- 首次提出命中后 residual scheduling

这些单点都太容易被反驳。

## 11. 当前最推荐的推进顺序

如果现在继续做，我建议顺序是：

1. 先把这条组合主线作为新的主 thesis 固定下来；
2. 第一实现优先级做 `TypedPrefixKV` 的 P0/P1；
3. 同时补 `ResidualModeKV` 的 profiling；
4. 把 `PromptABIKV` 作为 trace/diagnostics 层一起做，而不是单独立项。

更直接一点说：

- 先证明 exact-prefix contract 缺位；
- 再证明命中后仍有执行模式浪费；
- 最后再把两者串成完整系统论文。

## 12. 当前判断

如果现在必须在现有所有方向里选一个最值得继续推进的主 thesis，我的建议就是这条组合版：

> `ExactPrefixKV: Contract-Guided Exact Prefix Serving in vLLM`

它不是三条旧线的简单叠加，而是把三条线放进同一条因果链里。相比继续押注 `PublicStateKV` 或 `ZeroLagKV`，这条路现在更稳，也更有希望写成一个防守边界更清楚的系统故事。

## 13. 参考资料

- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Don't Break the Cache: <https://arxiv.org/abs/2601.06007>
- Serve Programs, Not Prompts: <https://arxiv.org/abs/2510.25412>
- Pie: <https://arxiv.org/abs/2510.24051>
- LAPS: <https://arxiv.org/abs/2601.11589>
- OpenAI Prompt Caching: <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
