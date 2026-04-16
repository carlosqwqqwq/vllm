# 审稿意见分析与补强路线（2026-04-15）

## 1. 总判断

这份审稿意见整体上是高质量、且大体成立的。

它不是在挑措辞小毛病，而是在指出一个更根本的问题：

- 我们当前最强的证据形态，仍然是
  `measurement-backed problem + two prototype bridges`
- 但文稿中已经有不少地方在按
  `unified safe system already validated`
  的口气写

这两者之间存在真实张力。

如果不处理这个张力，论文会在三个维度上同时失分：

1. claim 太强
2. 实验证据口径不统一
3. 系统贡献与问题贡献之间还没有完全闭环

因此，这份意见不能用“部分 rebuttal 就够了”的方式处理，而必须分成两层来应对：

1. 写作层立即收紧 claim
2. 系统与实验层补齐真正缺失的证据

## 2. 对主要问题的逐条判断

### 2.1 “安全恢复”并没有被真正证明，只是被定义了

#### 判断

- 这个批评基本成立。

#### 为什么成立

我们当前在文稿里给出的 `soundness` 仍然是条件式的：

- 如果 `SafeReuseContract` 的谓词都被正确、完备、在线地验证，
  那么恢复是安全的。

这在逻辑上没有错，但它不是“已经证明系统安全”，而是：

- “给出了 soundness-first 的判定原则”
- “并说明只要这些谓词可执行且成立，就不会越过安全边界”

问题在于，当前文稿自己已经承认：

- 还没有完整的 `top-k / rank / logprob` 输出等价性审计
- 还没有把所有 contract predicate 变成完整、可执行、可审计的在线判定器

因此，如果摘要或引言写成：

- “安全稳定兑现”
- “理论证明我们的方法是安全的”

就会比现有证据更强。

#### 当前最稳的回答

当前最稳的说法应当是：

- 我们提出的是 `soundness-first recovery discipline`
- 它把“什么时候能恢复、什么时候必须拒绝恢复”形式化了
- 当前只给出了初步运行证据与局部正确性验证
- 尚不能把它写成“完整安全性已经被理论和实验充分证明”

#### 必须补的硬证据

如果坚持系统论文路线，这一条至少要补两块：

1. 完整输出等价性审计
   - 文本输出
   - `prompt_logprobs`
   - `top-k` 集合
   - rank 稳定性
   - 必要时加 tolerance / exact 两套口径
2. contract predicate 的可执行化
   - 不是只在文中定义
   - 而是明确哪些谓词由运行时在线检查
   - 哪些谓词仍是保守拒绝

#### 对文稿的直接修改建议

- 题目、摘要、引言里不要再写“安全稳定兑现”
- 改成：
  - “soundness-first recovery”
  - “conservative exact-hit recovery”
  - “under explicit reuse contracts”

### 2.2 论文自称统一系统，但实验实际上还是“两条分离机制 + 两种证据口径”

#### 判断

- 这个批评成立，而且是当前系统论文姿态中最伤的一条。

#### 为什么成立

当前证据结构确实是分裂的：

1. 第一条 bridge
   - `prompt_logprobs selective replay`
   - 主要证据来自 dedicated validation runner
2. 第二条 bridge
   - `TwinPathServe backend mismatch`
   - 主要证据来自 `MT-Bench W5` public runner

这可以证明：

- 两条机制都“有效”

但还不能证明：

- “统一 substrate 已经被同一套评测框架验证”

换句话说，当前最强事实是：

- 我们有一个统一 thesis
- 也有两条具体机制
- 但还没有一套统一 runner / baseline matrix / public workload protocol
  把它们真正收束成一个系统结果

#### 当前最稳的回答

当前应把文稿里的 `TwinPathServe` 写成：

- “统一系统 thesis 与 runtime substrate”

而不是：

- “已经被完整公共评测验证的统一系统”

#### 必须补的硬证据

必须补齐同一口径下的三件事：

1. 同一 public runner 层支持
   - `W1 selective replay`
   - `W5 backend mismatch`
2. 同一 baseline matrix
   - vanilla
   - ad hoc fix
   - unified TwinPathServe
3. 同一 workload family 组合下的比较
   - 不能永远一条机制一个脚本一个口径

#### 这条意见对路线的实际含义

如果这三件事补不出来，就应主动降级 framing：

- 从“强系统论文”降到
- “measurement-backed problem paper with prototype bridges”

### 2.3 W5 的 backend mismatch 识别有后验解释风险

#### 判断

- 这个批评大体成立，而且非常关键。

#### 为什么成立

当前 `W5` 的逻辑里，确实存在 reviewer 所说的风险：

1. vanilla runtime 里
   - 这些请求并不会原生被记成 `backend_mismatch`
2. `TwinPathServe` 里
   - admission policy 又会依据 family 历史与 route metadata
     给它们打上 `backend_mismatch`
3. 最后实验再报告
   - `route_reason = backend_mismatch`

