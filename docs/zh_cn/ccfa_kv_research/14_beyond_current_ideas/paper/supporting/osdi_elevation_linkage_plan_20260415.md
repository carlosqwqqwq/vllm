# 从当前方向扩成 CCFA/OSDI 工作量的联动方案

更新日期：2026-04-15

## 1. 先给结论

只做“系统里明明存在 exact 或 logical-exact reuse 条件，但 runtime 没把它兑现成 physical hit”这一句问题定义，本身是对的，但**单独做成一个 case-driven runtime patch 集合，工作量和论文张力都还不够稳**。

要把它撑到 `CCF-A / OSDI` 量级，关键不是再横向发散更多零散 case，而是把当前方向扩成一条有明确层次、明确接口、明确实证闭环的系统 thesis：

- 第一层：`measurement surface`
- 第二层：`reuse contract`
- 第三层：`realization policy`
- 第四层：`TwinPathServe`

这四层分别回答：

- 问题是否真实存在、规模有多大
- 哪些 miss 在语义上属于“可恢复候选”
- 哪些可恢复候选在当前预算与队列条件下值得兑现
- runtime 如何把这些判定落实成真实 `physical hit`

没有第二层和第三层，这个方向很容易退化成“两个 patch + 一堆实验”；有了这两层，才能把 `W1`、`W5` 以及后续可能补进来的 `W2` 放进同一篇系统论文里。

## 2. 为什么当前方向单独还不够

如果只看当前最直观的成果，外部 reviewer 很容易把工作理解成下面这种弱版本：

- `prompt_logprobs` 有一个 selective replay 修复
- backend mismatch 有一个 twin-pool route 修复
- 两个修复看起来都跟 prefix reuse 有关

这套叙事的问题是：

- 解释力不够：为什么这两类问题属于同一个系统现象，而不是两个独立 feature bug
- 可扩展性不够：第三个 mismatch 进来时，论文无法说明它该接到哪里
- 评测口径不够统一：一个机制看时间收益，一个机制看路由恢复，最后只剩 patch-by-patch 罗列
- 工作量不够“纵深”：读起来像工程 hardening，不像一套新的 runtime substrate

所以，当前方向不是错，而是**需要补出中层抽象和决策层，才能把工程修复抬升成系统方法**。

## 3. OSDI 级 thesis 应该怎么立

最稳的 thesis 不是“我们修了两个 bug”，而是：

`LLM serving runtimes lack a contract-and-realization layer that turns existing exact reuse conditions into actual physical hits under heterogeneous APIs and execution classes.`

中文可以稳定表述为：

- 现代 serving runtime 缺少一层“契约与兑现”基础设施
- 它导致系统明明已经具备 exact 或 logical-exact reuse 条件，却无法把这些既有计算稳定兑现成当前请求的 `physical hit`
- `TwinPathServe` 是这层基础设施的第一版系统实例，而不是论文唯一贡献

这一定义的价值在于，它把贡献从“修复两个 case”改写成“提出并验证一层此前缺失的 runtime capability”。

## 4. 如何把当前材料联成一条线

### 4.1 第一条线：问题证据线

这条线回答“问题真实存在，而且不是单一 benchmark 幻觉”。

当前已有：

- `W1` 在 `MT-Bench` 与 `LongBench` 上的公开 prevalence 证据
- `W5` 的 cross-backend logical false-negative 证据

接下来要补成：

- `W1/W2/W5` 的统一 problem surface
- 不同 workload family 上的 `logical hit -> physical hit -> false_negative_reason` 记账
- 不同 mismatch 家族在总 false-negative 里各占多少比例

只有做到这一步，论文才能回答“为什么这不是某个 API 的局部特例”。

### 4.2 第二条线：方法抽象线

这条线回答“为什么多个 mismatch 可以被放进同一个系统方法里”。

当前已经补到：

- `reuse contract`：判定 candidate miss 是否具备安全恢复资格
- `realization policy`：判定候选恢复在当前运行条件下值不值得做

这两层必须继续坚持，因为它们决定了论文是否仍然是 system thesis。更具体地说：

- `reuse contract` 解决的是语义统一问题
- `realization policy` 解决的是 runtime economics 问题

两者缺一不可。只有前者，论文像 taxonomy；只有后者，论文像 heuristic scheduler。

### 4.3 第三条线：runtime 落地线

这条线回答“抽象不是停在纸面，而是真的变成了 serving runtime 的控制面”。

这里当前已经有：

- `prompt_logprobs selective replay`
- `batch-invariant twin-pool route`

但 OSDI 级别不能只停在“有两个 operator”。更重要的是要证明：

- 它们共享同一条控制链
- 它们共享同一套 trace 语义
- 它们共享同一套 fallback discipline
- 它们可以被放进同一套 `realization` 指标里比较

也就是说，runtime 落地线的主角不应是单个 operator，而应是 `TwinPathServe` 这条控制面。

### 4.4 第四条线：实证闭环线

这条线回答“这不是漂亮概念，而是真的把 latent hit 兑现了出来，而且代价可解释”。

这一层建议统一用四组问题组织：

