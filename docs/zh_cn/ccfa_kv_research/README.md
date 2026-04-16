# 面向 CCF-A 的 vLLM KV Cache 研究专题

这组文档面向一个很具体的目标：

- 研究主题：KV cache 优化
- 目标平台：`vLLM`
- 论文类型：系统类 CCF-A
- 评价目标：轻量级、普适性、高吞吐、低延迟

这不是普通源码笔记，而是一条围绕 `vLLM + KV cache + 系统类顶会 thesis` 的持续收敛轨迹。当前目录状态已经收敛成：

- **当前主线**：`14_beyond_current_ideas/`
  - active thesis 是 `FalseNegativeKV`
  - 当前阶段是 `measurement-first`
  - 第一版系统固定为 `TwinPathServe`
- **旧主线归档**：`12_combined_main_thesis/`
  - 归档的是上一轮主 thesis `ExactPrefixKV`
  - 保留为历史参考，不再继续扩写为当前主线

## 文档结构

- [vLLM 现状](01_vllm_status/README.md)：当前架构、能力边界、关键短板。
- [GitHub 社区进展](02_github_progress/README.md)：release、PR、issue 所反映的真实趋势。
- [论文 Idea 与创新点](03_paper_ideas/README.md)：第一轮发散出的候选方向。
- [细化方案与对比](04_refined_ideas/README.md)：第一轮细化后的主线与横向比较。
- [调研过程与结果沉淀](05_research_archive/README.md)：调研方法、证据链、筛选标准与判断过程。
- [第二轮重构后的新候选](06_reframed_ideas/README.md)：换问题定义后的三条新 thesis。
- [第三轮收敛后的最终候选](07_final_candidates/README.md)：更轻量、更贴近 `vLLM` 的三条主线。
- [第四轮撞车感知后的新方向](08_collision_aware_new_ideas/README.md)：`ShapeKV`、`ElasticPageKV`、`ControlPlaneKV`。
- [第五轮持续迭代后的新方向](09_iterative_new_ideas/README.md)：`ResidualModeKV`、`TypedPrefixKV`、`PromptABIKV`。
- [第六轮按学术问题重开的新方向](10_academic_restart_ideas/README.md)：`PolyStateKV`、`TxnStateKV`、`CapsuleKV`。
- [第七轮从能力缺口反推的新候选](11_capability_gap_ideas/README.md)：`ObservationKV`、`InvariantKV`、`EligibilityKV`。
- [旧主线归档：ExactPrefixKV](12_combined_main_thesis/README.md)：上一轮组合 thesis、measurement plan 与 storyline。
- [第八轮 fresh restart 候选](13_fresh_restart_ideas/README.md)：`WritePathKV`、`ObservationSidecarKV`、`DecisionSidecarKV`。
- [当前主线：FalseNegativeKV](14_beyond_current_ideas/README.md)：measurement-first 执行文档、`TwinPathServe` 最小架构与 paper-ready 叙事。

## 当前推荐阅读顺序

如果你只关心当前 active thesis，按下面顺序读：

1. [14_beyond_current_ideas/README.md](14_beyond_current_ideas/README.md)
2. [false_negative_instrumentation_spec.md](14_beyond_current_ideas/implementation/false_negative_instrumentation_spec.md)
3. [paper_evaluation_matrix.md](14_beyond_current_ideas/experiments/paper_evaluation_matrix.md)
4. [twin_path_serve_minimal_architecture.md](14_beyond_current_ideas/implementation/twin_path_serve_minimal_architecture.md)
5. [paper_storyline_fnkv.md](14_beyond_current_ideas/paper/supporting/paper_storyline_fnkv.md)
6. [paper/README.md](14_beyond_current_ideas/paper/README.md)

如果你需要理解为什么旧主线被归档，再补读：

1. [12_combined_main_thesis/README.md](12_combined_main_thesis/README.md)
2. [measurement_plan.md](12_combined_main_thesis/measurement_plan.md)
3. [paper_storyline.md](12_combined_main_thesis/paper_storyline.md)

如果你需要回溯这条研究线是怎么一步步排重和收敛出来的，再向前补读：

1. `11_capability_gap_ideas`
2. `13_fresh_restart_ideas`
3. `05_research_archive`
4. `02_github_progress`

## Idea 演进脉络

这套材料记录的不是“很多互相独立的点子”，而是同一研究目标下的多轮否证与收敛：

- `03/04`
  - 第一轮大方向发散与细化。
- `06/07`
  - 在明显撞车后重新定义问题，再做一次压缩。
- `08/09/10/11`
  - 持续从工程切口退回学术命题，再从 `vLLM` 能力缺口反推新问题。
- `12`
  - 旧主线 `ExactPrefixKV`，把 `PromptABIKV`、`TypedPrefixKV`、`ResidualModeKV` 组合成完整 thesis。
- `13`
  - 放弃旧主线后重新 fresh restart，得到 `WritePathKV`、`ObservationSidecarKV`、`DecisionSidecarKV`。
- `14`
  - 最终收敛到当前 active thesis：
    - 问题名：`FalseNegativeKV`
    - 系统名：`TwinPathServe`
    - 阶段：`measurement-first`

## 当前总判断

截至 `2026-04-15`，当前更值得继续推进的 thesis 不是继续发散新方向，而是把下面这件事做厚：

> 在现代 LLM serving 中，许多请求明明是 `logical hit`，却没有变成 `physical hit`；
> 这类 `false-negative exact hit` 不是单点 bug，而可能是一类统一的 runtime pathology。

因此，当前主线的重点已经固定为：

1. 先证明问题规模足够大
2. 再证明统一 substrate 比 scattered fixes 更强
3. 再证明 `TwinPathServe` 能在两类 mismatch 上恢复收益
4. 最后才把结果包装成论文

## 维护约定

这份总 README 现在只负责四件事：

1. 标出当前主线和旧主线的目录状态
2. 提供当前推荐阅读路径
3. 概括 idea 的演进脉络
4. 给出当前最值得继续执行的 thesis 入口

后续维护规则固定为：

- `14_beyond_current_ideas` 是当前唯一主线
- `12_combined_main_thesis` 是旧主线归档
- 不再新增新的“高层 idea 比较”文档
- 后续新增内容只允许是执行型文档，除非 measurement gate 明确失败
