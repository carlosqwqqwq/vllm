# `FalseNegativeKV / TwinPathServe` 总控推进计划

更新日期：2026-04-16

## 1. 这份文档的作用

这份文档只负责一件事：

- 把当前分散在 `paper/`、`implementation/`、`experiments/` 与 `supporting/` 中的执行计划，收敛成一份真正可执行的总控计划。

它回答五个问题：

1. 当前已有计划是否足够，哪里还不够
2. 论文主张、系统实现、实验闭环之间是否已经对齐
3. 接下来固定应该先做什么、后做什么
4. 每个工作包交付什么 artifact，服务哪条论文 claim
5. 如果某条线做不出来，应该如何降级，而不是继续模糊推进

它不负责：

1. 替代已有细分规格文档
2. 重新发散 thesis
3. 虚构尚未完成的实验结果

## 2. 审核结论

## 2.1 现有计划需要完善

当前计划体系不是错误，而是**分层正确、总控不足**。

现状是：

- `implementation/` 里已经有 measurement、contract、policy、bridge、architecture 的细分规格
- `experiments/` 里已经有 evaluation matrix、public protocol、baseline matrix 设计
- `paper/supporting/` 里已经有 reviewer 问题矩阵、写作策略、submission readiness

但仍然缺一份文档，把下面三类关系真正钉死：

1. 哪条论文 claim 对应哪些代码模块
2. 哪个 reviewer 问题对应哪些实验 artifact
3. 哪些 work package 完成后，论文才允许升级 claim

换句话说，当前最主要的问题不是“缺文档”，而是：

- **缺一份跨层总计划，防止 paper、implementation、experiments 继续各自推进。**

## 2.2 现有计划里最需要修正的地方

### 2.2.1 `code_implementation_plan.md` 的顺序已经部分过时

这份文档的总体原则仍然对，但它的阶段顺序已经落后于当前代码状态。

文档中原本的顺序是：

1. 统一实验配置层
2. 公开 benchmark runner
3. `prompt_logprobs selective replay`
4. `TwinPathServe`
5. unified baseline matrix

而当前真实状态已经变成：

1. 统一 experiment config 已基本完成
2. `MT-Bench / LongBench / BFCL` runner 已存在
3. `prompt_logprobs selective replay` 已经实现并进入 public runner
4. `TwinPathServe` 已有 policy + twin-pool prototype + `W5` runner
5. 真正缺的是：
   - 方法层 runtime 化
   - ad hoc baseline 落地
   - unified matrix 自动化

因此，当前不应再把“先补更多 runner”视为主任务，而应转为：

- **先补方法层与 baseline 层，再补 benchmark 扩面。**

### 2.2.2 现有计划没有把论文 claim 分层冻结

当前文档里已经反复强调不要越过 `P1` 边界，但还没有把论文主张正式分成可升级的层级。

这会带来一个真实风险：

- 代码推进一次，正文就容易顺势把 claim 往上提一格；
- 但 reviewer 关注的是证据是否同步闭环，而不是功能是否“看起来更完整”。

因此，总计划必须先冻结 claim hierarchy，再安排实现顺序。

### 2.2.3 现有计划没有把 reviewer 问题映射到执行工作包

例如：

- “为什么 unified substrate 比 best ad hoc fix 更强？”
- “你们是否已经证明 output equivalence？”
- “W5 是不是 post-hoc labeling？”

这些问题在 supporting 文档里已经被识别，但还没有被映射成：

- 哪个代码模块要补
- 哪个实验要跑
- 哪章文稿可以升级

这也是这份总计划必须补上的部分。

## 3. 当前主线固定边界

这部分从现在开始视为锁死，不再继续改写。

### 3.1 active thesis

- 问题名：`FalseNegativeKV`
- 系统名：`TwinPathServe`
- 当前研究阶段：`measurement-first, system-closure-next`

### 3.2 P1 方法固定范围

P1 只允许包含两类 mismatch：

1. `observation mismatch`
2. `backend mismatch`

