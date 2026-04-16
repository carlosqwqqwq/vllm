# Reviewer 问题矩阵

更新日期：2026-04-16

## 1. 这份文档的作用

这份文档只负责一件事：

- 把当前已经预判到的 reviewer 高概率问题，映射到主文稿与 supporting 文档中的对应回答位置

它不负责：

- 给出完整 rebuttal 文本
- 替代正文
- 展开新的 thesis

这份矩阵的固定用途是：

1. 检查“哪些问题已经被正文覆盖”
2. 检查“哪些问题目前仍主要由 supporting 承担”
3. 指导下一轮文稿 hardening 的优先级

## 2. 当前判断

当前状态已经从“回答散落在 supporting 里”推进到：

- 高频 reviewer 问题在主文稿中都已经有明确主回答位置
- supporting 文档负责提供 case-level 证据、源码定位和可复用回答模板
- 剩余风险的核心已经不再是口径缺失，而是少数结论仍需后续实验闭环

因此，从现在开始，不再继续发散新问题，而是围绕下面这张矩阵守住 claim 边界，并把剩余工作集中到证据补齐上。

## 3. 问题矩阵

| reviewer 问题 | 当前主回答位置 | supporting 位置 | 当前状态 | 下一步动作 |
| --- | --- | --- | --- | --- |
| 这是不是两个工程 patch 的拼接？ | [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [paper_storyline_fnkv.md](paper_storyline_fnkv.md) | 主文稿已基本覆盖 | 继续保持统一 substrate 口径 |
| 这是不是 hash / cache key 没打通？ | [00_题目与摘要.md](../manuscript_cn/00_题目与摘要.md)、[01_引言.md](../manuscript_cn/01_引言.md)、[04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md) | [vllm_isolation_cases_and_bridgeability.md](vllm_isolation_cases_and_bridgeability.md) | 主文稿已覆盖，但 supporting 更强 | 后续避免回退到“已有缓存没利用”措辞 |
| 如果 `vLLM` 专门隔离了这些路径，肯定有原因，你们为什么还要复用？ | [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)、[05_TwinPathServe系统设计.md](../manuscript_cn/05_TwinPathServe系统设计.md) | [vllm_isolation_cases_and_bridgeability.md](vllm_isolation_cases_and_bridgeability.md) | 主文稿已初步覆盖 | 继续强调“不是取消保护，而是只 bridge 可验证 case” |
| 你们的复用是不是不安全，会不会影响输出？ | [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)、[05_TwinPathServe系统设计.md](../manuscript_cn/05_TwinPathServe系统设计.md) | [safe_reuse_soundness_note.md](safe_reuse_soundness_note.md) | 已通过本轮 hardening 进入主文稿 | 后续实验必须对齐 `soundness` 口径 |
| 你们的理论能证明安全吗？ | [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)、[05_TwinPathServe系统设计.md](../manuscript_cn/05_TwinPathServe系统设计.md) | [safe_reuse_soundness_note.md](safe_reuse_soundness_note.md) | 主文稿已加入 `SafeReuseContract` | 后续可在 rebuttal 中用 `soundness not completeness` 固定回答 |
| 你们是否已经证明 `prompt_logprobs selective replay` 不改变用户可见输出？ | [06_实验评估.md](../manuscript_cn/06_实验评估.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [prompt_logprobs_output_equivalence_audit_plan_20260416.md](prompt_logprobs_output_equivalence_audit_plan_20260416.md) | 主文稿已承认证据缺口，supporting 已补执行方案 | 下一步按审计方案补齐 top-k / rank / logprob 与 fallback 等价性 |
| 这是不是 `vLLM` 独有的工程问题，`SGLang` 有没有？ | [07_相关工作.md](../manuscript_cn/07_相关工作.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [sglang_capability_boundary_cases.md](sglang_capability_boundary_cases.md) | 本轮已补一般性回答 | 若 reviewer 深追，再用 supporting 展开 case-level 证据 |
| 为什么不分别修 `prompt_logprobs` 和 backend mismatch？ | [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [writing_strategy.md](writing_strategy.md) | 主文稿已覆盖 | 无需再扩写，保持收敛 |
| 为什么 unified substrate 比 best ad hoc fix 更强，而不是只是更复杂？ | [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)、[06_实验评估.md](../manuscript_cn/06_实验评估.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [../../experiments/unified_baseline_matrix_design_20260416.md](../../experiments/unified_baseline_matrix_design_20260416.md) | 主文稿已有方法主张，但统一对照矩阵仍需实验补齐 | 按统一 baseline matrix 固定 `best_ad_hoc_fix` 与 `TwinPathServe_P1` 的同表比较 |
| 当前 claim 会不会超出证据边界？ | [00_题目与摘要.md](../manuscript_cn/00_题目与摘要.md)、[01_引言.md](../manuscript_cn/01_引言.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [writing_strategy.md](writing_strategy.md) | 本轮已收紧 | 后续所有新增文字必须遵守 `P1` 边界 |
| 你们到底 claim 了哪些 case 是 `P1`，哪些不是？ | [05_TwinPathServe系统设计.md](../manuscript_cn/05_TwinPathServe系统设计.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [vllm_isolation_cases_and_bridgeability.md](vllm_isolation_cases_and_bridgeability.md) | 主文稿已基本清楚 | supporting 继续承担 case inventory 角色 |
| `W5` 到底是在线结论还是 cross-backend logical evidence？ | [03_FalseNegativeExactHit的测量.md](../manuscript_cn/03_FalseNegativeExactHit的测量.md)、[06_实验评估.md](../manuscript_cn/06_实验评估.md)、[08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md) | [paper_storyline_fnkv.md](paper_storyline_fnkv.md) | 主文稿已清楚 | 无需升级 claim |

## 4. 当前最需要盯紧的风险

虽然本轮已经把几个大问题推进正文，但后续仍要警惕下面三类风险：

1. 摘要、引言和结论的措辞重新滑回“已有缓存没用上”的弱表述
2. 后续系统实现描述把 `soundness` 误写成“所有 logical hit 都能恢复”
3. 在没有额外实验前，把 `SGLang` 对应现象写成“已证明完全同构”

## 5. 下一步维护规则

从现在开始，任何新增段落都要先经过下面两个检查：

1. 它在回答哪一个 reviewer 问题？
2. 它有没有越过当前矩阵中已经锁定的 claim 边界？

如果这两个问题回答不清，就不要把新内容写进主文稿。

## 6. 仍然不能靠措辞替代的证据缺口

下面三类问题，当前文稿已经能诚实回答边界，但还不能靠“多写几段”把证据补出来：

1. `W1` 的恢复收益与端到端 `TTFT` 改善，仍需要 `prompt_logprobs selective replay` prototype 与实验结果闭环
2. `W5` 当前仍主要是 cross-backend logical evidence，仍需要在线 `route metadata` 与 `backend_mismatch` 原生归因补齐
3. `SGLang` 目前提供的是 supporting-level 一般性证据，而不是跨框架主实验

这三项如果没有新增实验，主文稿就必须继续保持当前口径，不能写成“系统收益已经完整证明”。
