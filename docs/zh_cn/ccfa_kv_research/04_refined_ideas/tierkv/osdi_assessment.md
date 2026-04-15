# TierKV：OSDI 可行性评估

## 1. 先给结论

如果现在必须从已有候选里只选一条最可行、最能直接优化 `vLLM`、同时最有机会冲击系统类 CCF-A 甚至 `OSDI` 的主线，我仍然会选 **TierKV**。

但我要先把判断说得非常直接：

> **当前版本的 TierKV 想法，本身还不够 OSDI。**

它现在更像一个**很好的系统论文方向**，但如果只是停留在“统一 GPU / CPU / Remote KV + scheduler 协同”这一层，整体更像：

- `EuroSys`
- `USENIX ATC`
- 或偏工程实现型的强系统会论文

而不是已经稳到 `OSDI`。

如果想把它推进到 **真正有 OSDI 竞争力**，必须把它从“多级 KV 工程整合”升级成一个更强的系统命题：

> **把 KV cache 从若干分散 feature，提升为一套面向 LLM serving 的 exact KV virtual memory system，并证明调度器、层级、传输协议和时延控制必须协同设计。**

## 2. 为什么我仍然选 TierKV，而不是其他方向

## 2.1 相比 AffinityKV

`AffinityKV` 很强，也很贴近真实部署，但它主要发力在：

- router
- DP / 多实例
- 入口层路由

如果没有很强的集群实验条件或真实部署 trace，它会更容易被 reviewers 觉得是“很好的 serving optimization”，但系统命题可能不如 `TierKV` 宽。

## 2.2 相比 QoSKV

`QoSKV` 务实、容易先打出 latency 结果，但它更像 APC 的第二阶段 runtime-quality 问题。它很有价值，但“问题空间”天然比 `TierKV` 小。

## 2.3 相比 SegmentKV

`SegmentKV` 对 RAG / Agent 很有针对性，但如果你的论文目标是 `OSDI`，过强的 workload 依赖会带来风险。reviewers 往往会问：

- 这是不是一个更窄的应用优化？
- 对一般 LLM serving 是否同样重要？

## 2.4 相比 HeteroKV

`HeteroKV` 新意很强，但实现和验证成本也最高。它更像一条 potential 很高、但第一版很难压出完整证据链的 harder problem。

## 2.5 TierKV 的独特优势

`TierKV` 的最大优势在于它同时满足四件事：

1. **问题够大**
   - 不是局部 patch，而是 KV 运行时系统问题
2. **和 vLLM 当前主线高度兼容**
   - connector、offload、hybrid、scheduler 已经都在主线里
3. **证据链完整**
   - release、PR、issue、文档都在指向同一方向
4. **可以扩展出完整系统故事**
   - 统一抽象
   - 调度协同
   - 一致性协议
   - 多 workload / 多场景收益

## 3. 现在这版 TierKV 为什么还不够 OSDI

这一步必须诚实。

### 3.1 问题一：当前表述还偏“工程整合”

如果只说：

- 我们统一了 GPU / CPU / Remote KV
- 增加了预取、下沉、提升

reviewers 很容易会问：

- 这是不是把已有能力做了一次工程整合？
- 与 `LMCache + FlexKV + CPU offload + scheduler patch` 相比，本质新东西在哪里？

这类问题对 `OSDI` 很致命。

### 3.2 问题二：核心 insight 还不够尖锐

`OSDI` 喜欢的不只是“系统做得全”，而是：

- 抓住一个重要但被忽视的系统性矛盾
- 用一个清楚、可推广的 insight 重构系统设计

目前 `TierKV` 还缺一个足够尖锐的中心洞见。

### 3.3 问题三：如果实验只停留在单机 offload，很难到 OSDI

如果最后实验只是：

- 单机 GPU + CPU offload
- TTFT 降一点
- 吞吐升一点

那大概率不够。因为当前 `OSDI` 在 LLM serving 上的 bar 已经很高，近两年已经有：

- 调度系统
- prefill / decode 解耦系统
- serverless LLM serving 系统
- 动态 KV 管理系统

换句话说，**“有收益”不够，必须是“有明显更强的系统命题和更广的收益面”。**

## 4. 我认为真正可打 OSDI 的 TierKV 应该长什么样

我建议把 `TierKV` 升级成下面这个版本：

## 4.1 新的一句话定位

`TierKV is an exact KV virtual memory system for LLM serving that jointly manages placement, promotion, prefetch, and scheduling across GPU, host, and remote tiers under latency and throughput constraints.`

