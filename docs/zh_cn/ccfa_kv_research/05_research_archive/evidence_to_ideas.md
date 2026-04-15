# 证据到选题的映射

这份文档回答一个关键问题：

> 为什么这些证据，最后会支持这些 idea，而不是别的方向？

它的作用是把“原始证据”进一步整理成“**问题缺口 -> 研究主张**”。

## 1. 映射方法

本轮采用的映射方法很简单：

1. 先把证据按主题归类
2. 再提炼该主题背后的系统缺口
3. 最后判断这个缺口最适合长成哪条论文主线

只有当某个主题同时具备：

- 结构性问题
- 持续社区信号
- 与当前 `vLLM` 有自然接缝

它才会进入正式候选。

## 2. 主题一：多级 KV 能力已经存在，但没有统一成系统

### 2.1 证据

- 本地 `kv_connector/factory.py` 已注册大量 connector
- `SimpleCPUOffload` 已进入主线
- `LMCacheConnectorV1`、`FlexKVConnectorV1` 已 merged
- release 持续出现 offloading、connector、KV metadata、GPU-side KV events
- `#39702` 暴露了 offload 路径的一致性问题

### 2.2 缺口

当前的真实问题不是“有没有 KV offload / remote KV”，而是：

- 它们的状态语义不统一
- scheduler 没有把它们真正当成统一层级使用
- 一致性和收益都还不稳定

### 2.3 对应 idea

最自然对应的是：`TierKV`

### 2.4 为什么不是别的方向

因为这个主题的核心是“**系统整合**”，不是：

- prefix cache 单点策略
- 路由入口层
- segment 复用

## 3. 主题二：多实例 / DP 下缓存收益被路由浪费

### 3.1 证据

- `docs/serving/data_parallel_deployment.md` 明确说应考虑 KV cache state 做 intelligent routing
- production-stack router 已有 session routing
- router README 明确写了 `(WIP) prefix-aware routing`
- `production-stack #59` 是 prefix-cache-aware routing RFC
- `production-stack #855` roadmap 里明确列出 KV-cache-aware and prefix-aware routing

### 3.2 缺口

当前独立实例各自维护 KV cache，但路由层还没有稳定利用这些状态。结果是：

- 命中本该更高，却被错误分发
- 重复 prefill
- 真实 TTFT 变差

### 3.3 对应 idea

最自然对应的是：`AffinityKV`

### 3.4 为什么不是 TierKV

`TierKV` 主要解决的是实例内部和多级层次管理，而这里的问题发生在：

- DP rank 间
- serving instance 间
- router / gateway 层

## 4. 主题三：APC 已进入第二阶段问题

### 4.1 证据

- `#38194`：prefix hit 低于其他 stack
- `#37308`：严重 HoL blocking
- `#39321`：thinking tokens 污染 prefix cache
- `#33434`：prefix cache metrics 需求
- `#23083`：persistent / pinned prefixes 需求

### 4.2 缺口

APC 当前的问题已经不是“开不开”，而是：

- 是否会污染缓存
- 是否会拖慢短请求
- 是否能稳定保住热前缀
- 是否可监控、可解释

### 4.3 对应 idea

最自然对应的是：`QoSKV`

### 4.4 为什么它值得单列

因为这些问题可以收束成一条非常完整的 runtime-quality 主线，而不是零碎 patch。

## 5. 主题四：现代 workload 不总是严格前缀

### 5.1 证据

- production-stack 已开始讨论 agent workflow 与 context-aware routing
- 当前 APC 仍以严格 prefix 为核心
- 官方文档已承认共享上下文和 KV state 的价值

### 5.2 缺口

现代 RAG / Agent / tool-use 场景中，很多请求共享的是：

- system prompt
- history 片段
- retrieval chunk
- tool schema

它们高复用，但不一定形成单一最长前缀。

### 5.3 对应 idea

最自然对应的是：`SegmentKV`

### 5.4 为什么不是 QoSKV

`QoSKV` 的重点是 APC 的质量与稳定性；这个主题的问题更本质，是：

- APC 的“前缀假设”本身不够表达现代上下文结构

## 6. 主题五：Hybrid 模型要求异构 KV 表示

### 6.1 证据

- 本地 `kv_cache_manager.py` 注释已承认未来不同 group block size 会打破当前假设
- `docs/design/hybrid_kv_cache_manager.md` 已写出一系列 hybrid 限制
- `#23161` 直接提出 layer-specific dtype/layout 需求
- `#26201` 追踪 hybrid prefix caching 问题

### 6.2 缺口

当前 `vLLM` 虽然开始支持 hybrid KV，但底层表示仍然偏统一，难以自然支持：

- group-specific dtype
- group-specific layout
- group-specific block size
- group-specific backend

### 6.3 对应 idea

最自然对应的是：`HeteroKV`

### 6.4 为什么它排序没那么靠前

不是因为问题不重要，而是：

- 实现更深
- 验证更贵
- 第一版原型难度明显更大

## 7. 为什么最终排序是 `TierKV > AffinityKV > QoSKV > SegmentKV > HeteroKV`

### 7.1 `TierKV` 排第一

因为它在三方面最均衡：

- 问题大
- 兼容性高
- 系统故事最闭环

### 7.2 `AffinityKV` 排第二

因为它的证据链已经非常强，而且特别贴近真实部署，只是它更依赖多实例实验条件。

### 7.3 `QoSKV` 排第三

因为它更像 APC 的 runtime-quality 主线，务实且容易做，但如果包装不好，容易被看成 patch 集合。

### 7.4 `SegmentKV` 排第四

因为它对特定 workload 非常强，但通用性不如前三条。

### 7.5 `HeteroKV` 排第五

因为它最难，也最需要 hybrid benchmark 与深层实现支持。

## 8. 后续如何用这份映射

### 8.1 如果你要定主线

优先看每个主题背后的“缺口”是不是和你的实验条件匹配。

### 8.2 如果你要写 motivation

直接把：

- 证据
- 缺口
- 对应主张

三段式展开，就会很自然。

### 8.3 如果你要设计实验

直接从主题缺口反推 workload：

- 多级系统缺口 -> offload / remote / hybrid workload
- 路由缺口 -> DP / multi-instance workload
- APC 质量缺口 -> latency-sensitive workload
- segment 缺口 -> RAG / Agent workload
- hetero 缺口 -> hybrid model workload

## 9. 结论

这份映射文档的价值是：把“很多零散信号”压缩成“少数几个真正值得做的系统问题”。这样后面的人就不会重新回到“看到一个 issue 就想做一个题”的发散状态里。