P1 只允许包含两类机制：

1. `prompt_logprobs selective replay`
2. batch-invariant twin-pool route

`W3 / W4` 当前只承担：

- prevalence measurement
- supporting generality

不进入 P1 主恢复面。

### 3.3 明确不做

当前总计划明确不引入：

- PIC / arbitrary-position reuse
- hidden cache / activation restoration
- distributed cache pool
- cross-model prefill sharing
- workflow-aware retention
- 第三条主机制
- 新 thesis 备胎并行推进

## 4. 当前状态审计

| 层 | 当前状态 | 结论 |
| --- | --- | --- |
| measurement P0 | 已完成第一版 request-level trace、lookup/replay/route 聚合 | 可继续复用，不再重写 |
| `prompt_logprobs selective replay` | 已实现，且已在 dedicated validation 与 public `W2` smoke 中跑通 | 第一机制已进入可发表方法雏形 |
| `TwinPath` policy / runtime | 已有 `TwinPathPolicy` 与 twin-pool prototype，`W5` runner 已存在 | 第二机制已有原型，但还不是 unified runtime |
| `reuse contract` / `realization policy` | 文档冻结，runtime 未完整具象化 | 这是当前最关键缺口之一 |
| public runner 体系 | `MT-Bench / LongBench / BFCL / W5` 已有；`MTEB` 缺失 | runner 扩面不是当前第一优先级 |
| ad hoc baselines | schema 里有字段，真实实现基本没有 | 这是当前第二个关键缺口 |
| unified baseline matrix | 设计文档已写好，自动化收口还没完成 | 这是当前第三个关键缺口 |
| output-equivalence audit | 只有计划文档，缺正式实验 artifact | 这是当前 reviewer 风险最高的 correctness 缺口 |
| paper main draft | 文稿结构已成型，claim 边界比之前稳很多 | 后续应由证据驱动升级，不再靠措辞推进 |

## 5. 现有文档之间的固定关系

从现在开始，下面这些文档的职责固定如下：

- [implementation/false_negative_instrumentation_spec.md](implementation/false_negative_instrumentation_spec.md)
  - 负责 P0 观测面，不再承担系统推进顺序
- [implementation/reuse_contract_spec.md](implementation/reuse_contract_spec.md)
  - 负责方法层抽象
- [implementation/realization_policy_spec.md](implementation/realization_policy_spec.md)
  - 负责方法层决策语义
- [implementation/code_implementation_plan.md](implementation/code_implementation_plan.md)
  - 负责代码分层原则，但不再单独充当总控计划
- [experiments/paper_evaluation_matrix.md](experiments/paper_evaluation_matrix.md)
  - 负责 thesis 级 go / no-go gate
- [experiments/unified_baseline_matrix_design_20260416.md](experiments/unified_baseline_matrix_design_20260416.md)
  - 负责 reviewer 导向的主表设计
- [paper/supporting/reviewer_question_matrix.md](paper/supporting/reviewer_question_matrix.md)
  - 负责问题清单与回答位置

这份总计划的职责是：

- **把上面这些文档真正串成一条执行链。**

## 6. 论文 claim 分层

从现在开始，论文主张固定分成五层。

| claim 层级 | 固定内容 | 升级条件 |
| --- | --- | --- |
| `C0` | `logical hit` 与 `physical hit` 脱节是可测现象 | P0 trace 与 prevalence 成立 |
| `C1` | `prompt_logprobs` observation mismatch 是强实例 | bridge I 恢复 + correctness 审计 |
| `C2` | backend mismatch 可通过 twin-pool route 恢复 | bridge II 在线或准在线证据闭环 |
| `C3` | unified substrate 优于 best ad hoc fix | unified baseline matrix 完成 |
| `C4` | `TwinPathServe` 是完整系统 thesis，而非 patch collection | `C1 + C2 + C3` 全部成立 |

任何文稿升级都必须遵守：