如果这个标签本身来自控制面策略，再拿它来证明控制面策略是对的，
就会构成循环论证风险。

#### 当前最稳的回答

当前最稳的写法不是：

- “系统在线识别出了 backend mismatch”

而是：

- “当前 policy 在一个由外部测量支持的 backend-mismatch 假设空间内进行路由”

也就是说，`route_reason=backend_mismatch` 现在更像：

- policy action label

而不是：

- 独立 oracle 给出的 ground truth label

#### 必须补的硬证据

至少要把“识别”和“利用”拆开：

1. 独立 oracle
   - 由离线 reference / compatibility 审计产生标签
   - 这个 oracle 不能依赖 TwinPath policy 自己的决策结果
2. held-out evaluation
   - 用一部分 family 构建 policy
   - 在另一部分 family 上验证识别与恢复
3. 在线判定与路由解耦
   - 报告 policy precision / recall
   - 再报告 route 带来的性能收益

#### 论文中必须避免的写法

不能再把下面两句话混成一句：

- “我们判断它是 backend mismatch”
- “因此我们把它路由到 reuse pool，收益也很好”

应该拆成：

1. 外部识别或独立 oracle 说明它属于 backend mismatch family
2. policy 根据有限在线信号近似识别并路由
3. 评估 policy 的正确率与收益

### 2.4 `logical_hit_tokens` 不是 ground truth，而是构造出来的 oracle

#### 判断

- 这个批评部分成立，而且必须正面承认。

#### 为什么成立

当前 `logical_hit_tokens` 的确不是自然观察量，而是：

- 用 reference-compatible 路径的 `physical_hit_tokens`
  构造出来的 counterfactual oracle

它的价值在于：

- 给出“如果兼容性障碍不存在，系统大概能兑现到什么程度”的上界参照

但它的问题也很明确：

- 它不是无条件 ground truth
- 尤其在 `W5` 里，backend class 本身就在改变可兑现边界

因此 reviewer 说它“可能把 hypothesis 编进 metric”，这个担心是合理的。

#### 当前最稳的回答

我们最稳的应对不是硬说它就是 ground truth，
而是明确把它定义为：

- `reference-path upper-bound oracle`
- 或者：
- `logical exact-hit upper bound under a compatible reference execution`

也就是说：

- `logical_hit_tokens` 仍然有用
- 但它是上界型、反事实型 oracle
- 不是客观自然存在的唯一真值

#### 必须补的硬证据

至少应增加两类稳健性分析：

1. 多 reference sensitivity
   - 不同 compatible backend
   - 不同 scheduling shape
   - 看 `logical_hit_tokens` 是否稳定
2. upper/lower bound reporting
   - 不只报一个 oracle
   - 而是报 conservative lower bound 与 reference upper bound

#### 对写作的直接影响

文稿里不能再把 `logical hit` 写成：

- “真正应该命中的 token 数”

更准确的写法应是：

- “under a compatible reference execution, the exact-hit upper bound we seek to recover”

### 2.5 外部效度与 framing 明显不匹配

#### 判断

- 这个批评成立。

#### 为什么成立

我们的 framing 已经铺到了：

- tool-rich
- multimodal
- mixed carrier
- heterogeneous backends

但当前主结果确实只来自：

- `vLLM`
- `Llama-2-7B-Chat`
- `W1/W5`
- `MT-Bench`
- 加上 dedicated validation

这意味着：

- 问题动机写得像一般 serving runtime 问题
- 证据却仍然是两个重点 case study

#### 当前最稳的回答

应主动收紧 scope：

- 当前论文研究的是
  `vLLM-like exact reuse runtime` 中的 false-negative exact hit
- 而不是已经证明了所有现代 serving 负载都存在同样规模的问题

W2/W3/W4 可以继续保留在 thesis 版图中，
但不应再被写成“主结果已覆盖”。

#### 必须补的硬证据

如果要保住当前 framing，必须至少新增：

1. `LongBench`
2. 一个 agentic / tool-rich benchmark
3. 至少一个非纯文本 family

否则就应主动收紧摘要和引言。

### 2.6 “为什么不能分别修”没有被实验回答

#### 判断

- 这个批评完全成立。

#### 为什么成立

当前我们确实证明了：

- 统一故事讲得通
- 两条 bridge 都有效

但还没有证明：

- unified substrate 比 best ad hoc fixes 更强

在系统审稿里，这两者是完全不同的命题。

如果没有同台 baseline matrix，reviewer 完全可以说：

- 你们只是把两个 patch 打包成了一个更大的故事

#### 必须补的硬证据

这一条没有捷径，必须直接补：

1. vanilla
2. per-feature ad hoc fix
3. unified TwinPathServe

在同一 runner、同一 workload、同一指标下比较。

#### 额外建议

为了让“统一比分别修更强”更像系统论文，
不应只比性能，还应比：

- 覆盖 workload family 的广度
- 新增机制时的增量改动量
- admission / route 层是否能复用

