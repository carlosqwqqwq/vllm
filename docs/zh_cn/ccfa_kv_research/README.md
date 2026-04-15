# 面向 CCF-A 的 vLLM KV Cache 研究专题

这组文档面向一个很具体的目标：

- 研究主题：KV cache 优化
- 目标平台：`vLLM`
- 论文类型：系统类 CCF-A
- 评价目标：轻量级、普适性、高吞吐、低延迟

这不是普通的源码阅读笔记，也不是一次性的 brainstorming 记录，而是一套围绕 `vLLM + KV cache + 系统类 CCF-A` 持续收敛的研究总览。核心问题只有三个：

1. 当前 `vLLM` 在 KV cache 上已经做到什么程度？
2. GitHub 社区最近一年在往哪个方向演进？
3. 真正值得写成系统论文的创新点应该落在哪里？

## 文档结构

- [vLLM 现状](01_vllm_status/README.md)：当前架构、能力边界、关键短板。
- [GitHub 社区进展](02_github_progress/README.md)：release、PR、issue 所反映的真实趋势。
- [论文 Idea 与创新点](03_paper_ideas/README.md)：第一轮发散出的候选方向、首选主线、代码改造点和实验设计。
- [细化方案与对比](04_refined_ideas/README.md)：第一轮细化后的 5 条主线、目录组织方式与横向比较。
- [调研过程与结果沉淀](05_research_archive/README.md)：调研方法、证据链、筛选标准与判断过程。
- [第二轮重构后的新候选](06_reframed_ideas/README.md)：在排除上一轮明显重合方案后，重新收敛出的三个新 thesis。
- [第三轮收敛后的最终候选](07_final_candidates/README.md)：继续排除重合较高方向后，保留下来的 3 条最新主线与总对比。
- [第四轮撞车感知后的新方向](08_collision_aware_new_ideas/README.md)：在确认多条旧思路与已有工作重合后，重新收敛出的 `ShapeKV`、`ElasticPageKV`、`ControlPlaneKV` 三条方向。
- [第五轮持续迭代后的新方向](09_iterative_new_ideas/README.md)：继续删除 eviction/look-ahead 等拥挤方向后，保留下来的 `ResidualModeKV`、`TypedPrefixKV`、`PromptABIKV` 三个新候选。
- [第六轮按学术问题重开的新方向](10_academic_restart_ideas/README.md)：明确排除上一轮偏工程切口候选后，从“可复用状态的表示、生命周期与组合语义”三个学术母题重新收敛出的 `PolyStateKV`、`TxnStateKV`、`CapsuleKV`。
- [第七轮从能力缺口反推的新候选](11_capability_gap_ideas/README.md)：不再从大词发散，而是从 `vLLM` 当前 prefix cache 进不去的能力面反推出来的 `ObservationKV`、`InvariantKV`、`EligibilityKV` 三条新方向。

## 建议阅读顺序

如果你是第一次接手这个方向，建议按下面顺序阅读：

1. `01_vllm_status`
2. `02_github_progress`
3. `03_paper_ideas`
4. `04_refined_ideas`
5. `06_reframed_ideas`
6. `07_final_candidates`
7. `08_collision_aware_new_ideas`
8. `09_iterative_new_ideas`
9. `10_academic_restart_ideas`
10. `11_capability_gap_ideas`

如果你已经大致知道问题背景，只想快速抓住“当前最值得继续验证什么”，建议直接读：

1. `08_collision_aware_new_ideas`
2. `09_iterative_new_ideas`
3. `10_academic_restart_ideas`
4. `07_final_candidates`
5. `11_capability_gap_ideas`

如果你需要核对“这些判断为什么成立”，或者准备继续扩展这条研究线，再补读：

1. `05_research_archive`
2. `02_github_progress`
3. 对应 idea 子目录中的可行性评估 / 重合度分析文档

## Idea 演进脉络

这套材料里最容易让人迷路的地方，不是文档多，而是 idea 在多轮调研中持续变化。更准确地说，这几层目录记录的是一条逐步收敛的演进链：

- `03_paper_ideas`
  - 第一轮偏“大方向发散”，核心代表是 `TierKV`、`HeteroKV`、`RouteKV`。
