# HeteroKV：面向 Hybrid LLM 的异构 KV 虚拟内存

## 1. 一句话定位

`HeteroKV` 的目标是：打破当前 `vLLM` 对不同 attention group 共用统一 KV 表示的隐含假设，让 hybrid model 的不同层可以拥有 **不同 dtype、layout、block size、backend 和 cache policy**，并在逻辑层面被统一调度。

## 2. 为什么这条线值得保留

虽然它不是我最推荐的首发方向，但它在学术上很强，因为它抓住了一个越来越明显的趋势：

- LLM 不再都是统一的 dense full-attention transformer
- Hybrid model 里可能同时出现 full attention、sliding window、MLA、linear attention、Mamba/SSM

这些层的 KV 需求并不一样。当前 `vLLM` 已经有 `Hybrid KV Cache Manager`，但本地注释和社区 RFC 都表明：

- 现有 block 表示仍然偏统一
- 未来很可能需要 group-specific block size / dtype / layout

这说明问题已经暴露，只是社区还没有完全重构到底。

## 3. 论文问题定义

可以把问题写成：

> 随着 hybrid LLM 的普及，不同 attention group 对 KV cache 的数据类型、布局、页大小和存储后端提出异构需求。现有统一 KV 表示会导致资源浪费、实现约束和兼容性问题。如何设计一种 heterogeneity-aware 的 KV 虚拟内存，让不同 group 的物理实现可以差异化，而请求层仍看到统一的逻辑上下文？

## 4. 核心设计

## 4.1 机制一：heterogeneous page class

不同 attention group 使用不同 page class：

- group-specific dtype
- group-specific layout
- group-specific block size
- group-specific storage backend

这样不同 group 就不必被迫套进同一物理页抽象。

### 4.2 机制二：logical span table

请求层不直接面对“第几个 block”，而是面对“某个 group 上某段 token span 的缓存状态”。

这意味着：

- 逻辑 token span 统一
- 物理页实现异构
- 调度器可在逻辑层统一做决策

### 4.3 机制三：group-aware prefix hit 与放置决策

当前很多逻辑默认各 group 的 block 粒度能简单对齐。`HeteroKV` 需要把 hit 判定改成：

- 基于逻辑 span 的交集
- group-specific 的物理映射
- group-specific 的放置与 eviction

### 4.4 机制四：backend capability negotiation

不是所有 group 都必须用同一个 backend。例如：

- MLA group 用最适合它的 backend
- full attention group 保持现有 backend
- 某些 group 允许 offload，另一些 group 不允许

这会让 hybrid 模型的系统支持真正“长出来”。

## 5. 在 vLLM 里的兼容性判断

## 5.1 为什么不是低兼容，而是中等兼容

虽然改动很深，但方向本身是顺着现有架构走的：

- 已经有 `Hybrid KV Cache Manager`
- 已经有 multiple KV groups
- 本地代码和 RFC 已经承认未来需要不同 block size / layout

所以它不是错误方向，而是 **更深、更慢、更贵** 的方向。

### 5.2 重点改动模块

- `vllm/v1/core/kv_cache_interface.py`
- `vllm/v1/core/kv_cache_manager.py`
- `vllm/v1/core/kv_cache_coordinator.py`
- `vllm/v1/core/kv_cache_utils.py`
- `vllm/v1/worker/gpu_model_runner.py`
- attention backend 适配层

## 6. 这条线最强的学术点

### 6.1 直接回答 hybrid LLM 的系统支持问题

它不是一般性的 cache policy，而是面向下一代模型结构的系统重构。

### 6.2 把“统一逻辑视图 + 异构物理实现”做出来

这是很标准、也很强的系统论文结构。

### 6.3 可以自然连接 hybrid prefix caching 的未解问题

如果写得好，它能把：

- group-specific layout
- hybrid prefix caching
- offload / backend 选择

串成一个完整故事。

## 7. 实验设计建议

### 7.1 模型

一定要选 hybrid 模型，否则这条线的优势体现不出来，例如：

- GPT-OSS 类
- Gemma 3
- Bamba / Minimax 一类
- 其他带 hybrid attention / Mamba 结构的模型

### 7.2 baseline

- 当前 `vLLM` hybrid KV manager
- 统一 block size / dtype / layout 的实现
- 仅做多 group，不做 heterogeneous page class 的实现

### 7.3 指标

- KV 内存效率
- TTFT / TPOT
- hybrid prefix hit
- backend compatibility
- 不同 group 的 cache waste

## 8. 风险与边界

### 8.1 风险

- 实现复杂度明显最高
- 对 hybrid benchmark 依赖非常强
- 稍不小心就会变成“架构重写”，第一版难度大

### 8.2 边界

第一版不要同时做：

- 多级 tier
- 跨实例路由
- 复杂 semantic reuse

否则 scope 会爆炸。

## 9. 什么时候值得优先选

如果你们团队：

- 很强于底层系统实现
- 有 hybrid 模型和多 backend 实验条件
- 愿意打更偏 harder problem 的选题

那 `HeteroKV` 是值得押一把的。

## 10. 结论

`HeteroKV` 不是当前最稳的第一选择，但它是五条方案里 **学术新意最强的一条**。如果做成，论文会非常有辨识度；如果资源有限，它更适合作为中长期方向，而不是第一版就全力冲刺的主线。
