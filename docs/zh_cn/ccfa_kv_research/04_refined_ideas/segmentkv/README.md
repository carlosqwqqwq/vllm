# SegmentKV：面向 RAG 与 Agent 的精确分段 KV 复用

## 1. 一句话定位

`SegmentKV` 的目标是：把 prompt 从“单一线性前缀”提升为“可识别、可组合的 segment 序列”，从而让 `vLLM` 能在 RAG、Agent、工具调用等 workload 中复用 **非严格前缀但高度重复** 的上下文 KV。

## 2. 为什么这条线有独立价值

当前 `vLLM` 的 prefix caching 假设是：

- 请求之间共享一个从左到右的最长公共前缀

但真实 RAG / Agent workload 往往不是这样。常见上下文结构包括：

- 固定 system prompt
- 可部分复用的对话历史
- 多段 retrieved chunks
- 工具说明或 schema 模板
- 本轮用户增量输入

这些内容往往 **重复很多，但顺序、边界和拼接方式不完全一致**。因此它们不一定形成严格前缀，却非常值得复用。

## 3. 论文问题定义

可以把问题写成：

> 现有 prefix caching 主要优化严格左前缀复用，而 RAG、Agent 和工具调用场景中的上下文通常由多个高重复但非严格前缀的 segment 组成。如何在保持 exact semantics 的前提下，让推理框架能够识别、组合并复用这些 segment 的 KV，而不退化成大规模重算？

## 4. 核心设计

## 4.1 机制一：segment-aware 输入表示

请求在进入引擎前，不应立刻完全扁平化成 token 流，而要尽量保留 segment 身份，例如：

- `system`
- `history`
- `retrieval`
- `tool`
- `user_delta`

这一步的关键不在于 API 设计多复杂，而在于让运行时知道哪些内容是“可复用单元”。

### 4.2 机制二：exact segment hash graph

不是只对整条 token prefix 做 hash，而是对 segment 序列做 hash：

- segment 内容 hash
- segment 顺序信息
- segment 边界元数据
- 必要的 parent dependency

这样系统可以知道：

- 哪些 segment 可直接复用
- 哪些 segment 只是位置变了
- 哪些接缝需要重算

### 4.3 机制三：boundary re-materialization

`SegmentKV` 的关键不是“完全不重算”，而是 **只重算接缝**。

比如：

- system prompt 全复用
- retrieval chunk B/C 复用
- 新插入的 chunk A 与 user question 做局部重算

这样可以避免整段 prompt 全部重来。

### 4.4 机制四：selective re-prefill

对需要 stitch 的部分，只做局部 prefill，而不是整条 prompt prefill。这样它既保持 exact semantics，又能显著减少 TTFT。

## 5. 这条线与现有工作怎么区分

### 5.1 相比 prefix caching

你优化的是 **非严格前缀复用**，而不是更换 APC 的 eviction policy。

### 5.2 相比 LMCache / external KV reuse

你关注的是 **segment-level reuse 语义**，而不是单独做外部 KV store。

### 5.3 相比 CacheBlend 一类工作

如果写论文，差异点可以强调在：

- native vLLM integration
- 与 scheduler 的联动
- 面向实际 serving 输入结构的 segment IR

## 6. 在 vLLM 里怎么落地

## 6.1 兼容性判断

我给它的兼容性判断是：**中等偏高**。

原因是：

- 当前 `vLLM` 已有 prefix cache、connector、input processor 等基础设施
- 但输入路径默认偏“token stream first”
- 因此需要扩展请求和输入预处理的语义

### 6.2 重点改动模块

- `vllm/v1/engine/input_processor.py`
  - 注入 segment 元数据
- `vllm/v1/request.py`
  - 存储 segment 边界与 cacheability 信息
- `vllm/v1/core/kv_cache_manager.py`
  - 支持 segment-level hit / stitch metadata
- `vllm/v1/core/sched/scheduler.py`
  - 调度 selective re-prefill
- 上层 renderer / API 协议
  - 如果要让 segment 身份更稳定，需要配合输入构造层

## 7. 最适合的场景

这条线如果拿对 workload，会非常强：

- RAG 多段检索
- Agent 任务链
- 工具调用 / JSON schema / function template 频繁复用
- workflow 中多阶段共享上下文

如果 workload 大多是传统纯聊天，收益就会弱一些。

## 8. 实验建议

### 8.1 baseline

- 原生 prefix caching
- 原生 prefix caching + LMCache
- 全量重算

### 8.2 workload

- 共享 system prompt + 部分共享检索片段
- 多轮 agent planning / execution / validation
- 工具调用链中的模板化上下文

### 8.3 指标

- TTFT
- prefill tokens saved
- partial reuse ratio
- recomputation saved
- output quality consistency

## 9. 风险与边界

### 9.1 风险

- 这条线非常依赖 workload 设计
- 如果 segment 边界提取不稳定，会影响说服力
- 容易被质疑“是不是只适合 RAG/Agent”

### 9.2 边界

不要把它包装成“对所有通用 LLM serving 都最优”。更稳妥的定位是：

- 面向现代 RAG / Agent / tool-use serving 的精确 KV 复用系统

## 10. 什么时候值得优先选

如果你们手里已经有：

- RAG 数据
- Agent benchmark
- 工具调用 workload
- 共享模板很重的真实 trace

那 `SegmentKV` 会非常值得做，而且容易讲出“为什么 prefix cache 不够”的故事。

## 11. 结论

`SegmentKV` 不是最通用的主线，但它是最贴合 **RAG/Agent 真实上下文结构** 的主线。如果你的目标场景就在这里，它完全可能比 `TierKV` 更有针对性、更容易做出醒目的结果。
