# FalseNegativeKV 公开 Benchmark 驱动评测协议

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

- 把 `FalseNegativeKV` 后续实验从“自造 workload 为主”收束到“公开 benchmark 为主”

它不负责：

- 改写 `H1-H4` 的论文级评测门槛
- 替代 `paper_evaluation_matrix.md`
- 为所有 workload family 立即提供恢复实现

从现在开始，`FalseNegativeKV` 的主结果必须优先建立在公开 benchmark / 公开数据集之上。自造 synthetic workload 只允许承担下面三种角色：

1. smoke test
2. 机理 debug
3. appendix 中的补充控制实验

不允许把 synthetic workload 当作主结果的主体。

## 2. 公开数据优先的固定原则

### 2.1 固定原则一：主表优先使用公开 benchmark

论文主表、主图和 headline result 优先来自：

- 公开 benchmark
- 公开评测集
- 公开可下载的数据集

不使用：

- 私有 trace
- 手工编造的问答对
- 只存在于内部脚本中的 synthetic corpus

### 2.2 固定原则二：允许“工作负载重放”，不允许“重新造数据”

为了评测 serving runtime，我们允许对公开 benchmark 做最小工作负载重放，但重放必须局限于下面四类操作：

1. 固定公开可复现的 system / tool wrapper
2. 依据公开样本构造 `prefix family`
3. 在相同公开样本上切换 API contract 或 backend class
4. 调整请求到达顺序、并发度和 warm/cold cache 条件

不允许：

1. 人工新写问题与答案
2. 人工新造长文档作为主数据
3. 手工拼接只服务于某一机制的主评测集

### 2.3 固定原则三：公开 benchmark 也要区分“主结果”和“补充结果”

即便都来自公开数据，也不是所有 benchmark 都同等重要。

从现在开始：

- `W1`、`W2`、`W5` 的主结果必须来自公开 benchmark
- `W3`、`W4` 至少要有公开 benchmark 的 prevalence 测量
- 若某 family 暂时只能用 synthetic 才能跑通，则只能进入 appendix，不进入正文主结论

## 3. workload family 到公开 benchmark 的固定绑定

## 3.1 `W1`：长共享前缀 / 多轮 chat

### 主 benchmark

- `LongBench / LongBench v2`

### 辅 benchmark

- `MT-Bench`

### 绑定理由

`W1` 的核心不是普通 chat 打分，而是：

- 长共享前缀
- 多个请求之间的稳定 family
- 可切换 `prompt_logprobs` 等消费条件

`LongBench` / `LongBench v2` 提供公开长上下文任务，适合构造长共享前缀 family；`MT-Bench` 提供公开多轮对话结构，适合作为多轮 chat 语义的补充验证。

### 允许的最小重放方式

只允许：

1. 使用 benchmark 原始公开 context / conversation
2. 对同一公开 context 关联的多个问题或多个对话 turn 构造 family
3. 加固定 system wrapper，并在相同 wrapper 下切换 `prompt_logprobs`

不允许：

1. 手工扩写长系统提示词作为主数据来源
2. 自己编造额外对话 turn 进入主结果

### 在论文中的角色

- `W1` 是 `prompt_logprobs selective replay` 的主战场
- 主 headline result 可以优先来自这一组 benchmark

## 3.2 `W2`：tool-rich / agentic

### 主 benchmark

- `BFCL`

### 辅 benchmark

- `ToolBench`

### 绑定理由

`W2` 需要的不是普通问答，而是：

- 固定 tool schema
- 多轮或多步函数调用
- 同一工具描述在多个请求中反复出现

`BFCL` 当前已经公开包含 real-world tool-use、multi-turn interaction 与 agentic evaluation；`ToolBench` 则提供更大规模的公开 tool-use 基础。

### 允许的最小重放方式

只允许：

1. 复用 benchmark 原始公开 tool schema / function schema
2. 以相同 tool catalog 构造 `prefix family`
3. 在不改动问题语义的前提下切换 API contract 或 backend

不允许：

1. 手工编写新工具定义作为主评测集
2. 自行扩写新的多步 agent 任务进入主表

### 在论文中的角色

- `W2` 是证明“这不是 niche `prompt_logprobs` 特例”的关键 family
- 若 `W2` 没有公开 benchmark 支撑，全文很容易被打成 microbenchmark paper

## 3.3 `W3`：`prompt_embeds`

### 主 benchmark

- `MTEB`

### 辅 benchmark

- `BEIR` 系列 retrieval 数据

### 绑定理由

`W3` 的目标不是回答 embedding 模型质量，而是回答：

- 当 client 以 `prompt_embeds` 而非 token 形式提交前缀时
- serving runtime 是否更容易出现 carrier mismatch 型 false-negative

`MTEB` / `BEIR` 提供公开文本检索与嵌入任务数据，适合作为 `prompt_embeds` carrier 的公开来源。

### 允许的最小重放方式

只允许：

1. 使用公开 query / document 文本
2. 用固定公开 embedding model 或固定本地 embedding 预处理脚本生成向量
3. 在相同公开文本上切换 token carrier 与 embed carrier

不允许：

1. 人工构造只为触发 embed mismatch 的伪文档集作为主结果

### 在论文中的角色

- `W3` 当前以 prevalence measurement 为主
- 在 P1 中不要求必须完成恢复，只要求证明问题面更广

## 3.4 `W4`：multimodal

### 主 benchmark

- `MMBench`

### 辅 benchmark

- 通过 `VLMEvalKit` 可直接获取的公开多模态 benchmark