- `04_refined_ideas`
  - 在第一轮基础上进入细化与横向比较，形成 `TierKV`、`AffinityKV`、`QoSKV`、`SegmentKV`、`HeteroKV` 五条主线。
- `06_reframed_ideas`
  - 在确认多级、路由、热度、普通 prefix 调度等方向周围已有大量强工作后，重新换问题定义，收敛为 `SessionKV`、`LineageKV`、`FrontierKV`。
- `07_final_candidates`
  - 继续压缩与重排，转向更轻量、更贴近 `vLLM` 主线的 `FastPathKV`、`ZeroLagKV`、`PublicStateKV`。
- `08_collision_aware_new_ideas`
  - 在进一步做撞车感知后，把主线收束到 `ShapeKV`、`ElasticPageKV`、`ControlPlaneKV`，并明确三者之间的主次关系。
- `09_iterative_new_ideas`
  - 在继续排除 queue-aware eviction、short-prefill clustering、security-only APC 等方向后，把候选重新压缩成 `ResidualModeKV`、`TypedPrefixKV`、`PromptABIKV`。
- `10_academic_restart_ideas`
  - 在进一步反思“工程痛点不等于论文命题”之后，把上一轮三个工程味较强的候选显式移出候选集，并从 `state representation`、`state lifecycle`、`state composition semantics` 三个学术母题重新提出 `PolyStateKV`、`TxnStateKV`、`CapsuleKV`。
- `11_capability_gap_ideas`
  - 在继续确认高抽象状态类 thesis 也开始靠近近邻工作后，转而从 `vLLM` 当前 prefix cache 被保守关闭或跳过的真实能力缺口反推，收敛出 `ObservationKV`、`InvariantKV`、`EligibilityKV` 三条更轻量、也更贴近实现边界的新候选。

因此，这份专题不是“有很多互相独立的点子”，而是“同一研究目标下，多轮否证、重构和收敛后的决策轨迹”。

## 先给总判断

当前这轮调研给出的总体判断是：`vLLM` 已经不只是“带 PagedAttention 的单机推理引擎”。它正在演进为一个同时覆盖：

- 本地 prefix cache
- hybrid KV cache
- CPU offload
- 外部 KV connector
- disaggregated prefill
- 多级 KV transfer

的统一推理运行时。

因此，最有潜力的系统论文切入点，不再是“再发明一种 KV cache 技巧”，而是把这些已经存在但仍然分裂的能力，收束成一个**统一、稳定、可度量、可泛化的 KV 管理系统**。

不过，随着这轮调研继续往前推进，我们也逐渐确认：传统的多级、分段、粒度、路由这些方向周围已经站满了强工作。真正还有希望留下来的空隙，开始转向：

- `KV state 何时可发布`
- `KV hit 之后系统还在为哪些东西付费`
- `什么状态才应该进入共享缓存`

如果只看当前入口级推荐，最值得优先继续推进的是：

1. 先看 `10_academic_restart_ideas` 里的 `PolyStateKV`、`TxnStateKV`、`CapsuleKV`
2. 再看 `11_capability_gap_ideas` 里的 `ObservationKV`、`InvariantKV`、`EligibilityKV`
3. 然后回看 `09_iterative_new_ideas` 里的 `ResidualModeKV` 与 `TypedPrefixKV`
4. 最后把 `08_collision_aware_new_ideas` 里的 `ShapeKV` 当作仍然值得保留的执行层近邻方案

这并不意味着前几轮方案失效。更准确地说，`03/04/06/07` 这些目录保留的是：

- 被后续方案继承的问题意识
- 已验证过的淘汰理由与撞车边界
- 仍然可作为备选 thesis 或对比基线的方向池

## 维护约定

这份 `README` 只负责四件事：

1. 说明这套专题的目标与范围
2. 提供阅读路径和目录导航
3. 概括 idea 的演进脉络
4. 给出当前入口级推荐结论

具体某个 idea 的机制细节、实验草案、重合度分析和风险评估，继续留在对应子目录中维护；跨多个 idea 的证据链、方法论和结果沉淀，继续放在 `05_research_archive/`。
