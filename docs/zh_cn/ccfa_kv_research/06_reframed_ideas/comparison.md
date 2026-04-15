# 三个新候选的对比与推荐

## 1. 先给结论

如果目标是：

- 直接增强 `vLLM`
- 避开前一轮已经明显重合的路线
- 保留冲 `OSDI` 这类系统顶会的空间

那我当前最看好的排序是：

1. `SessionKV`
2. `LineageKV`
3. `FrontierKV`

## 2. 为什么是这个排序

## 2.1 SessionKV：问题最“大”，最像系统抽象升级

`SessionKV` 的优势在于，它不是在现有 APC 上继续修修补补，而是在挑战今天的 serving 边界：

- 为什么 Chat / Agent 明明是 stateful 的，服务层却还是 stateless replay？
- 为什么 fork / retry / tool call / rollback 这些天然存在的执行结构，没有成为一等公民？

这条线如果做成，会让 `vLLM` 获得一类新的能力，而不只是更高一点 hit ratio。

## 2.2 LineageKV：问题非常真实，而且足够“尖”

`LineageKV` 的优势在于：

- 它直接击中了 reasoning / tool 调用时代 APC 语义失真这个新问题；
- 它不是 ordinary prefix scheduling，也不是 hotness policy；
- 它能和当前 `vLLM` 社区里的 `thinking token pollution`、prefix hit 不稳等现象直接对上。

它的问题定义很锋利，但范围比 `SessionKV` 更窄。

## 2.3 FrontierKV：最工程可落地，但 thesis 略小

`FrontierKV` 很有价值，因为固定 block size 的 tradeoff 在 `vLLM`、`TensorRT-LLM` 和多篇系统里都真实存在。

但相比前两条，它更像：

- 一个重要的 runtime / cache geometry 创新
- 而不是一个服务抽象层面的重定义

所以它很可能是一篇强系统优化论文，但主张的“高度”略低一点。

## 3. 三条线分别最适合解决什么

### 3.1 SessionKV

最适合解决：

- 多轮会话
- agent tool call
- workflow branching
- retry / rollback / A/B 分支

### 3.2 LineageKV

最适合解决：

- reasoning tokens
- hidden chain-of-thought
- tool traces / transient metadata
- “执行过，但不该变成共享前缀”的内容

### 3.3 FrontierKV

最适合解决：

- block size 导致的边界 miss
- 最后一块整块重算
- 粗粒度 block 让 exact APC 的收益被硬件友好性吞掉

## 4. OSDI 潜力的主观打分

- `SessionKV`
  - `问题高度`: `9/10`
  - `vLLM 兼容性`: `7.5/10`
  - `工程复杂度`: `高`
  - `OSDI 潜力`: `8/10`

- `LineageKV`
  - `问题高度`: `8/10`
  - `vLLM 兼容性`: `8.5/10`
  - `工程复杂度`: `中`
  - `OSDI 潜力`: `7/10`

- `FrontierKV`
  - `问题高度`: `7/10`
  - `vLLM 兼容性`: `8/10`
  - `工程复杂度`: `中高`
  - `OSDI 潜力`: `6.5/10`

## 5. 最终建议

如果你想追求**最强的新意和最像大论文的问题定义**，优先看 `SessionKV`。

如果你想在**保证新意的同时更快落地**，优先看 `LineageKV`。

如果你更想做一个**硬核 runtime / cache geometry 方向**，优先看 `FrontierKV`。