1. 问题规模：latent hit 到底有多少
2. 契约有效性：多少 miss 可以被 contract 正确归类
3. 兑现效率：`realization_efficiency` 有多高
4. 兑现代价：`bridge_cost_per_realized_token` 有多大

这样写的好处是，所有 bridge operator 都可以被纳入同一个评测框架，而不是各自找一套最有利的指标。

## 5. 什么东西最能补足“工作量不够”的质疑

如果 reviewer 说“方向有意思，但工作量不够大”，最有效的不是再塞进更多散点 case，而是补下面五类东西。

### 5.1 补一层完整的 `realization` 评测语言

这件事的意义非常大，因为它把论文从“能恢复”提升到“恢复得值不值”。

至少要统一报告：

- `counterfactual_hit_tokens`
- `recovered_physical_hit_tokens`
- `realization_efficiency`
- `bridge_cost_per_realized_token`
- fallback rate

这会让 `W1` 和 `W5` 真正进入同一张结果表。

### 5.2 补第一类 bridge 的 output-equivalence 审计

这是最容易被抓住的 correctness 缺口，也是最值得补的地方。

至少要覆盖：

- top-k token 一致性
- rank 一致性
- logprob 数值容差
- 长尾 fallback path 的接口一致性

这不是小修小补，而是把第一类 bridge 从“跑得快”抬升到“在可审计前提下跑得快”。

### 5.3 补第二类 bridge 的多 workload 与 overload 行为

当前 `MT-Bench W5` 已经是好的开始，但还不够撑满系统论文的 breadth。

更强的支撑应该包括：

- 多 benchmark / 多 workload family
- overload 下 route/fallback 稳定性
- 与 best ad hoc fix 的 baseline matrix

这能证明 `TwinPathServe` 不是只在一个 carefully prepared setting 里成立。

### 5.4 至少再补一个“不同类型但同语义”的 operator

这一步不是为了堆机制数量，而是为了证明 `reuse contract + realization policy` 真的有扩展性。

理想的补法不是再找一个和 `prompt_logprobs` 很像的 observation case，而是找一个性质不同、但仍能落在同一 contract 语言下的 case，例如：

- `carrier_incompatibility`
- `partial_prefill_incompatibility`
- pooling / embedding 侧的 logical-exact miss

只要能做到“同一 contract 语言 + 同一 realization 指标 + 同一 fallback discipline”，哪怕只是 P2 supporting 结果，也会显著增强论文的系统性。

### 5.5 补出 unified baseline matrix

很多系统论文最后输，不是因为 idea 不对，而是因为 baseline matrix 太薄。

这里最需要的是把三类对象放进一张统一矩阵：

- vanilla runtime
- per-feature ad hoc fix
- `TwinPathServe`

然后统一比较：

- false-negative reduction
- realization efficiency
- bridge cost
- TTFT / throughput
- fallback rate

这张矩阵一旦成立，论文就从“两个点状成功案例”升级成“统一 substrate 的经验优势”。

## 6. 对 `vLLM` 实现的联动要求

虽然当前阶段先只动文档，但如果目标是最终落到 `vLLM` 并支撑论文，那么从现在开始文档和实现必须按同一组接口推进。

最关键的是五类 runtime 接口位：

- `measurement surface`
  - 给出 request 级 `logical / physical / false_negative_reason`
- `reuse contract`
  - 给出 `contract_status`、`contract_violation_reason`、`bridgeability_class`
- `realization policy`
  - 给出 `realization_mode`、`bridge_cost_proxy`、`realization_efficiency`
- `bridge operator`
  - 给出 operator 触发条件、成功/失败与 fallback 原因
- `route/admission`
  - 给出 pool 选择、family binding 与 overload fallback

如果未来代码推进不围绕这五组接口留 trace，那么论文叙事会重新退化成“有结果，但说不清楚系统是怎么做决策的”。

## 7. 最推荐的推进顺序

在不再发散新 thesis 的前提下，最推荐的顺序是：

1. 把主文稿全部统一成四层主线
2. 把 `realization` 指标正式写入所有实验设计
3. 补第一类 bridge 的 output-equivalence 审计
4. 扩第二类 bridge 的 workload 与 baseline matrix
5. 选择一个与现有两类机制类型不同的 P2 operator
6. 最后再统一收敛摘要、引言、结论与主图

这条顺序的好处是：每一步都在给论文“增厚”，而不是靠改措辞把当前结果写得更像终稿。

## 8. 一句话判断

当前方向不是不够好，而是还处在“问题定义正确、第一批机制成立、但还没完全长成系统 thesis”的阶段。

真正能把它撑到 `CCF-A / OSDI` 量级的，不是继续堆零散 bridge，而是把它明确做成：

- 一套统一的问题测量面
- 一套统一的契约与兑现抽象
- 一条可验证的 runtime 控制链
- 一组统一的 realization 评测语言

只要这四件事联起来，`系统里明明存在 exact 或 logical-exact reuse 条件，但 runtime 没把它兑现成 physical hit` 就不再只是一个现象描述，而会变成一篇有方法层、有系统层、有评测层的完整论文主线。