### 绑定理由

`W4` 的关键在于：

- 相同图像或相同视觉上下文反复出现
- 文本 suffix 或任务要求变化
- 系统可能因 carrier / partial prefill 约束丢掉本应存在的复用

`MMBench` 是公开、稳定、可通过官方工具链下载和评测的多模态 benchmark，适合作为 `W4` 的主公开来源。

### 允许的最小重放方式

只允许：

1. 使用 benchmark 原始公开图像与问题
2. 按相同 image id 或相同文档图像构造 family
3. 加固定公开 wrapper，不改变样本本身语义

不允许：

1. 手工合成图片-问题对作为主表数据
2. 自行生成图像的合成 workload 作为主结果

### 在论文中的角色

- `W4` 主要承担“公开多模态场景下也存在 false-negative”这一测量角色
- 若恢复机制未闭环，可以只进入 prevalence 图，而不进入 recovery 主表

## 3.5 `W5`：APC 高命中但 residual 碎片化 / backend mismatch

### 主 benchmark

- `LongBench / LongBench v2`

### 辅 benchmark

- `InfiniteBench` 中的真实长上下文子集

### 绑定理由

`W5` 需要的是：

- 足够长的公开 context
- 同一 context 上的多个公开 query
- 在不同 backend class 下重复运行相同 family

`LongBench` 适合构造现实长上下文 family；`InfiniteBench` 可作为更长上下文的补充来源，但正文主结果优先使用其中真实任务子集，而不是 synthetic retrieval 子集。

### 允许的最小重放方式

只允许：

1. 在相同公开 long-context 样本上切换 backend class
2. 保持 query、context 和 family 不变
3. 比较 reference backend 与 probe backend 的复用差异

不允许：

1. 主要依赖 synthetic key-value retrieval 子集来支撑正文主结果
2. 为了放大效果手工拼接超级长文档作为主数据

### 在论文中的角色

- `W5` 是 `TwinPathServe` 双池路线必须拿下的公开 benchmark family
- 如果 `W5` 只能靠 synthetic workload 成立，backend mismatch 主张会明显变弱

## 4. 固定 benchmark 选型优先级

从现在开始，数据选型按下面的优先级执行：

1. benchmark 原生就带多轮 / 多问题 / 多工具结构
2. benchmark 原生提供长 context 或多模态载体
3. benchmark 有官方 repo、官方脚本或官方 leaderboard
4. benchmark 已被社区广泛使用，审稿人认知成本低
5. benchmark 可以在 `vLLM` 或相邻工具链中直接落地

若两个 benchmark 同时满足条件，优先选择：

- 下载更稳定
- 脚本更成熟
- 更容易构造 `prefix family`

## 5. 与 baseline 和相关工作的对比策略

公开 benchmark 选定之后，固定对比顺序仍然不变：

1. `vanilla vLLM V1`
2. status quo skip / disable 行为
3. ad hoc per-feature fix
4. `FalseNegativeKV` P1

此外再补一条公开 benchmark 驱动下的固定原则：

- 相关工作只有在问题边界足够接近、且可复现时，才进入同机实测对比。

不强行把下面这类工作做成主实验 baseline：

- 长上下文 general benchmark paper
- 分布式 KV / prefill-decoding disaggregation 系统
- 公平调度或资源调度论文

它们应主要出现在 `Related Work` 和 `Discussion` 中，而不是硬塞进同机主表。

## 6. 主结果与补充结果的固定要求

### 6.1 正文主结果

正文主结果至少满足：

1. 至少三个 workload family 来自公开 benchmark
2. 其中至少两个 family 的恢复结果来自公开 benchmark
3. 至少一个 tool-rich 或 agentic family 来自公开 benchmark
4. 至少一个 backend mismatch family 来自公开 benchmark

### 6.2 appendix / 补充结果

下面这些可以留到 appendix：

- synthetic control workload
- 极端长上下文 stress test
- 单机制 debug 专用 microbenchmark

### 6.3 不允许的弱写法

不允许在正文中写成：

- “我们主要使用自造 workload，因为更容易控制变量”
- “公开 benchmark 不容易触发问题，所以主结果使用内部构造样本”

这种写法会让审稿人自然认为：

- 问题不够真实
- 结论不够通用
- 系统收益主要来自工作负载挑选

## 7. 论文中对“最小重放”的固定表述

为了避免 reviewer 把公开 benchmark 上的 workload replay 理解为“重新造数据”，正文建议固定使用下面这条表述：

> 我们不重新标注或重新编造任务数据；相反，我们只在公开 benchmark 的原始样本上施加最小运行时重放，包括固定 system/tool wrapper、family grouping、API contract 切换与 backend 切换，以测量同一公开样本在不同 serving 条件下的复用兑现差异。

中文更简洁的写法可以是：

> 本文不自造主评测集，只对公开 benchmark 样本进行最小工作负载重放，以评估同一公开样本在不同运行时条件下的复用损失与恢复效果。

## 8. 立即执行的公开 benchmark shortlist

从现在开始，优先落地下面这一组公开 benchmark shortlist，不再继续发散：

1. `LongBench / LongBench v2`
   - 负责 `W1` 与 `W5`
2. `MT-Bench`
   - 作为多轮 chat 辅验证
3. `BFCL`
   - 负责 `W2`
4. `MTEB`
   - 负责 `W3`
5. `MMBench`
   - 负责 `W4`

只有当这五组 benchmark 中某一组确实无法在当前环境落地时，才允许启用备选 benchmark。
