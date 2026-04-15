# 调研过程与方法

## 1. 调研目标

本轮调研不是泛泛地“看一看 vLLM”，而是有明确约束的：

- 目标方向：`KV cache`
- 目标载体：`vLLM`
- 目标论文类型：系统类 `CCF-A`
- 目标要求：
  - 轻量级
  - 普适性
  - 高吞吐
  - 低延迟
  - 能作用到当前 `vLLM`

这个约束非常重要，因为它决定了调研不会把重心放在“纯算法近似优化”上，而会优先寻找**系统层缺口**。

## 2. 本轮采用的调研顺序

这轮采用的是一种“**本地事实优先，时间敏感信息再外查**”的顺序。

### 2.1 第一步：先看本地源码和官方文档

先建立不会轻易变化的结构事实，包括：

- 当前引擎主线是不是 `V1`
- scheduler、KV manager、worker、connector 怎么组织
- `Hybrid KV Cache Manager` 到了什么程度
- 官方设计文档自己承认了哪些限制

这样做的原因是：

- 先建立架构地基，避免被社区讨论带偏
- 后面看 GitHub issue / PR 时，能判断哪些是真缺口，哪些只是局部抱怨

### 2.2 第二步：只用官方 / 一手渠道验证时间敏感信息

会变化的信息包括：

- 最新 release
- 最新 merged PR
- 当前 open issue
- production-stack roadmap

这些信息全部通过官方或一手来源确认：

- `vllm-project/vllm`
- `vllm-project/production-stack`
- 相关官方仓库的 PR / issue / release

这里刻意避免依赖二手博客或转述材料。

### 2.3 第三步：把“代码事实”和“社区信号”对齐

一个想法只有在下面三类证据里至少占两类，才会被提升为候选方向：

1. **代码信号**
   - 当前代码里已经有相关基础设施，或明确暴露了结构限制
2. **文档信号**
   - 官方文档明确承认问题重要，或 roadmap 中出现相关方向
3. **社区信号**
   - 有持续的 PR / RFC / issue 指向同一类缺口

这样可以减少“只看 issue 就拍板”的风险。

## 3. 具体看了哪些东西

## 3.1 本地代码与文档

重点看了这些内容：

- `vllm/engine/llm_engine.py`
  - 用来确认 `V1` 是否已经是主线
- `vllm/v1/engine/*`
  - 理解请求主链路和调度主循环
- `vllm/v1/core/sched/scheduler.py`
  - 看 scheduler 是否已经接入 KV connector / offload 路径
- `vllm/v1/core/kv_cache_manager.py`
  - 看 KV block 表示、prefix cache 命中和 group 假设
- `vllm/distributed/kv_transfer/kv_connector/factory.py`
  - 确认有哪些 connector 已经被纳入统一入口
- `docs/design/prefix_caching.md`
  - 理解 APC 的当前语义和限制
- `docs/design/hybrid_kv_cache_manager.md`
  - 理解 hybrid KV 当前的抽象与局限
- `docs/serving/data_parallel_deployment.md`
  - 看官方是否承认 KV-aware routing 的价值
- `docs/deployment/integrations/production-stack.md`
  - 看 production-stack 的定位和关键词

## 3.2 GitHub 社区与官方路线

重点核对了：

- `vLLM` 最新 release
- 与 KV 直接相关的 merged PR
- 与 prefix cache / hybrid KV / offload / routing 相关的 open issue
- `production-stack` 的 router README、RFC 和 roadmap

## 4. 这轮调研采用的判断标准

为避免“idea 很多但无法收敛”，这轮对每个方向都按同一组标准打分。

### 4.1 保留标准

一个方向要进入候选方案，至少要满足：

1. **能兼容当前 vLLM**
2. **更像系统问题，而不是单模型技巧**
3. **能解释明确痛点**
4. **第一版原型可做**
5. **能设计清楚 baseline 和指标**

### 4.2 降级标准

下面几类方向会被主动降级：

- 明显需要改模型训练或改模型结构
- 强依赖某一种硬件或某一种 kernel
- 很难解释与当前 `vLLM` 主线的关系
- 结果容易只体现为局部 patch，而不是完整系统故事

## 5. 这轮如何从事实走到 idea

本轮不是“先有 idea 再找证据”，而是按下面这条链路推进：

### 5.1 先确认系统现状

结论是：

- `V1` 已是主线
- `vLLM` 已有 prefix cache、CPU offload、external KV connector、hybrid KV groups

这意味着当前研究重点不应再是“从零做 KV cache”。

### 5.2 再找未闭环的系统缺口

从文档、源码和 issue 看，真正重复出现的缺口主要有：

- 多级 KV 能力存在，但没有统一系统
- 多实例 / DP 下缺少 KV-aware routing
- prefix cache 有时延副作用和污染问题
- RAG / Agent 的非严格前缀复用不足
- hybrid model 的 KV 抽象仍偏刚性

### 5.3 最后才收敛为候选主线

于是最终保留了五条主线：

- `TierKV`
- `AffinityKV`
- `QoSKV`
- `SegmentKV`
- `HeteroKV`

## 6. 这轮调研里最重要的方法经验

### 6.1 不要只看“功能有没有”，要看“体系闭环了没有”

例如 `vLLM` 已经有 CPU offload、LMCache、FlexKV，并不代表“这个方向已经做完了”。真正的问题往往是：

- 这些能力有没有统一抽象
- scheduler 能不能真正利用这些能力
- 是否已经能稳定转化为吞吐和时延收益

### 6.2 不要把 issue 数量等同于研究价值

issue 多只能说明“大家在讨论”。更重要的是：

- 官方文档是否承认问题
- release / PR 是否持续推进
- 本地源码里是否已经露出结构限制

### 6.3 官方 roadmap 是非常强的信号

尤其像：

- KV-aware routing
- prefix-aware routing
- hybrid KV
- CPU offloading

这些如果同时出现在文档、PR 和 roadmap 中，就说明它们不是偶发问题，而是主线方向。

### 6.4 选题不能只看“新”，还要看“和当前 vLLM 的接缝”

系统类论文最怕的是：

- idea 很新
- 但和现有系统结合不自然

所以这轮一直优先考虑“顺着 vLLM 演进方向补完”的思路。

## 7. 后人如果要复用这套方法，建议怎么做

建议以后继续沿着下面的最小流程：

1. 先读本地核心代码与设计文档
2. 再查官方 release / PR / issue / roadmap
3. 把事实整理成“系统现状 -> 未闭环缺口 -> 候选主线”
4. 用统一标准给候选方向排序
5. 最后再写论文提案或实验计划

## 8. 本轮调研的局限

这轮调研虽然尽量建立在一手资料上，但仍然有边界：

- 没有实际跑 benchmark，只做了架构与社区层面的判断
- 没有系统性读完 production-stack 全部代码
- 对某些外部 KV backend 的内部机制只做了与 `vLLM` 接缝相关的抽取

因此它更适合作为：

- 选题与系统设计的前期依据

而不是：

- 最终实验结论

## 9. 结论

这轮调研最值得保留下来的，不只是“推荐做 `TierKV`”，而是这套方法本身：

- 先看本地架构
- 再看官方变化
- 再找持续暴露的系统缺口
- 最后才收敛选题

这比直接堆论文或堆 issue 更适合系统类方向。
