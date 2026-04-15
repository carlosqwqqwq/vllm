# 第七轮从“能力缺口”反推的新候选

这一轮不再从“大而全的系统母题”继续发散，而是反过来问一个更现实的问题：

> `vLLM` 的 prefix cache 今天到底在哪些高价值能力面上还进不去快路径？

如果一个方向同时满足下面三个条件，它才会进入这一轮候选集：

1. `vLLM` 当前真实存在能力缺口，而不是纯想象问题。
2. 近两年公开工作还没有把这个问题完整占满。
3. 第一版可以用**轻量原型**落到 `vLLM`，并有希望带来吞吐或延迟收益。

基于这轮重新筛选，最终保留三个方向：

- [ObservationKV](./observationkv/README.md)
  - 面向混合 API 的观测友好前缀缓存。
- [InvariantKV](./invariantkv/README.md)
  - 面向异构后端的批不变精确前缀缓存。
- [EligibilityKV](./eligibilitykv/README.md)
  - 面向复杂模型与请求形态的可验证 cacheability 协议。

## 为什么是这三个

前面多轮已经基本确认，下面这些空间已经过于拥挤，或者离已有工作太近：

- 分层 / offload / transfer
- prefix-aware scheduling / short re-prefill clustering
- RAG/Agent 分段复用
- speculative KV 事务
- 普通 typed prefix / tool schema cache
- 多级 page 粒度 / partial block 复用

相较之下，这一轮保留下来的三个方向分别卡在三个不同层面：

- `ObservationKV`
  - 解决“为什么很多请求明明共享前缀，却因为要返回更多观测信息而被迫绕过 APC”。
- `InvariantKV`
  - 解决“为什么一旦打开 batch invariance 或换更快后端，APC 反而被系统保守地关掉”。
- `EligibilityKV`
  - 解决“为什么系统今天只能用一堆静态黑名单判断 prefix cache 能不能开，而不是更细粒度地做可验证启用”。

## 这一轮的总判断

这三个方向都**不能保证全球没人做过**。

但截至 `2026-04-15` 这轮检索，我的判断是：

- 这三条线都**没有找到完全等价的公开系统论文**；
- 三者都比前几轮那些 routing / offload / segmentation / scheduling 方向更不容易直接撞上最新强工作；
- 其中最值得优先继续验证的是：
  1. `ObservationKV`
  2. `InvariantKV`
  3. `EligibilityKV`

## 建议阅读顺序

1. [comparison.md](./comparison.md)
2. [ObservationKV](./observationkv/README.md)
3. [InvariantKV](./invariantkv/README.md)
4. [EligibilityKV](./eligibilitykv/README.md)
5. [research_log.md](./research_log.md)
