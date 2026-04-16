# 主文稿一致性审计

更新日期：2026-04-15

## 1. 目的

这份文档只做一件事：

- 记录 `00-08` 主文稿当前已经统一下来的写法、口径和证据边界

它不负责：

- 重新发散 thesis
- 代替主文稿写正文
- 用“措辞优化”掩盖证据还不存在的问题

这份审计的用途，是让后续继续改论文的人先知道哪些说法已经锁定，哪些地方仍然只能保守写，避免不同章节再次把 claim 强度写乱。

## 2. 已锁定的总口径

### 2.1 论文定位

当前最稳的定位是：

- `measurement-backed problem paper`
- `design-forward system thesis`
- `contract-and-realization-forward system thesis`

不能写成：

- “完整系统收益已经在公开 benchmark 上充分闭环”
- “所有 prefix miss 都可以通过统一 substrate 恢复”

### 2.2 主问题定义

主问题统一写作：

- `false-negative exact hit`

核心定义统一为：

- 在 reference execution condition 下，exact 或 logical-exact reuse 已经成立
- 但当前 request 没有把这份既有计算兑现为自己的 `physical hit`

不能退化成：

- `hash` 没命中
- cache key 没打通
- 一般意义上的 prefix miss

### 2.3 安全性主张

安全性统一写成：

- `soundness`，不是 `completeness`

允许写：

- `SafeReuseContract` 给出安全恢复的充分条件
- 若条件不成立、不可验证或成本超预算，系统必须 fallback

不允许写：

- “已有 state 原则上都应被复用”
- “只要命中 hash 就应直接复用”
- “本文已经证明所有 bridge 的完整输出等价性”

### 2.5 方法主线

主文稿当前统一按四层主线组织：

- `measurement surface`
- `reuse contract`
- `realization policy`
- `TwinPathServe`

允许写：

- `reuse contract` 负责识别哪些 miss 在语义上具备被恢复资格
- `realization policy` 负责判断这些候选 miss 在当前预算与队列条件下是否值得兑现
- 第五章与第六章共享 `reuse oracle -> contract gate -> realization policy -> bridge / route -> fallback` 控制链

不允许写：

- 直接把“识别到 mismatch”写成“应该立刻恢复”
- 跳过 `realization policy`，把所有恢复动作都写成无成本、无预算约束的必然步骤

### 2.4 `W1` 与 `W5` 的定位

`W1 prompt_logprobs` 统一写成：

- 已有 request-level 强测量证据
- 已有 runtime prototype 恢复结果
- 已有 `32-family` formal validation
- 已有必要条件消融

`W5 backend mismatch` 统一写成：

- 在 vanilla runtime 侧是 `cross-backend logical false-negative` 证据
- 在 `TwinPathServe` 原型侧已有 `MT-Bench W5` 公开 workload 的恢复结果
- 当前还不能写成“在线原生 `backend_mismatch` 分类已经闭环”

## 3. 统计口径统一

主文稿现在统一使用两层统计粒度：

- 单 request / 单 family 平均值
- 配对 probe 聚合总量

具体约定如下：

- `1088`、`148`、`64` 默认表示单 request 或单 family 平均命中
- `8704`、`8576`、`592`、`256` 默认表示配对 probe 聚合后的总量

这一约定的原因：

- 平均值更适合说明“单个 family 在 reference path 上能拿到多少命中”
- 聚合值更适合与第三章的 `logical_hit_tokens` / `physical_hit_tokens` 记账直接对齐

后续写图表时必须避免：

- 在同一段落里混用平均值和总量，却不说明统计粒度
- 先说 `1088 -> 0`，下一句又说 `8704 -> 8576`，但不解释前者是平均、后者是聚合

## 4. 章节级一致性结论

### 4.1 `00_题目与摘要.md`

当前状态：

- 题目与摘要已经与后文 prototype 结果对齐
- 没有停留在“仅提出设计”的弱强度
- 仍然保留了“不是完整系统收益闭环”的边界

后续注意：

- 摘要可以继续压缩，但不能新增还未实证的主张
- 不能把 `W5` 写成 vanilla runtime 在线分类已经完成

### 4.2 `01_引言.md`

当前状态：

- 已明确研究路径是“先测量、后设计、再以最小原型验证恢复路径”
- 三点贡献与第三章、第六章当前证据边界一致

后续注意：

- 引言里提到 prototype 时，要始终保持“第一版 bridge 证据”这一强度
- 不要在引言里抢先写多 benchmark 主实验结论

### 4.3 `02_背景与动机.md`

当前状态：

- 已经完成与引言的功能分工
- 四个操作性条件把问题边界讲清楚了
- `1098 -> 1088` 的 block 粒度说明已经补足

后续注意：

- 第二章不能再次承担“重新证明 novelty”的任务
- 不要把 mixed-API serving 的复杂性铺得比引言更长

### 4.4 `03_FalseNegativeExactHit的测量.md`

当前状态：

- `RQ1` 和 `RQ2` 结构清楚
- measurement-first 的职责明确
- `W1` 与 `W5` 的证据边界已经分开写清

后续注意：

