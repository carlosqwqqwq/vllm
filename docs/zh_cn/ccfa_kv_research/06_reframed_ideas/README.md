# 第二轮重构后的新候选

## 1. 为什么还要再来一轮

前一轮调研已经暴露出一个问题：很多看起来合理的 `KV cache` 方向，一旦落到相关工作上，就会明显重合。

重合最严重的几条线包括：

- 多级 KV hierarchy / offload / external cache
- prefix-aware routing
- hot-prefix retention / QoS-aware prefix scheduling
- 非前缀分段复用
- hybrid model 的异构 KV 管理

这些方向并不是坏方向，而是**周围已经有很多强工作**。如果继续沿着它们做，论文很容易变成“在已有路线上的再优化”。

因此，这一轮的目标不是再发散更多 patch，而是基于前面的淘汰结果，重新找三个**问题定义更干净、和旧方案边界更清楚、同时又能明显增强 `vLLM` 的新 thesis**。

## 2. 这一轮怎么选题

我刻意避开了前一轮已经拥挤的“层次、热度、路由、普通 prefix 调度”，转而围绕三个此前没有被系统性占满的空隙来想：

- `复用语义`
  - 当前 APC 到底缓存的是什么，以及什么才应该算“可复用前缀”。
- `缓存粒度`
  - 当前固定 block/page 粒度是不是已经成了 exact reuse 的天花板。
- `服务抽象`
  - Chat / Agent 场景本质上是有状态、可分叉的，但今天的 serving 仍然基本是“把整段上下文重新发进来，再祈祷 APC 命中”。

## 3. 三个新候选

- [LineageKV](lineagekv/README.md)
  - 把“执行历史”和“可复用前缀”分开，解决 reasoning / tool / hidden trace 让 APC 语义失真的问题。
- [FrontierKV](frontierkv/README.md)
  - 只在真正需要的复用边界上做细粒度缓存，把固定 block size 从 hard tradeoff 变成可管理对象。
- [SessionKV](sessionkv/README.md)
  - 把对话和 agent 运行时的 KV state 提升成一等公民，支持 snapshot / fork / rollback / commit，而不是继续走“无状态重放 + prefix cache”。

## 4. 这三条线和旧方案的关系

- `LineageKV` 不是 `QoSKV` 的变体。
  - `QoSKV` 更偏 policy 和 QoS；`LineageKV` 更偏 APC 的**语义边界**。
- `FrontierKV` 不是 `HeteroKV` 的变体。
  - `HeteroKV` 关注 model group heterogeneity；`FrontierKV` 关注 **reuse frontier 的粒度适配**。
- `SessionKV` 不是 `SegmentKV` 的变体。
  - `SegmentKV` 试图复用任意段；`SessionKV` 只管理**精确、可验证、带生命周期的会话状态**。

## 5. 当前总判断

如果以“既能增强 vLLM，又要尽量避开已知重合，还要保留冲系统类 CCF-A / OSDI 的空间”为标准，我目前的排序是：

1. `SessionKV`
2. `LineageKV`
3. `FrontierKV`

这个排序不是按“最容易做”来的，而是按“问题定义是否足够像一篇大论文”来的。
