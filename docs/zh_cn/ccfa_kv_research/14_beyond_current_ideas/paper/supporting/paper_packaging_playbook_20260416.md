# 论文包装与写作手册

更新日期：2026-04-16

## 1. 这份文档只负责什么

这份文档只负责：

- 论文包装
- 论文写作
- claim 的呈现方式
- reviewer 第一眼会看到的 paper package

它不负责：

- 新实验设计
- 新实现路线
- 新机制发散

换言之，这份文档处理的是“怎么把现有内容写得像一篇顶会系统论文”，而不是“还应该做什么系统工作”。

## 2. 当前最推荐的 paper package

当前最稳的 package 不是：

- 一个 `prompt_logprobs` patch paper
- 一个 `TwinPathServe` 工程设计报告
- 一个“完整系统已经闭环胜出”的强结果 paper

当前最稳的 package 是：

- 一个 `measurement-backed` 的 runtime problem paper
- 一个 `contract-and-realization-forward` 的 system thesis
- 一个以 `TwinPathServe` 作为第一版系统实例的设计型系统论文

这三句话必须同时成立，缺一不可。

## 3. 标题怎么包

当前最推荐的标题只保留一主一备。

### 3.1 主标题

- `FalseNegativeKV：大模型推理中前缀复用假阴性的测量、表征与契约感知恢复`

### 3.2 备选标题

- `TwinPathServe：面向前缀复用假阴性的 correctness-aware reuse control plane`

标题层的固定原则只有两条：

1. 问题名必须出现
2. 方法主张必须出现，但不能把标题写成单个 feature patch

## 4. 一句话 thesis 怎么写

当前最稳的一句话 thesis 固定为：

> 现有大模型 serving runtime 只显式优化 `physical hit`，却没有系统建模那些在参考执行条件下本应成立、但没有在当前请求中兑现为物理命中收益的 `false-negative exact hit`；`FalseNegativeKV` 将其提升为一个可测、可解释、可恢复的系统问题。

这一句的作用不是“解释全部技术细节”，而是先把 paper 的智识重心钉死。

## 5. 摘要怎么包

当前摘要最稳的顺序固定为六步：

1. 背景：
   - prefix reuse 已是 serving 核心优化
2. 缺口：
   - 现有 runtime 主要优化 `physical hit`
3. 问题：
   - `logical hit` 在运行前被丢失，形成 `false-negative exact hit`
4. 方法：
   - `measurement surface + reuse contract + realization policy + runtime bridge/route`
5. 当前证据：
   - `W1` 与 `W5` 的强证据
6. 意义：
   - exact reuse 应被写成 contract-aware runtime capability，而不是 scattered fixes

摘要层还必须额外遵守两条文风约束：

1. 不要在摘要中一次性引入过多内部技术术语，尤其避免把 `measurement surface`、`reuse contract`、`realization policy`、`bridge operator` 等内部层次全部堆进同一段。
2. 不要把摘要写成技术报告式模块罗列；摘要应优先呈现研究问题、经验发现、核心思想和总体结论，而不是实现分层与接口细节。

当前摘要最值得保留的风格是：

- 问题写强
- 方法写清
- prototype 结果写实
- 最后一两句不越界

## 6. 引言怎么包

当前引言最推荐固定成六段，但真正决定第一印象的是前四段。

### 6.1 第一段

先把 prefix reuse 写成 serving 基础设施，而不是局部优化。

### 6.2 第二段

写清现代 mixed-API workload 让 `logical hit` 与 `physical hit` 脱钩。

### 6.3 第三段

用 `prompt_logprobs` 与 backend mismatch 作为统一 motivating example，而不是两个分离故事。

### 6.4 第四段

给出核心 insight：

- 这不是零散 feature gap
- 而是一类 `false-negative exact hit` runtime pathology

如果前四段写稳了，后面的设计和实验就有了承接空间；如果前四段没写稳，整篇论文会很容易被误读成 patch collection。

## 7. 贡献怎么包