- 没完成 `C1`，就不能把第一机制写成“稳定安全兑现”
- 没完成 `C2`，就不能把 `TwinPathServe` 写成双机制都已实证闭环
- 没完成 `C3`，就不能把 unified 说成明显优于 separated fixes

## 6.1 `CCF-A` 系统论文的达标线

如果目标是把 `FalseNegativeKV / TwinPathServe` 包装成一篇真正的 `CCF-A` 系统论文，而不是 measurement paper 加两个 prototype，那么至少必须同时满足下面四层标准。

这里的“达标线”必须按下面口径理解：

- 它是**必要条件**，不是**充分条件**
- 全部满足后，只能说明这项工作已经具备进入 `CCF-A` 系统论文竞争区间的基本条件
- 它不能保证接收；最终结果仍取决于新颖性定位、实验完成度、写作质量以及同期投稿对手强度

### 6.1.1 问题层必须成立

论文必须证明：

1. `false-negative exact hit` 不是零星 feature bug，而是现代 serving workload 中可重复出现、可测量的系统性损失。
2. 这个问题的规模足以支撑系统工作量，而不是“偶发发生但优化收益有限”的边缘现象。
3. 问题边界清楚，能够和普通 prefix miss、普通 hash miss、family 未对齐、prompt 真变化清晰区分。

如果问题层做不到，整篇论文最多只能成立为：

- 一个 runtime measurement note
- 或一个 feature-specific optimization paper

### 6.1.2 抽象层必须不可替代

论文必须证明：

1. `reuse contract + realization policy + bridge operator` 不是命名包装，而是解释问题与指导实现都必需的统一抽象。
2. 至少两类 mismatch 需要共享同一套判定语义，才能自然落到同一个 runtime substrate 上。
3. 用 best ad hoc fix 分别修补时，会在表达力、配置一致性、评测统一性或运行时控制面上明显更差。

如果抽象层做不到，reviewer 会自然把论文降级理解为：

- 两个 patch 的并列汇报
- 或一个为 `vLLM` 特定 feature gap 发明的新术语系统

### 6.1.3 系统层必须闭环

论文必须证明：

1. `TwinPathServe` 不是两条互不相干的机制展示，而是一套统一的 runtime capability。
2. 两个主机制共享同一 control plane、同一 trace 语义、同一 baseline matrix 与同一主表口径。
3. 关键运行时决策不是后验解释，而是可在线或准在线判定、可落盘、可审计的系统行为。

如果系统层做不到，最合理的 paper package 只能是：

- problem + design sketch + limited prototype evidence

### 6.1.4 评估层必须达到系统论文强度

论文必须证明：

1. 不仅有恢复率或 phase wall time，还要有 `TTFT`、throughput、fallback、route 代价与 batching 副作用。
2. 不仅有单点正例，还要有统一 public matrix 与统一 baseline matrix。
3. 不仅展示“能优化”，还要展示“值得统一做、且优于分别修”。

如果评估层做不到，即使设计讲得通，也很难支撑 `CCF-A` 的系统定位。

### 6.1.5 新颖性与外部定位必须成立

论文必须证明：

1. `FalseNegativeKV` 相对近邻工作不是“换一种语言重述已有 patch 集合”，而是明确提出了更高层的问题边界与统一恢复对象。
2. `TwinPathServe` 相对近邻系统不是“把两个已知修补点放到一起”，而是提供了近邻工作未明确给出的统一 control plane、判定语义或比较基线。
3. 相关工作定位不只停留在文字排异，而要能落到可审查的比较矩阵上：
   - 我们解决什么
   - 近邻工作解决什么
   - 双方在问题边界、恢复对象、正确性边界与系统收益上分别覆盖什么

如果新颖性与外部定位做不到，整篇论文即使工程实现扎实，也仍然很容易被降级理解为：

- 一个有价值但不够新的系统化整理
- 或一个以 `vLLM` 为背景的局部优化工作

## 6.2 反分散约束

从现在开始，主 thesis 的包装必须严格遵守下面五条“单一主线”约束；任何违反都会让 paper 再次显得分散。

### 6.2.1 一个问题名