这句话和“多级缓存”最大的区别是：

- 你不再是在做一个 cache feature
- 你是在做 **LLM serving 的 KV virtual memory**

这会让论文站位一下子更稳。

## 4.2 必须补上的核心 insight

我认为最值得押的中心 insight 是：

> **在 LLM serving 中，KV block 的价值不是静态的，也不是只由 recency 决定；它同时取决于阶段关键性（prefill/decode）、未来复用概率、回迁代价和请求时延预算。因此，多级 KV 不能只靠传统 cache policy 管理，而必须由 scheduler 协同进行 criticality-aware 管理。**

这个 insight 很关键，因为它把你和普通 cache/offload 系统拉开了。

它说明：

- 同一个 block，在不同时间的价值不同
- decode-critical block 和 cold reusable block 的优先级不同
- 传统 LRU / ARC 或 connector 自治策略不够
- 多级 KV 必须由调度器、内存层级和传输协议协同设计

这才像 `OSDI` 级别的 central thesis。

## 4.3 论文的贡献包应该收敛成 4 条

### 贡献 1：提出 exact KV virtual memory 抽象

不是“offload some blocks”，而是把 KV block 统一建模为：

- logical ownership
- physical residence
- transfer state
- criticality
- reload cost

并能跨 GPU / CPU / Remote tier 统一管理。

### 贡献 2：提出 criticality-aware KV scheduling

引入一套由 scheduler 驱动的 block 管理机制，核心不是 recency，而是：

- phase criticality
- reuse likelihood
- reload / transfer cost
- latency budget

这一步要明确写出为什么传统 cache policy 不够。

### 贡献 3：提出一致性安全的异步迁移协议

这是工程和系统价值都很高的一块。你必须证明：

- 多级 KV 的真正难点不只是 policy
- 还包括 allocator、scheduler、connector、transfer state 的一致性

如果这部分做扎实，会显著增强系统论文味道。

### 贡献 4：在真实 serving 场景下证明它是广泛收益，而不是单点收益

这一步必须覆盖：

- 单机长上下文
- 多轮会话
- external KV / PD
- 可能的 hybrid model
- latency-sensitive mixed workload

## 5. 这个方案是否真的能优化 vLLM

我的判断是：**能，而且接缝非常自然。**

## 5.1 技术上为什么可行

因为 `vLLM` 当前主线已经具备这些接缝：

- `V1 scheduler`
- `KVConnectorFactory`
- `SimpleCPUOffloadConnector`
- `Hybrid KV Cache Manager`
- `kv_transfer_config`
- `disaggregated prefill`

也就是说，`TierKV` 不是要创造一套新引擎，而是把已有的多条 KV 路径整合成统一运行时。

## 5.2 最适合改的模块

- `vllm/v1/core/sched/scheduler.py`
  - 负责 criticality-aware placement / prefetch / admission
- `vllm/v1/core/kv_cache_manager.py`
  - 负责统一 block metadata 与 residence model
- `vllm/v1/core/kv_cache_coordinator.py`
  - 负责多 group 协调
- `vllm/v1/simple_kv_offload/*`
  - 作为 host tier 的起点
- `vllm/distributed/kv_transfer/kv_connector/*`
  - 作为 remote tier 的统一入口

## 5.3 第一版原型怎么做才稳

不要一上来就做最大全家桶。最稳妥的三阶段是：

1. **GPU + CPU 两级**
   - 先验证 VM 抽象和 criticality-aware scheduler
2. **GPU + CPU + Remote 三级**
   - 接入 LMCache / FlexKV
3. **Hybrid-aware 扩展**
   - 增强 group-aware 管理

所以从实现角度看，这条线是**可做的**，而不是空想。

## 6. 它是否足够发表 OSDI

这里我给一个明确而克制的结论。

## 6.1 以“当前想法原型”来评估

**不够稳。**

如果你只做到：

- 多级统一
- 一些启发式预取和放置
- 在若干 benchmark 上优于 `vLLM + offload`

那更像一篇很好的系统优化论文，但还不够稳到 `OSDI`。

### 6.2 以“升级后的 TierKV”来评估

**有机会，但门槛很高。**

我认为它有 `OSDI` 潜力，前提是同时满足下面 5 条。

#### 条件 1：问题表述必须从“功能整合”升级为“KV virtual memory”

如果还是按“cache hierarchy”讲，容易偏弱。  
如果按“LLM serving 的 exact KV virtual memory”讲，问题层级会明显抬高。

#### 条件 2：必须有清晰 central insight

也就是前面那条：

