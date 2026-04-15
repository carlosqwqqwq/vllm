# 第七轮三个新方向对比

## 1. 先给最终排序

如果目标是：

- `vLLM` 真实可落地
- 尽量轻量
- 明显提升吞吐或延迟
- 与已有工作保持尽可能清楚的边界

我当前建议的排序是：

1. `ObservationKV`
2. `InvariantKV`
3. `EligibilityKV`

## 2. 为什么这么排

### 2.1 `ObservationKV`

它最强的地方是：

- 缺口真实且在源码里直接可见；
- mixed-API workload 很常见；
- 第一版不需要引入大规模新系统。

它最弱的地方是：

- 很容易被误读成 feature completion。

所以它是否能走到 OSDI，关键看你能不能把故事讲成：

- “generation-only APC 已经不适合真实混合服务”

而不是：

- “我把 prompt_logprobs 和 APC 搭起来了”

### 2.2 `InvariantKV`

它最强的地方是：

- 更像系统论文；
- 与现有 scheduling / offload / segmentation 论文边界更清楚；
- 新颖性空间比很多旧 idea 好。

它最弱的地方是：

- 容易滑向 kernel-heavy implementation；
- 第一版实验组织比 `ObservationKV` 更难。

### 2.3 `EligibilityKV`

它最强的地方是：

- 对 `vLLM` 当前“静态黑名单”现状打击很准；
- 与 `ObservationKV`、`InvariantKV` 都能形成互补。

它最弱的地方是：

- 单独做时容易显得太抽象；
- 必须用覆盖面扩大和真实性能回收来证明价值。

## 3. 哪些旧方向这轮被明确删除

这轮继续不保留以下方向作为主候选：

- 路由 / affinity / hot-prefix scheduling
- 分层 / offload / transfer / tiering
- segment / partial block / page-granularity 复用
- transaction / lifecycle / early publish
- hybrid state 大一统
- 普通 typed prefix / prompt ABI

原因不是它们没价值，而是：

- 公开工作已经很多；
- 或者离我们之前提出的旧方案太近，重合风险偏高。

## 4. 如果只允许选一条主线

如果只能保留一条，我建议：

- **主选 `ObservationKV`**

因为它最符合：

- 轻量级
- 普适性
- 直接改 `vLLM`
- 更容易较快做出 prototype 与收益

如果你追求更强的“系统味”，再把：

- `InvariantKV`

作为第二主线。

如果你希望最后论文结构更完整，则可以把：

- `EligibilityKV`

作为语义/协议层，给前两者提供统一约束。

## 5. 最可能的论文组合

如果后面真的要往 CCF-A/OSDI 打磨，我最推荐的组合是：

- **主线：`ObservationKV`**
- **增强模块：`InvariantKV`**
- **协议层：`EligibilityKV`**

这三者的关系可以写成：

1. `EligibilityKV`
   - 定义什么时候 APC 允许启用。
2. `ObservationKV`
   - 解决可启用请求为什么仍然绕过缓存。
3. `InvariantKV`
   - 解决启用 APC 后为什么快后端又把它关掉。

这样论文故事就不是三个零散 patch，而是一条完整链条：

- **先判断能不能 cache**
- **再让更多 API 真能用 cache**
- **最后让 fast path 也能稳定用 cache**

## 6. 当前总判断

截至 `2026-04-15`，这三个方向都不能说“确保没人做过”。

但相较前面几轮，它们已经是我重新筛过后保留下来的、当前更符合你约束的三个候选：

- 更轻量
- 更贴 `vLLM`
- 更少和近期热门空间正面冲突
- 更适合先出 prototype 再决定是否继续做大
