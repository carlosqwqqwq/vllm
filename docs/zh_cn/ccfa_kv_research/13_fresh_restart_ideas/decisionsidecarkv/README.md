# DecisionSidecarKV：在 prefix 共享场景下复用 decision-side state

更新日期：2026-04-15

## 1. 一句话 thesis

`DecisionSidecarKV` 的核心主张是：

> 当 KV data plane 越来越快时，structured output、grammar decoding、schema validation、sampling decision 这类 decision plane 正在成为新的开销来源；
> 如果多个请求共享前缀并共享 schema / grammar / tool 定义，系统不应只复用 KV，也应复用 prefix-conditioned 的 decision-side state。

## 2. 为什么这是一个值得考虑的新方向

过去大家把复用几乎都放在：

- KV data plane
- prefix token reuse
- cache routing / movement

但现在有一个越来越现实的问题：

- 即使 KV 已经命中；
- 某些请求仍要在 decision plane 上继续支付不小代价。

最典型的场景是：

- structured output
- grammar-constrained decoding
- JSON schema generation
- 某些 prefix-conditioned 采样辅助状态

这说明：

- **并不是所有“共享前缀”的收益都已经在 KV 层兑现完了。**

## 3. 为什么它和最近工作不完全一样

### 3.1 与 `SIMPLE` 的边界

- `SIMPLE`
  - 更关注 sampling decision plane 的并行化和服务化。
- `DecisionSidecarKV`
  - 研究的是：
  - **跨请求的 prefix-conditioned decision-state reuse**

### 3.2 与 `XGrammar 2` 的边界

- `XGrammar 2`
  - 更关注 structured generation engine、grammar execution 和 caching。
- `DecisionSidecarKV`
  - 不打算重新做 grammar engine；
  - 研究的是：
  - 在 `vLLM` request / scheduler / structured-output runtime 中，
  - 如何让共享前缀请求复用部分 decision-side state。

### 3.3 当前结论

截至 `2026-04-15`，我没有找到一篇公开系统论文直接把：

- `prefix-conditioned cross-request decision-side reuse`

作为主命题写清楚。

但它和最新 decision-plane 工作的距离确实比前两条更近，所以风险更高。

## 4. 为什么它能在 vLLM 中落地

### 4.1 `structured_output_request` 已经是一等对象

`vllm/v1/request.py`

当前 request 生命周期里已经显式维护：

- `structured_output_request`

这说明：

- 决策侧状态已经进入 request 对象，而不是完全在模型外部。

### 4.2 scheduler 已显式处理 grammar bitmask

`vllm/v1/core/sched/scheduler.py`

当前已经有：

- `structured_output_manager.grammar_bitmask(...)`
- `GrammarOutput(...)`
- grammar validate tokens

这说明：

- decision-side state 已经进入调度和执行路径；
- 不是纯理论概念。

### 4.3 第一版可以只选很小的对象

第一版无需一上来复用所有决策状态。

可以只选：

1. grammar / schema bitmask 相关 state
2. prefix-conditioned validation state
3. 某些轻量采样辅助 state

这样风险会低很多。

## 5. 可能的机制

### 5.1 sidecar object

定义一个极小 decision sidecar，包含：

- 与 prefix 对齐的 grammar / schema state
- 可快速恢复的 validation context
- 请求类型 / schema 指纹

### 5.2 匹配条件

只有当下面条件都满足时才允许复用：

- prefix 一致
- schema / grammar 一致
- tool / output mode 一致
- 决策状态语义与当前请求兼容

### 5.3 执行模式

第一版可以只支持：

- `REBUILD`
  - 不复用 decision-side state
- `REUSE_DECISION_SIDECAR`
  - 命中并恢复决策侧状态

## 6. 为什么它符合你们的要求

### 6.1 轻量级

第一版可以不碰模型 kernel，只动：

- request metadata
- scheduler structured output path
- grammar / schema runtime 的状态恢复

### 6.2 普适性

它并不绑定某个具体 schema，只要系统支持：

- structured output
- grammar constraints
- 复杂 generation mode

这条线就成立。

### 6.3 高吞吐 / 低延迟

如果 structured generation 越来越占比高，那么减少 decision-plane 重复工作：

- 会改善尾延迟；
- 也会释放 CPU / 控制平面资源。

## 7. 它为什么不是当前第一优先级

主要有两个原因：

1. 与 `SIMPLE / XGrammar 2` 更近
   - 需要更精细地切边界。
2. 更容易偏离 “KV cache 优化”
   - 如果讲不好，会被认为已经转成 structured generation 论文。

所以它更像：

- 高风险高收益的备选方向

而不是现在最稳的主线。

## 8. 当前判断

截至 `2026-04-15`：

- **创新性：高**
- **与最新论文重合风险：中**
- **vLLM 落地性：中高**
- **包装难度：高**
- **当前三条里的推荐程度：第三**

## 9. 参考资料

- SIMPLE: <https://arxiv.org/abs/2512.00719>
- XGrammar 2: <https://arxiv.org/abs/2601.04426>