- 第三章只负责“问题存在且可测”
- 不要把恢复结果提前写进第三章

### 4.5 `04_契约感知复用抽象.md`

当前状态：

- 抽象层和测量层已经接上
- `SafeReuseContract` 的五个条件是当前安全性论证的核心锚点
- 已补上 `reuse contract` 之后还需要 `realization policy` 的衔接说明

后续注意：

- 第四章不能把理论命题写成“完整安全性已完全证毕”
- 需要始终保持“充分条件 + fallback”的写法

### 4.6 `05_TwinPathServe系统设计.md`

当前状态：

- 系统设计已经围绕两类已测量 mismatch 展开
- `route key`、family binding、`contract gate`、两类 bridge 的关系清楚
- 今日已把控制链统一改成 `reuse oracle -> contract gate -> realization policy -> bridge operator -> route/admission`

后续注意：

- 第五章可以继续增强执行细节，但不能偷偷扩面到更多 mismatch
- 所有新增设计都必须回到 `SafeReuseContract`

### 4.7 `06_实验评估.md`

当前状态：

- 已明确与第三章分工
- 已把 `W1` 的 formal validation、机制消融、`W5` 的 `MT-Bench W5` formal 结果写进来
- 今日已补齐统计粒度说明，避免 `1088`、`8704`、`592` 等口径混乱
- 今日已补齐 output-equivalence 仍未完整闭环的边界说明
- 今日已把 `realization_efficiency` 与 `bridge_cost` proxy 明确纳入评测语言

后续注意：

- 第六章最容易被 reviewer 抓住的点是“你证明了收益，但是否证明了输出不变”
- 第六章第二个容易被抓的点是“vanilla logical evidence”和“twin-path public recovery evidence”是否被混写
- 因此后续若补实验，应优先补：
  - top-k token / rank / logprob 数值对齐
  - `W1` full-method public runner
  - 长尾 fallback path 的接口一致性
  - overload 下 route/fallback 的稳定性

### 4.8 `07_相关工作.md`

当前状态：

- 已按“默认 reuse 假设”分层组织，而不是按组件列名
- 与 runtime、routing、shared-prefix、PIC、state-medium、workflow continuity 的边界都已明确

后续注意：

- 相关工作可以继续加 citation，但不要破坏当前问题边界的层级结构
- 本章的任务是“说明本文不属于哪条近邻路线”，不是再讲一次系统设计

### 4.9 `08_讨论与结论.md`

当前状态：

- 已将一般性、安全边界、可证伪性和 full-thesis 缺口分开讨论
- 今日已补写“条件化 soundness”口径
- 今日已把 output-equivalence audit matrix 明确列为仍需补齐的证据

后续注意：

- 结论里不能把 supporting evidence 写成跨框架主实验
- 不能把“条件化 soundness”写成“完整正确性已经完全证明”

## 5. 当前仍未完全闭环的问题

下面这些不是“再润色一下文字”能解决的问题，只能靠后续实验和材料补齐。

### 5.1 输出等价性审计还不完整

目前已经有：

- `prompt_logprobs` 接口级 smoke
- 必要条件消融
- 非零 hit 与显著时间收益
- `32-family` formal validation

目前还缺：

- top-k token 一致性
- rank 一致性
- logprob 数值容差报告
- 长尾 fallback path 是否保持接口与输出语义一致

### 5.2 第二类 bridge 还缺更完整的大样本评估

目前已经有：

- `W5` 的强 logical evidence
- twin-pool formal validation
- `MT-Bench W5` 正式公开 workload 结果

目前还缺：

- 多 benchmark
- 多 workload family
- overload / fallback 行为
- 与 best ad hoc fix 的完整矩阵比较

### 5.3 统一 substrate 的“全面经验优势”还不能强写

目前可以稳写：

- 统一 substrate 提供了更强的解释力
- 两类 mismatch 已能放入同一 contract 语言并得到第一版 formal 恢复证据

目前不能强写：

- 在所有代表性 workload 上统一 substrate 都显著优于最佳 scattered fix

## 6. 后续修改时必须坚持的写作纪律

1. 任何章节新增 claim 前，先问它有没有对应的 trace、实验或源码证据。
2. 若一条结论只是 supporting-level 迹象，就只能写 supporting-level，不得抬升为主实验结论。
3. 若数字来自聚合结果，必须说明聚合粒度；若数字来自平均值，必须说明平均对象。
4. 一旦涉及“是否安全”“是否影响输出”，默认回到 `soundness not completeness` 的写法。
5. 任何新增 bridge 或 mismatch，若没有进入第三章测量与第六章评估闭环，就不要写进主线贡献。

## 7. 最推荐的下一步

如果继续沿主文稿 hardening 往前走，优先级建议如下：

1. 补第一类 bridge 的 output-equivalence 审计设计与结果
2. 扩第二类 bridge 的公开 workload 与 baseline matrix
3. 再考虑是否需要收紧摘要、引言和结论的措辞

当前阶段最不值得做的事是：

- 再发散新的 mismatch 家族
- 再改大标题
- 再写一版与现有证据强度不匹配的“终稿式结论”
