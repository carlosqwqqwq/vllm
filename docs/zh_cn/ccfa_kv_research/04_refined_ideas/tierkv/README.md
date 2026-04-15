# TierKV：面向 vLLM 的多级精确 KV 系统

## 1. 一句话定位

`TierKV` 的核心目标是：把当前 `vLLM` 里已经存在但彼此分散的 GPU prefix cache、CPU offload、外部 KV connector 和 disaggregated KV transfer，统一成一个 **精确语义、调度协同、多级管理** 的 KV hierarchy，从而同时提升吞吐与时延。

## 2. 为什么这条线最值得做

### 2.1 当前 vLLM 的真实状态

`vLLM` 已经不是只有 `PagedAttention + Prefix Cache` 了。现在主线里已经有：

- `V1 scheduler`
- `Hybrid KV Cache Manager`
- `SimpleCPUOffloadConnector`
- `LMCacheConnectorV1`
- `FlexKVConnectorV1`
- `NixlConnector`
- `MooncakeConnector`

这说明社区已经沿着“多级 KV / 外部 KV / hybrid-aware KV”在快速演进。

### 2.2 真正的缺口不是“没有能力”，而是“没有统一系统”

现在这些能力更多像并列组件，而不是一个统一 memory system：

- GPU 上的 prefix cache 自己有一套 block 生命周期。
- CPU offload 有自己的调度和状态机。
- 外部 connector 更像旁路能力，而不是系统统一的 remote tier。
- scheduler 只部分理解 KV 状态，无法全局决定“哪些块该留、该取、该迁移”。

这正是系统论文最有价值的切入点：**不是再发明一个点状优化，而是把碎片能力整成系统。**

## 3. 论文要回答的核心问题

可以把论文问题定义成：

> 在不改变模型语义、不依赖近似 KV 压缩的前提下，如何让 `vLLM` 的本地缓存、CPU offload 和外部 KV store 形成统一的多级 KV 层次，并让调度器能够显式利用这些层次来提升推理效率？

这里的关键词是：

- **exact reuse**
- **multi-tier**
- **scheduler coordinated**
- **hybrid-aware**

## 4. 核心主张

我建议把 `TierKV` 的贡献收敛成三条：

### 4.1 统一的多级 block 抽象

所有 KV block 都不再只是“在不在 GPU”。

每个 block 需要显式维护：

- `location`: GPU / CPU / Remote
- `state`: resident / pinned / transferring / evictable
- `reuse_score`: 未来复用价值
- `reload_cost`: 回迁或重算代价
- `group_id`: 对应哪个 KV group

这一步的价值在于：把今天 scattered 的 KV 能力统一到一个元数据抽象上。

### 4.2 调度器协同的收益感知放置与预取

当前很多路径还是“命中了就用、显存紧了就丢”。但真正的系统收益来自 scheduler 协同：

- prefill 热块优先保 GPU
- decode 敏感块提前预取
- 长尾低复用块优先下沉到 CPU / Remote
- 对 hybrid model 的不同 group 做差异化决策

可以采用一个简单但好解释的启发式分数：

`benefit = reuse_prob * reload_cost - transfer_cost - pressure_penalty`

不一定要上学习器。对于系统论文，**一个能解释、能分析、能消融的启发式策略反而更稳**。

### 4.3 一致性安全的异步传输协议

从社区 issue 看，KV offload 的一个真实风险是 race 和 TOCTOU。

所以 `TierKV` 不能只做“多一个层级”，还要做一套统一的异步协议，例如：

1. `probe`
2. `pin`
3. `reserve`
4. `transfer`
5. `commit`
6. `release`

这能系统性解决：

- block 在二次查找前被回收
- transfer 尚未完成就被 allocator 重用
- multi-tier state 之间不一致

## 5. 在 vLLM 里怎么落地

## 5.1 高兼容性原因

这条线和当前 `vLLM` 是 **顺着架构演进方向做增强**，不是逆着重写：

- `V1 scheduler` 已经是主线
- `KVConnectorFactory` 已经把多种 connector 纳入统一入口
- `SimpleCPUOffloadConnector` 已经证明 CPU tier 是可行的
- `Hybrid KV Cache Manager` 已经提供 group-aware 的起点

所以兼容性判断是：**高**。

### 5.2 重点改动模块

- `vllm/v1/core/sched/scheduler.py`
  - 增加 tier-aware 调度、预取窗口、transfer budget
- `vllm/v1/core/kv_cache_manager.py`
  - 扩展 block 元信息与统一 tier lookup 接口
- `vllm/v1/core/kv_cache_coordinator.py`
  - 把多 group 命中与多级放置协同起来
- `vllm/v1/simple_kv_offload/*`
  - 从“独立 CPU offload 路线”改为统一二级 cache 实现
- `vllm/distributed/kv_transfer/kv_connector/*`
  - 把外部 connector 视作 remote tier，而不是旁路
- `vllm/v1/metrics/*`
  - 增加 tier hit、prefetch hit、transfer overlap 等指标

## 6. 和已有工作的边界

### 6.1 相比 vLLM 本地 prefix caching

你不是在做另一个 prefix cache policy，而是在做 **多级 KV 管理系统**。

### 6.2 相比 LMCache / FlexKV

你不是做一个新的远端缓存层，而是在做 **scheduler 协同的统一层级系统**。

### 6.3 相比 KIVI / H2O / RocketKV

你不依赖近似压缩、裁剪或训练无关量化，而是保持 **exact semantics**。

## 7. 预期收益

如果设计得当，我预期它最容易在下面几类 workload 上出效果：

- 多轮 exact-prefix 会话：吞吐提升明显
- 长上下文 + 高并发：TTFT 降低明显
- PD / external KV 场景：P99 ITL 改善明显
- hybrid model：收益会比当前分裂式 offload 更稳定

适合主打的指标：

- output tokens/s
- req/s
- TTFT
- TPOT
- P99 ITL
- GPU / CPU / Remote hit ratio
- transfer bytes
- prefetch accuracy

## 8. 风险与边界

### 8.1 风险

- 实现跨度较大，会同时碰 scheduler、cache manager、connector
- 实验矩阵较大，需要同时覆盖本地与外部 tier

### 8.2 边界

这条线**不应该**在第一版里就做：

- 近似 KV 压缩
- 学习式 policy
- 强依赖某个特定 remote backend 的优化

第一版应坚持 KISS：先把精确语义的多级系统做顺。

## 9. 推荐实现顺序

### 9.1 阶段一：GPU + CPU

先做单机两级，把收益感知放置、预取和一致性协议打通。

### 9.2 阶段二：GPU + CPU + Remote

把 `LMCache/FlexKV/NIXL` 接成远端 tier。

### 9.3 阶段三：Hybrid-aware 扩展

把 group-aware 放置和多级 tier 协同扩到 hybrid 模型。

## 10. 结论

如果目标是“**尽量高概率做成一篇系统类 CCF-A 论文**”，`TierKV` 仍然是我最推荐的主线。它最大的优势不是某个局部技巧，而是：

- 问题大
- 证据足
- 兼容当前 vLLM
- 容易形成完整系统故事

如果你只能选一条线先投入，我建议优先做它。