但注意：

- 工程复杂度只能当辅助证据
- 不能代替性能与覆盖面的同台对比

### 2.7 系统评估指标不够顶会标准

#### 判断

- 这个批评成立，而且是系统线能不能走下去的硬门槛。

#### 为什么成立

当前主结果过于依赖：

- `phase wall time`

它对原型验证足够，
但对系统论文来说远远不够。

我们确实还缺：

- `p50/p95/p99 TTFT`
- steady-state throughput
- overload / backpressure 下的 fallback rate
- co-batching 对 twin-pool 的影响
- batching 效率副作用

#### 必须补的硬证据

如果要冲系统论文，这一条必须直接补实验矩阵：

1. latency
   - `TTFT p50/p95/p99`
2. throughput
   - requests/s
   - tokens/s
3. overload
   - queueing
   - fallback
   - route instability
4. batching efficiency
   - batch size distribution
   - padding / co-batching 效率
   - twin-pool admission 对整体吞吐的副作用

## 3. 对次要问题的判断

### 3.1 术语密度偏高

- 这个批评成立。

当前术语系统虽然已经比之前统一很多，
但仍然有 abstraction inflation 的风险。

最直接的处理方式不是再发明更好的术语，
而是减少正文中新术语的数量：

- 保留最核心的 4 到 5 个术语
- 其余移到 supporting 或脚注

### 3.2 route key / prefix family 在线稳定性写得太强

- 这个批评成立。

我们当前更偏向写“理想构造”，
但还缺：

- noisy online workload 里的失败案例
- key 漂移
- family 污染
- 误绑定的后果与 fallback

这一块要补 failure mode，而不是补正向故事。

### 3.3 文稿仍然有 measurement paper 与 system paper 拼接感

- 这个批评成立。

这实际上是前面所有问题的外在表现。

只要 measurement 与 system 的证据口径还是分裂的，
这种拼接感就不会消失。

## 4. 我们现在最应该如何回应

### 4.1 当前最诚实的结论

如果今天就投稿，这篇论文更接近：

- 强问题论文
- 带设计感的系统 thesis
- 但还不是 fully-validated system paper

也就是说，reviewer 让我们“主动降级成 measurement-backed problem paper”，
这不是恶意建议，而是一个真实稳妥的 fallback。

### 4.2 如果坚持系统论文路线，必须补的三大块

这份审稿意见已经把最关键的三块缺口说得很清楚。

我认为完全正确，必须按下面顺序补：

1. `prompt_logprobs selective replay` 的完整输出等价性审计
2. `vanilla / ad hoc / unified` 同台 baseline matrix
3. 多 benchmark 下的 `TTFT/p95/p99/throughput/overload` 闭环

此外，还必须单独加一条：

4. `W5 backend mismatch` 的非循环识别方案

因为这一条如果不补，系统章最核心的因果链会一直被质疑。

## 5. 对题目、摘要、引言的立即修改建议

这部分可以立刻做，不需要等新实验。

### 5.1 题目

题目中不要把“恢复”写得太满，
也不要暗示“已经安全证明、已完全统一部署”。

更稳的方向应强调：

- false-negative exact hits
- exact reuse under contracts
- soundness-first recovery

### 5.2 摘要

摘要必须做三处收紧：

1. 删除或替换
   - “安全稳定兑现”
2. 删除或替换
   - “统一系统已经证明成立”
3. 明确当前证据边界
   - 问题表征强
   - 两条 bridge 有初步 formal evidence
   - 统一系统仍在闭环中

### 5.3 引言

引言应明确：

- 当前工作首先是“定义并测量 false-negative exact hit”
- 然后给出统一 substrate 与两条 bridge
- 最后展示初步系统证据

不要在引言前两段就让 reviewer 以为：

- 我们已经完成了多 benchmark 的系统闭环

## 6. 这份意见对当前执行优先级的影响

这份意见会直接改写后续执行顺序。

当前最优先的事情不应再是：

- 继续发散新方向
- 继续美化高层叙事

而应固定为：

1. LongBench / 更多 public runner 闭环
2. unified baseline matrix
3. output-equivalence audit
4. backend-mismatch 独立识别
5. TTFT / throughput / overload 指标

## 7. 最终判断

如果严格按系统顶会标准看，这份审稿意见是对的：

- 它没有误判我们的方向
- 但准确指出了我们当前还没有把 thesis 做成“闭嘴级系统证据”

这不是坏消息，反而说明论文主线本身是成立的。
真正的问题不是 idea 错了，而是：

- 当前证据还不足以支撑我们已经写出来的最强 claim

因此最合理的策略是：

1. 立刻收紧文稿 claim
2. 继续补统一 public runner 与 formal metrics
3. 在补不齐前，不把论文包装成 fully-validated system paper

换句话说，这份意见不是让我们推倒重来，
而是在强迫我们把“论文姿态”和“实际证据”重新对齐。