- KV block value is phase- and latency-dependent, not just recency-dependent

没有这个 insight，系统容易显得只是把已有拼图拼起来。

#### 条件 3：必须有系统级、而非点状的收益

至少要证明：

- 提升 throughput
- 降低 TTFT / P99 ITL
- 在多 workload 下稳定
- 不牺牲 exact semantics

而不是只在一个长上下文 benchmark 上好看。

#### 条件 4：必须和强 baseline 打透

至少应包括：

- 原生 `vLLM`
- `vLLM + SimpleCPUOffload`
- `vLLM + LMCache`
- `vLLM + FlexKV`
- 若场景允许，还应对照：
  - `DistServe`
  - `Llumnix`
  - 与 offloading / dynamic KV management 接近的代表系统

#### 条件 5：最好要有 deployment-level 说服力

如果只是 microbenchmark，会偏弱。  
更有说服力的是：

- 多模型
- 多 workload
- 单机 + 多机
- PD / external KV 场景
- 最好有接近真实 trace 的实验

## 6.3 我的总体评估

如果按当前成熟度，我会这样判断：

- **可行性**：`8.5 / 10`
- **优化 vLLM 的自然度**：`9 / 10`
- **系统论文潜力**：`8 / 10`
- **OSDI 发表把握（当前版本）**：`4.5 / 10`
- **OSDI 发表把握（升级为 KV virtual memory thesis 后）**：`7 / 10`

这个分数不是“录用概率预测”，而是从论文成熟度和方向匹配度做的主观评估。

## 7. 它和近年 OSDI LLM serving 论文相比，差在哪

`OSDI` 最近在 LLM serving 上已经有一些很强的工作，例如：

- 调度与重调度
- prefill / decode 解耦
- serverless LLM serving
- 动态 KV 管理

这些工作的共同点是：

1. 都不是小 patch
2. 都抓住了一个根本矛盾
3. 都把矛盾上升为系统设计原则
4. 都有非常完整的实验

所以 `TierKV` 想进这个梯队，必须做到同样级别：

- 不是“支持更多 tier”
- 而是“重新定义 KV 在 LLM serving 里的系统管理方式”

## 8. 我建议怎么把它补到更像 OSDI

## 8.1 不要只做 TierKV，要做 `Criticality-Aware TierKV`

也就是说，论文标题和内容都要体现：

- criticality-aware
- scheduler-coordinated
- exact KV virtual memory

### 8.2 把 QoSKV 的一部分机制吸进来

尤其是：

- short-request latency protection
- hot prefix retention
- prefix state observability

这样能明显增强 runtime-quality 与 latency story。

## 8.3 适度吸收 AffinityKV，但不要让主线跑偏

可以把多实例 / router 场景当作扩展实验，证明：

- TierKV 的 residence / metadata 也能帮助上层 routing

但不要在第一版里把主线变成 routing paper。

## 8.4 实验上一定要突出“广泛而稳定”

推荐至少覆盖：

- Dense model
- Hybrid model
- chat workload
- long-context workload
- mixed short/long latency-sensitive workload
- PD / external KV workload

## 9. 最终建议

我的真实建议是：

### 9.1 如果你的目标是“先做成一篇稳的系统论文”

直接做 `TierKV`，但先按 `EuroSys / ATC` 标准打磨，等系统和实验足够强，再向 `OSDI` 靠。

### 9.2 如果你的目标是“从一开始就按 OSDI 打”

那就不要把它当“多级 offload 方案”，而要从第一天就把它定义成：

**LLM serving 的 exact KV virtual memory system**

并且尽快围绕这个 thesis 组织：

- 设计
- 指标
- baseline
- 实验场景

### 9.3 我是否建议继续押这条线

**建议。**

因为在当前所有候选里，它仍然是：

- 最自然优化 `vLLM`
- 最容易形成完整系统故事
- 最有希望长成 OSDI 级别命题

但前提是：**我们必须主动把它从“工程整合”提升成“系统原理 + 运行时协同”的论文。**

## 10. 参考依据

- [OSDI '25 CFP](https://www.usenix.org/conference/osdi25/call-for-papers)
- [DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin)
- [Llumnix: Dynamic Scheduling for Large Language Model Serving](https://www.usenix.org/conference/osdi24/presentation/sun-biao)
- [ServerlessLLM: Low-Latency Serverless Inference for Large Language Models](https://www.usenix.org/conference/osdi24/presentation/fu)
- [OSDI '24 Low-Latency LLM Serving Session](https://www.usenix.org/conference/osdi24/session/low-latency-ml-serving-1)
