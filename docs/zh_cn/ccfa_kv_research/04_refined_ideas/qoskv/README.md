# QoSKV：面向低延迟交互场景的前缀缓存优化

## 1. 一句话定位

`QoSKV` 的目标是：让 `vLLM` 的 prefix caching 不只是“理论上能命中”，而是能够在真实交互 workload 下做到 **低延迟、少污染、可观测、稳定收益**。

## 2. 为什么值得单独做成一条论文线

第二轮调研里，我觉得一个很重要的信号是：社区已经不只是抱怨“有没有 prefix cache”，而是在抱怨 **prefix cache 的副作用和不稳定性**。典型表现包括：

- prefix hit 低于其他 inference stack
- 在 asymmetric batch 下出现严重 HoL blocking
- reasoning tokens 污染 APC，形成死缓存
- 缺少 active blocks / prefix blocks 的监控分离
- 热前缀无法被显式 pin 住

这说明 prefix caching 已经进入“第二阶段问题”：

- 第一阶段：能不能做缓存
- 第二阶段：缓存会不会带来新的排队、污染和可观测性问题

而第二阶段问题，正是系统论文很适合切入的地方。

## 3. 论文问题定义

可以把问题定义成：

> Automatic Prefix Caching 在提升复用率的同时，也可能引入缓存污染、热前缀不稳定、异步负载下的头阻塞以及观测盲区。如何在保持 exact prefix reuse 的前提下，让前缀缓存成为一个 latency-safe、pollution-aware、observable 的运行时机制？

这条线的重点不是把命中率堆高，而是把 **缓存收益稳定地转化为用户可见的低延迟**。

## 4. 核心设计建议

## 4.1 机制一：前缀分类

不是所有可缓存前缀都一样。可以把前缀分成几类：

- `persistent`: 应长期保留的系统提示词、模板前缀
- `hot`: 最近高复用的共享上下文
- `ephemeral`: 可缓存但生命周期短的普通前缀
- `dead/unreachable`: 未来几乎不会再命中的链条

这一步非常重要，因为当前 APC 更偏“一视同仁”，而真实工作负载不是这样的。

### 4.2 机制二：污染感知的 cacheability 控制

一些 token 实际上不适合进入 APC 主链，例如：

- reasoning `<think>` token
- 不会出现在后续 prompt 中的中间输出
- 只服务当前回合、不具复用意义的 transient 内容

`QoSKV` 可以显式区分“参与 hash 链的 token”和“只存在于本轮执行但不进入共享缓存的 token”，减少死缓存。

### 4.3 机制三：热前缀保留与柔性 pinning

社区已经有人提出 pinned prefix 的需求。这里不一定非要做死板的强 pin，可以做 **soft reservation**：

- 给 persistent/hot prefixes 预留一定 quota
- 当显存压力增大时，优先回收 ephemeral 前缀
- 对长期高频前缀给更高的保留优先级

这样既能保护关键前缀，又不至于把 cache 全部锁死。

### 4.4 机制四：latency-safe 的 cache-aware scheduling

prefix caching 一个隐蔽问题是：命中本身不等于低时延。若调度器不区分“大请求”和“小请求”的 cache hit 结构，仍然可能发生 HoL blocking。

可以增加几条调度规则：

- 对大 hit-length 的重型请求设置 admission cap
- 为短请求保留快速通道
- 把“命中但仍需长 prefill / decode”的请求与“马上可首 token”的请求区分开
- 在显存压力高时避免让长尾请求占满可复用热块

### 4.5 机制五：把 APC 状态真正暴露出来

如果连 active blocks 和 prefix-cached blocks 都分不清，就很难做 capacity planning，也很难解释 latency regression。

因此 `QoSKV` 应该显式暴露：

- active blocks
- prefix-cached blocks
- dead/unreachable blocks
- hot prefix quota usage
- prefix cache pressure

## 5. 在 vLLM 里怎么落地

## 5.1 为什么兼容性很高

这条线主要改的是现有 APC 的策略与调度协同，不需要引入新 backend，因此兼容性很高。

重点模块包括：

- `vllm/v1/core/kv_cache_manager.py`
  - 前缀分类、cacheability、统计
- `vllm/v1/core/kv_cache_utils.py`
  - hash 链与 block 元信息
- `vllm/v1/core/sched/scheduler.py`
  - latency-safe 的 cache-aware admission / rescue path
- `vllm/v1/metrics/*`
  - 前缀状态指标
- `vllm/v1/request.py`
  - 如果要显式标记某类 token 不参与 APC，需要在请求 token 流语义上做支持

## 5.2 兼容性判断

我给它的兼容性评估是：**很高**。

因为它主要是在当前 prefix cache 主线之上做增强，而不是改变整个运行时边界。

## 6. 这条线最强的创新点怎么写

### 6.1 前缀从“可缓存/不可缓存”变成“分层可管理对象”

这是对 APC 语义的一次系统化增强。

### 6.2 把缓存收益转换成时延收益

不是只谈 hit ratio，而是显式关注：

- TTFT
- P99 TTFT
- HoL blocking
- prefix pollution

### 6.3 让 APC 变得可观测、可解释、可调优

很多系统论文的亮点不只是“更快”，还包括“更可操作”。这条线在生产价值上很强。

## 7. 适合的实验设计

### 7.1 workload

- 短请求和长请求混合突发
- reasoning model 多轮对话
- system prompt 高复用服务
- 高并发对话服务

### 7.2 baseline

- 原生 APC
- 原生 APC + 现有 scheduler
- 仅加入前缀保留，不做调度协同
- 仅加入调度协同，不做污染控制

### 7.3 指标

- TTFT
- P95 / P99 TTFT
- hit ratio
- dead prefix blocks
- hot prefix retention ratio
- cache pressure under burst

## 8. 风险与边界

### 8.1 风险

- 如果只做 pinning 或只做 metrics，会显得像 patch，不像论文
- 必须把“污染控制 + 低时延调度 + 可观测性”整成一个统一故事

### 8.2 边界

这条线不应扩展到：

- 非精确语义的 semantic cache
- 很复杂的跨实例迁移
- 模型压缩或 token 裁剪

应坚持“精确 APC 的系统化增强”。

## 9. 什么时候优先选 QoSKV

如果你更关注：

- 在线服务 latency
- P99 问题
- prefix cache 真实收益不稳定
- reasoning / interactive workload

那它可能比 `TierKV` 更容易快速打出一版有说服力的结果。

## 10. 结论

`QoSKV` 是第二轮调研里新增的重要候选。它比 `TierKV` 更聚焦，比 `SegmentKV` 更通用，也比 `HeteroKV` 更容易落地。如果你想打一条“**prefix cache 第二阶段系统问题**”的路线，它会很适合。
