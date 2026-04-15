# EligibilityKV：面向复杂模型与请求形态的可验证 cacheability 协议

## 1. 一句话 thesis

`EligibilityKV` 的核心主张是：

> `vLLM` 今天对 prefix cache 的很多启停决策，仍然依赖静态黑名单与保守规则；更好的做法应该是把“当前请求 / 模型 / 后端 / 输入形态是否允许 exact cache reuse”变成一个可验证、可组合、可部分启用的协议。

## 2. 为什么它是一个真实缺口

当前 `vLLM` 里，prefix caching 的支持与否存在大量硬编码边界：

- pooling models with bidirectional attention 不支持 prefix caching；
- hybrid / attention-free / encoder-decoder generative model 不支持 prefix caching；
- 某些模型不支持 `'all' prefix caching`；
- 某些请求类型会通过 `skip_reading_prefix_cache` 主动绕过 APC；
- 某些 backend + batch invariance 组合会强制关闭 APC。

这说明系统当前采用的是：

- “如果不确定安全，就整体禁用”

这种策略对 correctness 是保守的，但它的问题在于：

- 可启用与不可启用之间缺乏中间态；
- 无法把“部分可安全复用”的情况表达出来；
- 新模型、新 backend、新 API 一来，就只能继续堆黑名单。

## 3. 最近邻工作与重复风险

这条线最接近的不是单篇论文，而是若干相邻方向：

- `TypedPrefix` / provider prompt caching
  - 强调什么输入算同一个 exact prefix。
- `TensorRT-LLM Early Reuse`
  - 强调何时可更早共享。
- `CacheSolidarity` 这类多租户安全工作
  - 强调哪些范围允许共享。

但 `EligibilityKV` 与它们的差别在于：

- 它不只讨论 key/hash 或 cache scope；
- 它研究的是 **cacheability decision itself**；
- 它试图把 “能否开 APC” 从静态开关，变成一个 runtime protocol。

截至这轮检索，我**没有找到与“runtime cacheability certificate for APC in vLLM-like runtimes”完全等价的公开系统论文**。

### 当前判断

- **重复风险：中**
- **学术抽象潜力：中高**
- **落地难度：中**

## 4. 我认为可防守的 refined 命题

> 现有推理引擎通常把 prefix cache support 建成一组 hard-coded enable/disable rules。随着模型结构、输入类型、后端和 API 越来越复杂，这种二值化支持矩阵越来越脆弱。我们提出一个 cacheability protocol，把 exact prefix reuse 的适用条件显式建模，并允许部分启用、局部回退和可解释诊断。

## 5. 可能的机制设计

可以把 cacheability 分成若干维度：

- `semantic-safe`
  - 输入语义是否满足 exact match
- `backend-safe`
  - 当前 kernel/backend 是否满足 APC 所需稳定性
- `observable-safe`
  - 当前 API 是否需要额外观测结果
- `share-scope-safe`
  - 当前请求是否允许共享到对应 scope

然后由 runtime 生成一个轻量 certificate：

- 满足全部条件
  - full APC
- 满足部分条件
  - partial APC / selective replay / local-only reuse
- 不满足
  - full recompute

## 6. 为什么它符合你的要求

### 6.1 轻量级

- 第一版主要是协议、metadata、诊断和启用逻辑；
- 不需要从一开始就改 kernel。

### 6.2 普适性

- 不是只绑定某种模型或某个 workload；
- 适用于 `vLLM` 正在快速扩张的多模型、多后端、多 API 环境。

### 6.3 高吞吐 / 低延迟

- 通过“局部可启用”代替“整体禁用”，可以回收原本被保守丢掉的缓存收益；
- 同时减少新模型接入时的性能折损。

## 7. 最大风险

这条线的最大风险是：

- 如果只停留在“更优雅的启用矩阵”，会显得太抽象；
- 必须用真实收益证明它不是 configuration cleanup。

因此它需要强实验来证明：

1. 现有黑名单真的过于保守；
2. certificate 能在保证 correctness 的同时显著扩大 APC 覆盖面；
3. 这件事对吞吐 / TTFT 有可测收益，而不只是更好维护。

## 8. 当前判断

截至 `2026-04-15`：

- **可行性：中高**
- **vLLM 落地性：高**
- **优化效果把握：中**
- **与公开论文撞车风险：中**
- **OSDI 潜力：中**

它比 `ObservationKV` 更抽象，比 `InvariantKV` 更偏协议层，但三者组合起来时，`EligibilityKV` 是那个负责把“何时能开 cache”说清楚的层。