主文稿首页到实验章节只围绕一个问题名展开：

- `FalseNegativeKV`

不再在主叙事中平行推进新的问题命名。

### 6.2.2 一个系统名

主系统只保留一个名字：

- `TwinPathServe`

两个具体机制都必须被写成这个系统内的实例化，而不是各自独立的 feature 方案。

### 6.2.3 一个主抽象链

主抽象链固定为：

1. measurement
2. `reuse contract`
3. `realization policy`
4. `TwinPathServe`

论文任何章节都不能绕开这条链直接写“某个 patch 带来收益”。

### 6.2.4 一个主评测矩阵

主结果必须最终收口到同一套 public matrix 上：

- workload 侧固定为 `W1 / W2 / W5`
- baseline 侧固定为 `vanilla / status_quo / best_ad_hoc_fix / TwinPathServe_P1`
- 指标侧固定覆盖 `logical hit`、`physical hit`、`false_negative_ratio`、`TTFT`、throughput 与 runtime overhead

`W3 / W4` 只能作为 prevalence 或 supporting，不再各自独立长成主结果分支。

### 6.2.5 一张主表回答核心问题

最终论文必须至少有一张 reviewer 一眼就能读懂的主表，同时回答下面三个问题：

1. 这类损失是否普遍
2. unified 是否优于 ad hoc
3. 收益是否足以覆盖成本

如果这三个问题需要 reviewer 在三四张不同表之间自己拼接，paper 仍然会显得分散。

## 6.3 `CCF-A` 投稿前必须具备的 artifact 套件

在主文稿允许升级为“系统论文口径”之前，下面三类 artifact 必须全部具备。

### 6.3.1 方法与 runtime artifact

至少应包含：

1. `WP1` 定义的 runtime-visible 方法层字段
2. `bridge I` 的 sidecar boundedness / eviction / memory accounting
3. `bridge II` 的 route / bind / fallback 统一 trace
4. 非循环的 `backend mismatch` 识别证据

### 6.3.2 correctness artifact

至少应包含：

1. `prompt_logprobs selective replay` 的 output-equivalence audit
2. `logical hit` 与 `physical hit` 的 oracle 边界说明
3. `W5` route reason 不依赖 control-plane 自证的识别或 held-out 证据

### 6.3.3 评估 artifact

至少应包含：

1. `W1 / W2 / W5` 的统一 public matrix
2. `vanilla / ad hoc / unified` 同台基线结果
3. `p50/p95/p99 TTFT`
4. steady-state throughput
5. fallback rate / route 覆盖率 / batching side effect
6. 至少一组可重复运行结果，含重复次数或方差口径，而不是单次最优截图
7. 至少一组消融与敏感性结果，证明收益来自核心机制而不是参数偶然性

只要这三类 artifact 缺一，paper 都不应提前写成“完整系统已闭环”。

### 6.3.4 新颖性与外部效度 artifact

至少应包含：

1. 一张近邻相关工作比较矩阵，覆盖问题边界、系统对象、恢复对象与正确性边界
2. 至少两个公开模型配置上的结果，优选跨规模或跨模型族；如果只能做到单模型，题目、摘要与结论必须同步收紧到该边界
3. 一份明确的 external-validity note，解释哪些结论是 `vLLM` 特有，哪些结论更可能推广到通用 serving runtime

如果这组 artifact 缺失，paper 即使内部逻辑自洽，也仍然会在“外部效度不够”与“新颖性定位不足”上遭遇强攻击。

## 6.4 当前最关键的红线

下面这些情况如果继续存在，哪怕结果有正收益，论文仍然很难被当成成熟的 `CCF-A` 系统论文。

