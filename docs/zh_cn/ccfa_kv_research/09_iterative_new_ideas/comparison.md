# 第五轮三个方向的横向比较

## 1. 先给结论

如果只保留一个方向先做下一步验证，我现在的排序是：

1. `ResidualModeKV`
2. `TypedPrefixKV`
3. `PromptABIKV`

但如果只问“哪个方向的新颖度潜力更大”，排序会更接近：

1. `TypedPrefixKV`
2. `ResidualModeKV`
3. `PromptABIKV`

也就是说：

- `ResidualModeKV` 更适合先做原型；
- `TypedPrefixKV` 更可能在包装成功后形成更强的新颖度；
- `PromptABIKV` 更适合成为另外两条线的系统接口层支撑。

## 2. 对比表

| 方向 | 核心问题 | 最接近已有工作 | 当前重合风险 | 实现难度 | 证据难度 | 当前建议 |
| --- | --- | --- | --- | --- | --- | --- |
| `ResidualModeKV` | APC hit 后 residual work 是否用了错误执行模式 | `LAPS`、`k-LPM`、`SGLang` | 中 | 中 | 中 | 优先做 profiling |
| `TypedPrefixKV` | 异构输入下什么才算同一个 exact prefix | `Prompt Cache`、provider prompt caching、vLLM typed-input issues | 中低到中 | 中高 | 高 | 作为强备选并继续收证据 |
| `PromptABIKV` | 应用层如何稳定地产生可命中的 exact prefix | provider docs、`Don't Break the Cache` | 中 | 中 | 中高 | 作为支撑层保留 |

## 3. 三者之间是否重合

它们有关联，但不是同一个点：

- `ResidualModeKV`
  - 关注 **命中之后如何执行**
- `TypedPrefixKV`
  - 关注 **命中之前系统如何定义 exact prefix**
- `PromptABIKV`
  - 关注 **应用层如何持续生成这个 exact prefix**

因此，更自然的关系不是“三选一”，而是一个潜在的层次化故事：

1. `PromptABIKV` 负责产生稳定输入；
2. `TypedPrefixKV` 负责在 runtime 内定义和验证 typed exact prefix；
3. `ResidualModeKV` 负责命中之后挑选更快的执行模式。

## 4. 为什么不再保留 eviction/look-ahead 那条线

这轮最明确的结论之一就是：**不要再把时间继续花在 prefix cache eviction / lease / look-ahead retention 上。**

原因很简单：

- `k-LPM`
- `PCR`
- `KVFlow`
- `HotPrefix / LPC`

这条线周围已经很拥挤了。即使还能做，也很难变成当前阶段最干净的新主线。

## 5. 最小验证路线

### 5.1 对 `ResidualModeKV`

先做：

1. residual length histogram
2. graph replay/fallback reason
3. padding ratio
4. cascade enable reason
5. TTFT p50/p99

只要发现 baseline 在 APC high-hit 下仍然有稳定 inefficiency，这条线就能继续。

### 5.2 对 `TypedPrefixKV`

先做：

1. prompt embeds 是否实际禁用 APC
2. multimodal 是否存在 correctness-driven conservative path
3. cache salt / raw prompt / tool ordering 是否导致 typed exactness 缺位
4. workload coverage 扩展是否明显

只要能证明 “typed inputs currently fragment APC semantics”，它就有继续价值。

### 5.3 对 `PromptABIKV`

先做：

1. trace 中 APC miss 的 breaker 分类
2. 有多少 miss 实际来自模板漂移 / 工具顺序 / schema 动态字段
3. 规范化后 hit rate 和 TTFT 能提升多少

如果 miss 的主要来源不是 ABI drift，这条线就应该保持配套定位。

## 6. 当前推荐的推进策略

最稳妥的下一步不是立刻押注其中一个，而是：

1. 用最快速度验证 `ResidualModeKV` 的 profiling 假设
2. 同步做一个小规模 `TypedPrefixKV` 事实核查
3. 暂时把 `PromptABIKV` 当成 trace/diagnostics 需求收集

如果 `ResidualModeKV` 收益明显，就先推进它。

如果 `ResidualModeKV` 收益一般，但 `TypedPrefixKV` 能明显扩展 APC 覆盖面，就转向 `TypedPrefixKV`。

## 7. 当前最终判断

这一轮之后，我不再认为“继续发明新的 prefix eviction 策略”是最佳路线；更有希望留下来的是：

- 命中之后的执行模式
- 异构输入上的精确前缀语义
- 应用到 runtime 的 cacheability 契约

这三条线更贴近 vLLM 当前真实缺口，也更容易继续往系统论文叙事上收敛。