从现在开始，最推荐的贡献条数固定为三条，不再扩成四条或五条。

### 7.1 贡献一

- 问题表征与测量

### 7.2 贡献二

- `measurement -> reuse contract -> realization policy -> TwinPathServe` 的统一主线

### 7.3 贡献三

- 两类 bridge 的第一版 prototype 证据

这样包装的好处是：

- 第一条回答“你发现了什么新问题”
- 第二条回答“你提出了什么新系统抽象”
- 第三条回答“你不是只停在概念层”

## 8. 哪些话必须主动写

当前 packaging 层最值得主动强调的点固定有五个。

1. `false-negative exact hit` 不是 prefix miss 的同义词
2. hash 解决 identity，不解决 consumption feasibility
3. 问题发生在 `logical hit -> physical hit` 之间
4. `TwinPathServe` 首先是一层 runtime capability，其次才是两个具体 bridge
5. 本文证明的是 `soundness-first` 的恢复边界，不是所有 logical hit 的完备恢复

## 9. 哪些话绝对不要写

下面这些句子现在不应出现在摘要、引言或结论中。

- `We solve false-negative exact hits in vLLM.`
- `We already demonstrate complete end-to-end wins.`
- `We recover multiple mismatch classes with a unified runtime across all settings.`
- `This is a general solution for prefix cache misses.`

这些写法的问题不是“不够谦虚”，而是会直接把 reviewer 的注意力引到最脆弱的证据缺口上。

## 10. reviewer 第一眼最关心的是什么

如果只看 packaging，reviewer 第一眼通常只关心四件事。

1. 这是不是 patch collection
2. 这是不是 cache key / hash 工程问题
3. 你们的复用是否安全
4. 你们是不是已经把 claim 写过头了

所以当前 paper package 的任务，不是“把所有细节都先讲出来”，而是先在第一页到第二页把这四个疑问压住。

## 11. 当前最推荐的写作优先级

如果后续继续只做包装与写作，优先级应该固定为：

1. 继续打磨题目、摘要最后两句和引言首四段
2. 统一三条贡献与图标题、图注之间的说法
3. 收紧结论里的 claim 边界
4. 准备 reviewer-facing 的短答模板

当前不值得优先投入的写作动作包括：

- 再写新实验计划
- 再扩系统实现细节
- 再新增新方向 overview

## 11.1 学术文体约束

主文稿从现在开始默认采用正式学术论文语体，而不是项目汇报、内部方案说明或 rebuttal 口吻。具体约束如下：

1. 章节开头不要写“本章的职责不是……”或“本章要回答三个问题”这类元叙事说明，直接进入学术内容。
2. 不直接在正文中说“reviewer 会怎么理解”“reviewer 最容易误解的是”，这类判断应转写为中性学术表述。
3. 避免口号式表达，例如“本文拒绝这种写法”“这不是更优雅”，应改写为可论证的学术命题。
4. 避免项目管理式表达，例如“当前最稳”“当前主线”“当前阶段”，除非是在 supporting 文档中讨论写作策略。
5. 正文句子应优先追求定义清楚、逻辑递进和论证克制，而不是强调态度或制造修辞张力。
6. 摘要与引言应避免参数名、配置名、内部模块名的高密度堆叠，除非这些术语对问题定义本身不可替代。

## 11.2 paper-only 工作约束

从现在开始，如果当前轮次的目标是“只关注论文文档的写作”，则默认遵守下面三条约束：

1. 只修改 `paper/` 目录内的主文稿与 supporting
2. 不再为了“联动”去改 `implementation/`、`experiments/` 或根目录入口
3. 外部规格文档只作为背景引用，不再作为当前轮次的主要工作对象

## 12. 一句话结论

当前最好的论文包装，不是把工作写成“我们已经把系统全做完了”，而是把它写成“我们识别并结构化了一个此前没被清晰表述的 serving runtime pathology，并用 `TwinPathServe` 给出第一版 contract-aware 系统化回应”。这才是最像顶会 systems paper 的 package。