1. `W1` 与 `W5` 仍然使用两套不同 runner 语义、不同 trace 口径和不同表格语言。
2. `best_ad_hoc_fix` 一直没有真正实现，导致 unified 优势只能靠叙述成立。
3. `output equivalence` 仍停留在原则说明，没有正式 audit artifact。
4. `backend mismatch` 仍然主要依赖 post-hoc labeling 或 family-history 注入解释。
5. 主文稿仍需要 supporting 文档替它承担关键证据。
6. 摘要与引言的高度已经超过实验和实现实际上能支撑的高度。
7. 全文仍然只有单模型主结果，却把问题写成现代 LLM serving runtime 的一般性问题。
8. 结果主要来自单次跑分，没有重复性、方差或敏感性说明。
9. 相关工作比较仍然停留在文字排异，没有可核查的近邻工作比较矩阵。

## 7. 固定工作包

## 7.1 `WP1`：方法层 runtime 化

### 目标

把当前仅存在于文档中的方法层字段，变成 runtime 可输出、可聚合、可进主表的字段。

### 必做项

1. 在 runtime / trace / joined rows 中补齐最小方法层输出：
   - `selected_bridge_operator`
   - `realization_mode`
   - `fallback_mode`
2. 对第一版可行的 contract/policy 近似，补最小可见字段：
   - `bridgeability_class`
   - `contract_status` 或其保守代理
3. 在 analyzer / summary 中统一暴露这些字段 coverage

### 验收

1. `W1 / W2 / W5` 的 joined rows 都能输出上述字段
2. `unknown` 结果不能吞掉主 workload 大头
3. `06_实验评估.md` 可以直接引用这套字段，而不是只靠 supporting 解释

### 作用

这个工作包直接服务：

- `C1`
- `C2`
- `C3`

它是当前第一优先级。

## 7.2 `WP2`：bridge I hardening

### 目标

把 `prompt_logprobs selective replay` 从“已跑通原型”推进到“论文可交付机制”。

### 必做项

1. sidecar memory accounting
2. sidecar eviction / boundedness 语义
3. output-equivalence audit 实验脚本
4. `best_ad_hoc_fix` 的 prompt-logprobs 单点 baseline
5. `W1 + W2` 在统一 public runner 上的正式对照结果

### 验收

1. 有一份正式 correctness artifact，而不只是计划文档
2. `W1` 与 `W2` 至少一组 workload 上有稳定恢复收益
3. ad hoc prompt-logprobs fix 与 unified bridge I 能同台比较

### 作用

这个工作包直接决定：

- `C1` 能不能升级
- reviewer 对 output equivalence 的攻击能不能真正闭嘴

## 7.3 `WP3`：bridge II hardening

### 目标

把 `TwinPath` 从 benchmark/prototype 级双池，推进到真正能支撑主论文第二机制的系统原型。

### 必做项

1. 统一 route / fallback / bind 的 trace 语义
2. 把 `backend_mismatch` 的识别链条再收紧，避免循环论证
3. 明确 `family_binding`、`backend_cold_start`、`fallback` 的可解释边界
4. 落地 `best_ad_hoc_fix` 的 backend baseline
5. 在 `W5` 上形成 vanilla / ad hoc / unified 的同表比较

### 验收

1. `W5` 不再只有 prototype 片段，而是有 reviewer 可直接理解的统一表项
2. `route_reason` 不再只是 post-hoc 标签，而有独立可解释证据
3. backend ad hoc baseline 已可配置切换

### 作用

这个工作包直接决定：

- `C2` 能不能升级
- `TwinPathServe` 会不会继续被看成“设计图 + 一个 runner”

## 7.4 `WP4`：统一 baseline matrix 代码化

### 目标

把设计文档中的 baseline matrix 变成真实可运行矩阵，而不是人工拼表。

### 必做项

1. 让同一 config 层可切换：
   - `vanilla_vllm_v1`
   - `status_quo_skip`
   - `best_ad_hoc_fix`
   - `TwinPathServe_P1`
2. 统一 summary 输出字段
3. 固定 `W1 / W2 / W5` 的 recovery 主表格式

### 验收

1. 同一 runner 换 config 即可切 baseline
2. 同一 analyzer 直接输出主表 A/B/C 所需字段
3. 不再依赖人工解释“这一行数字属于哪类 baseline”

### 作用

