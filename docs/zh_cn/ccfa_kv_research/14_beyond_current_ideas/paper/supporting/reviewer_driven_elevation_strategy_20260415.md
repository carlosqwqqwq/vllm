# 按审稿意见提标而不降格的方案（2026-04-15）

## 1. 目标不是“把话说小”，而是“把贡献重心抬高”

面对这类审稿意见，最危险的错误有两个：

1. 硬顶
   - 明明证据还不够，却坚持把论文写成“统一安全系统已完整证明”
2. 过度收缩
   - 为了避开 reviewer 质疑，直接把论文降成
     `measurement paper + 两个 patch`

这两条路都不对。

真正合理的路线是：

- 该收紧的 claim 必须收紧
- 但贡献重心不能下降
- 相反，要把论文从“两个 bridge 的组合”
  提升成“一个 correctness-aware reuse control plane”

也就是说，我们不是把贡献写小，
而是把贡献从“修了两个点”
抬到“提出了一个新的 serving runtime capability layer”。

## 2. 当前论文真正应该争取的高度

### 2.1 不是 feature patch paper

如果继续把工作写成：

- `prompt_logprobs` 的复用补丁
- backend mismatch 的双池补丁

那 reviewer 对“为什么不分别修”的质疑会永远成立。

### 2.2 也不是纯 measurement paper

如果退成：

- 我们发现了一个新问题
- 顺便展示两个 prototype

又会把已经做出来的系统价值浪费掉。

### 2.3 应该抬到的目标高度

更合适的目标高度是：

> 本文提出的不是若干 isolated fixes，而是一个
> `correctness-aware recovery substrate for false-negative exact reuse`

这句话的意义在于，它把我们的贡献分成三层：

1. 问题层
   - `false-negative exact hit` 是独立的问题类
2. 原理层
   - 需要一层 correctness-aware / contract-aware 的 recovery control plane
3. 系统层
   - 两条 bridge 只是这层 substrate 的实例，而不是论文的全部

只要这一层立住，论文的高度就不再依赖“有没有刚好修了两个功能”，
而是依赖：

- 我们是否定义了一个此前缺失的系统能力

## 3. 如何在不虚夸的前提下扩大贡献范围

### 3.1 把“统一 runtime substrate”抬高成“correctness-aware reuse control plane”

当前 `TwinPathServe` 写法里最容易被 reviewer 压回去的点是：

- 它看起来像一个 twin-pool runtime

这会让贡献显得太工程化。

更高的写法应当是：

- `TwinPathServe` 是系统实例
- 真正的抽象层是 `correctness-aware reuse control plane`

这层 control plane 至少包括四个部件：

1. `reuse oracle`
   - 用来定义 reference-compatible 上界与 logical hit 边界
2. `reuse contract`
   - 用来定义什么时候允许恢复、什么时候必须拒绝恢复
3. `bridge operator`
   - 用来把 producer state 转成 consumer-visible reuse
4. `route / admission policy`
   - 用来在多执行路径中选择最可能安全兑现 reuse 的路径

这样一来：

- `prompt_logprobs selective replay`
  是一种 `bridge operator`
- `TwinPathServe twin-pool route`
  是一种 `route / admission policy`

论文就不再是“两个点子拼一起”，
而是“control plane 的两个实例化”。

### 3.2 把“安全性”抬高成“soundness-first systems discipline”

Reviewer 质疑“安全没有被证明”，这在当前证据下是成立的。

但这不意味着我们要放弃安全主张，
而是要把主张从：

- “我们已经证明系统完全安全”

改成：

- “我们提出了一个 `soundness-first systems discipline`”

这会把贡献从单个系统实现抬高到方法论层面。

对应地，论文可以主张的高度变成：

1. 我们区分了 `soundness` 与 `completeness`
2. 我们给出了 conservative recovery 的系统纪律
3. 我们说明了 contract、oracle、fallback 应该怎样组织
4. 我们用两个实例证明这套纪律不是空谈

这比“我们已经完全证明安全”更稳，
也比“我们只做了一些保守 patch”更高。

### 3.3 把“问题测量”抬高成“counterfactual reuse measurement”

`logical_hit_tokens` 被批评不是 ground truth，这个批评不能硬顶。

但我们完全可以把这一块从弱点转成贡献：

- 现有 serving 系统只观测 `physical hit`
- 我们提出的是一种
  `counterfactual reuse measurement`
- 它通过 compatible reference execution
  去估计“本该可兑现、但被 runtime 丢掉”的 exact reuse 上界

只要写清楚它是：

- upper-bound oracle
- 不是自然真值

那么这块工作反而会成为论文里一个更系统的方法学贡献。

也就是说，我们不把它写成：

- “这是 ground truth”

而把它写成：

- “这是 measuring false-negative exact reuse 所必须引入的 counterfactual oracle”

### 3.4 把“两个 case”抬高成“同一 pathology 在不同层的实例”

当前 reviewer 最大的不满之一是：

- 两条机制像两个分离故事

我们要做的不是硬把它们说成已经统一验证，
而是把它们抬到同一个分层框架里：

1. `observation mismatch`
   - consumer contract mismatch
2. `backend mismatch`
   - execution path mismatch

然后强调：

- 这不是两个独立 feature bug
- 而是同一类 pathology 在两个不同 runtime layer 的暴露

