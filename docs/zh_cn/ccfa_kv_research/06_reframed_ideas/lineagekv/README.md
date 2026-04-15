# LineageKV：面向推理与工具调用的可复用前缀语义运行时

## 1. 一句话定位

`LineageKV` 的核心主张是：

> 当前 `vLLM` 的 automatic prefix caching 把“执行历史”近似成了“可复用前缀”，但在 reasoning、tool calling、hidden trace 场景里，这个近似已经开始失真。系统需要一个显式的 `reuse lineage` 运行时，把真正可复用的前缀语义从执行轨迹中投影出来。

## 2. 为什么这是一个新问题

我们前面已经看到，很多旧方案都落在：

- 缓存放哪儿
- 哪些前缀更热
- 如何预取
- 如何调度

但 `LineageKV` 问的是一个更前置的问题：

- **哪些 token 根本不该进入共享 APC 的 lineage**
- **哪些输出虽然参与了当前执行，但不应该成为下一轮共享上下文的一部分**
- **系统如何在 exact 语义下，把“执行视图”和“复用视图”分开**

这和 `QoSKV` 不同。`QoSKV` 偏策略；`LineageKV` 偏语义。

## 3. 动机来自哪里

这个方向不是拍脑袋想出来的，至少有三类现象在共同支持它：

- reasoning token pollution
  - 当前 `vLLM` 社区已经有人指出：thinking tokens 会进入 hash chain，但后续 prompt 并不会再携带它们。
- tool call / hidden trace
  - agent 运行时里，工具调用结果、中间 JSON、调试 trace、框架插入的模板字段，并不都应该成为下一轮共享前缀。
- prefix hit 不稳定
  - 很多时候不是“没有共享内容”，而是“共享内容被不该共享的执行 token 打断了 lineage”。

## 4. 核心洞见

**prefix cache 的正确对象，不应该是“这个请求执行过的全部 token”，而应该是“后续请求会以相同语义再次引用的 prefix lineage”。**

这句话听起来抽象，落到系统里其实就是：

- 一个请求可以有多个 token view：
  - `execution view`
  - `reuse view`
- `execution view` 用于本轮模型计算；
- `reuse view` 用于 APC 哈希、发布、命中和 descendant 管理。

## 5. 系统设计

## 5.1 双视图 token 流

每个请求不再只有一条 `all_token_ids`，而是至少维护：

- `execution_tokens`
  - 本轮真实执行使用的 token 流
- `reusable_tokens`
  - 允许进入共享 APC lineage 的 token 流

两条流在大多数普通聊天请求里完全一致；但在 reasoning / tool / hidden trace 场景里可以分叉。

## 5.2 token class 与 lineage projection

系统把 token 或 token span 标记成几类：

- `public-reusable`
- `ephemeral-execution-only`
- `private-hidden`
- `conditionally-reusable`

然后由 `lineage projector` 把 `execution_tokens` 投影成 `reusable_tokens`。

这里最重要的是：**投影不是近似裁剪，而是服务层语义定义的一部分**。也就是说，用户或上层框架必须能明确声明哪些内容属于未来可引用上下文，哪些内容只是本轮执行细节。

## 5.3 descendant re-rooting

一旦 lineage 被显式投影，就不能再沿用现在“所有后续 block 都挂在唯一 parent hash 链上”的思路。

`LineageKV` 需要引入一个 `re-root` 机制：

- 当某段执行 token 不进入 reusable lineage 时；
- 后续真正可复用的回答 token 需要重新挂到“上一个可复用祖先”之下；
- 这样后续请求才能在不带 hidden trace 的情况下，仍然命中回答部分。

这一步是我认为最有研究味道的点，因为它改变的不是 eviction policy，而是 APC lineage 的结构定义。

## 5.4 lineage-aware metrics

`LineageKV` 不能只输出传统 `prefix_cache_hits`，还需要显式暴露：

- `projected_reusable_tokens`
- `projected_away_tokens`
- `re_rooted_descendants`
- `dead_lineage_avoided`
- `useful_hit_ratio`

否则系统很难证明自己不是“看起来更复杂，但其实只是换了几个统计口径”。

## 6. 为什么它和已有工作不一样

和 `HotPrefix / CachedAttention / Bidaw / TierKV` 不同：

- 它不先问“缓存该留在哪一层”，而先问“哪些状态才应当被缓存”。

和 `SegmentKV / Prompt Cache` 不同：

- 它不试图复用任意共享段，而是继续坚持 exact prefix reuse；
- 但它重写了 prefix 的语义边界。

和 `QoSKV` 不同：

- 它不是在当前 APC 之上修 policy；
- 它是在重定义 APC 的 lineage 语义。

## 7. 为什么它能显著增强 vLLM

如果做成，`vLLM` 会多出一类之前没有的能力：

- 可以正确服务 reasoning model，而不让 hidden trace 污染共享 APC；
- 可以更自然地接入 agent / tool-calling 框架；
- 可以把“是否进入共享 cache”从隐式副作用变成显式运行时契约。

这不是简单的“性能更快一点”，而是**APC 终于和现代 LLM 应用语义对齐了**。

## 8. 在 vLLM 里怎么落地

最自然的改动点包括：

- `vllm/v1/request.py`
  - 从单一 token 流扩展到 execution/reuse 双视图
- `vllm/v1/core/kv_cache_utils.py`
  - 扩展 block hash 与 parent linkage 的生成规则
- `vllm/v1/core/kv_cache_manager.py`
  - 支持 lineage-aware 命中与发布
- `vllm/entrypoints/openai/*`
  - 接 reasoning parser、tool call 元数据或新的 API hint
- `vllm/v1/metrics/*`
  - 暴露 lineage 相关指标

## 9. 论文价值与风险

### 9.1 价值

- 问题足够新，和旧方案重合较小
- 能直接解释当前 reasoning / tool workload 下 APC 失真的根因
- 对 `vLLM` 是自然增强，而不是外挂系统

### 9.2 风险

- 需要非常小心地定义“exact”边界
- 必须说明投影规则来自哪里，不能给人一种“偷偷改 prompt 语义”的感觉
- 评审会质疑这是不是 application-layer 问题，所以论文里必须把它提升成 serving runtime semantics mismatch

## 10. 我的判断

如果你希望找一个：

- 比 `QoSKV` 更不容易撞相关工作
- 比 `SessionKV` 更容易先做出原型
- 又明显不同于普通 cache policy

的方向，`LineageKV` 很值得继续往前推。