这个工作包直接决定：

- `C3` 能不能成立

## 7.5 `WP5`：公开 benchmark 闭环

### 目标

在方法层与 baseline 层收口后，再把公开 benchmark 结果真正跑成主结果，而不是继续只堆 measurement。

### 主恢复 workload

- `W1`
- `W2`
- `W5`

### prevalence / supporting workload

- `W3`
- `W4`

### 验收

1. `W1 / W2 / W5` 进入统一 recovery matrix
2. `W3 / W4` 若未完成恢复，则至少完成 prevalence，并在文稿中明确降级角色
3. 不能再出现“某个 workload 只有日志，没有主表定位”的状态

## 7.6 `WP6`：paper closure

### 目标

保证文稿升级严格跟随证据升级，而不是反过来。

### 必做项

1. 只有在 `WP2/3/4/5` 完成后，才允许升级：
   - 摘要
   - 引言
   - 06_实验评估
   - 08_讨论与结论
2. reviewer question matrix 与主文稿同步更新
3. submission readiness 文档重新审计

### 验收

1. 每一条 stronger claim 都能指到具体 artifact
2. supporting 不再替正文承担关键主证据
3. 文稿与实验结果之间没有“先写强、后补证据”的反向依赖

## 8. 固定执行顺序

从现在开始，固定执行顺序改为：

1. `WP1` 方法层 runtime 化
2. `WP2` bridge I hardening
3. `WP3` bridge II hardening
4. `WP4` 统一 baseline matrix 代码化
5. `WP5` 公开 benchmark 闭环
6. `WP6` paper closure

固定禁止的错误顺序：

1. 先继续扩更多 benchmark，再回来补方法层
2. 先升级摘要 / 引言 claim，再补实验
3. 在没有 ad hoc baseline 的前提下，提前写 unified 优势

## 9. 并行策略

虽然顺序固定，但允许下面两类并行：

1. `WP2` 与 `WP3` 可以在 `WP1` 完成后并行推进
2. GPU 跑正式实验时，可以并行推进：
   - output-equivalence audit 脚本
   - baseline config 实现
   - 文稿表格模板

不允许的并行方式：

1. 同时再开新 thesis
2. 在 `WP4` 未落地前，同时扩一堆新 benchmark runner

## 10. gate 与降级规则

### 10.1 thesis 继续推进的 gate

只有在下面三条里至少满足两条时，当前主 thesis 才继续维持完整系统方向：

1. `W1 / W2` 中至少一条 workload family 达到显著恢复收益
2. `W5` 能形成 reviewer 可接受的 backend mismatch recovery 证据
3. unified baseline matrix 显示 `TwinPathServe_P1` 不劣于 best ad hoc fix

### 10.2 降级规则

若最终出现下面任一情况，应立即降级，而不是继续写强 claim：

1. 只能稳定证明 bridge I，有效 bridge II 迟迟闭环不了
   - 降级为 measurement + one bridge paper
2. unified substrate 在主 workload 上并不优于 best ad hoc fix
   - 降级为 problem + patch line
3. `W3 / W4` 无法推进到可用 supporting
   - 保留一般性边界，但收紧题目与摘要口径

## 11. 当前最优先的具体动作

如果把下一轮工作压成最小可执行列表，固定应当是：

1. 补 `WP1` 的 runtime-visible 方法层字段
2. 补 bridge I 的 output-equivalence audit
3. 补 `best_ad_hoc_fix` 的 prompt-logprobs baseline
4. 补 `best_ad_hoc_fix` 的 backend baseline
5. 把 `W1 / W2 / W5` 收进同一 baseline matrix

这五步完成前，不再把“继续扩 benchmark”视为主任务。

## 12. 一句最终判断

如果只保留一句最该执行的话，这句应该固定为：

> 当前最需要完善的不是再写更多计划，而是把已有计划真正收敛成一条总执行链：先把方法层和 baseline 层补齐，再用统一矩阵收口 `W1 / W2 / W5`，最后再升级论文 claim。