只要这个层次被讲清楚，
“统一性”就不再只依赖统一 runner，
而首先依赖统一病理结构。

## 4. 如何扩大工作的范围和可信度

### 4.1 扩范围不能靠把动机写大，必须靠 benchmark 面扩出来

当前文稿最大的问题之一是：

- framing 已经写到现代 serving runtime 一般问题
- 但主结果还主要来自 `MT-Bench + dedicated validation`

要扩大范围，不能再靠写作铺陈，
必须靠 benchmark 面真正扩出来。

最合理的扩展顺序是：

1. `LongBench`
   - 回答长上下文、公开数据、真实 repeated context family
2. `BFCL` 或 tool-rich benchmark
   - 回答 agentic / tool-rich workload
3. `prompt_embeds` / multimodal controlled benchmark
   - 回答 mixed carrier

这样扩展之后，论文的范围就不是“靠摘要写大”，
而是“靠 workload family 扩大”。

### 4.2 可信度要靠“识别”和“恢复”解耦

Reviewer 指出 `W5` 有循环论证风险，这一点非常关键。

要增加可信度，最核心的动作不是改文案，
而是把下面两个问题拆开：

1. 这个 family 是否真的属于 `backend mismatch`
2. 如果属于，route 恢复是否真的有效

因此，我们必须引入：

- independent mismatch oracle
- held-out policy evaluation
- policy precision / recall

只要这三项补上，
`W5` 的可信度会显著上升。

### 4.3 可信度还要靠 best-baseline matrix

“为什么不能分别修”这一条是系统论文的生死线。

这里不能再只做 narrative 反驳，
必须靠同台 baseline：

1. vanilla
2. best ad hoc fix
3. unified substrate

而且不只是比绝对性能，
还要比：

- 覆盖 workload 的广度
- 新增一类 mismatch 时的增量实现代价
- control plane 是否复用

这样 reviewer 才会接受：

- unified substrate 不是把两个 patch 打包
- 而是减少系统碎片化的必要层

### 4.4 可信度需要完整 output-equivalence audit

这一条没有讨价还价空间。

如果要继续保住 `soundness-first` 这条主轴，
就必须补：

1. token output equality
2. `prompt_logprobs`
3. `top-k` 集合
4. rank 稳定性
5. tolerance vs exact 两套判定

这项工作本身也能抬高论文高度，因为它会把我们的贡献从：

- “一个看起来有效的优化”

抬到：

- “一个被 correctness harness 约束的 serving recovery mechanism”

## 5. 如何优化论文主张，而不是让它缩水

### 5.1 当前应该收紧的是“结果型 claim”

必须收紧的包括：

- `safe and stable recovery`
- `fully unified runtime validated`
- `general solution across modern workloads`

这些都属于结果型 claim，
当前证据还不够。

### 5.2 当前应该抬高的是“结构型 claim”

应该主动抬高的包括：

1. 我们定义了一个此前没被清晰命名的问题类
2. 我们提出了一个新的 correctness-aware reuse control plane
3. 我们给出了 measurement oracle、reuse contract、bridge operator、
   route policy 的统一组织方式
4. 我们证明这不是纯 measurement，也不是纯 patch，而是一个新的 runtime capability

这类 claim 更高，也更稳，
因为它们不依赖某一个 benchmark 上某几个百分点的提升。

### 5.3 题目和摘要的更高版本应该怎么写

如果要在不夸大的前提下抬高高度，题目和摘要应围绕下面这个核心：

> 现代 serving runtime 缺的不是更多 cache key，
> 而是一层 correctness-aware reuse recovery capability

这样一来：

- 问题更大
- 贡献更高
- 但不会虚构“我们已经全部做完”

## 6. 具体执行上，接下来该怎么改

### 6.1 文稿层

立刻要改三类地方：

1. 标题 / 摘要 / 引言
   - 收紧结果型 claim
   - 抬高结构型 claim
2. 系统设计章
   - 把 `TwinPathServe` 从 twin-pool runtime 抬成
     correctness-aware reuse control plane 的一个实例
3. 实验章
   - 明确区分：
     - problem evidence
     - bridge evidence
     - unified system evidence

### 6.2 实验层

实验优先级应固定为：

1. `LongBench` prevalence 正式结果
2. `W1 selective replay` 统一 public runner
3. `vanilla / ad hoc / unified` baseline matrix
4. `W5` 独立识别 oracle
5. `TTFT/p95/p99/throughput/overload`
6. output-equivalence audit

### 6.3 叙事层

叙事上不应再把论文写成：

- “我们已经完成统一系统”

而应写成：

- “我们提出了统一 control plane，并已经在两类 mismatch 上完成初步系统实例化与 formal evidence”

这比主动降级成 measurement paper 更强，
也比硬吹完整系统更稳。

## 7. 最终建议

如果目标是继续冲系统顶会，
当前最优路线不是退，也不是硬冲，
而是：

1. 收紧不成立的强结果 claim
2. 抬高真正成立的结构性贡献
3. 用下一轮实验把统一 control plane 的证据补齐

一句话概括就是：

> 我们不要把论文从“系统 thesis”降成“两个 patch 的问题论文”，
> 而要把它从“两个 bridge 的原型论文”提升成“correctness-aware reuse control plane”的系统论文。
