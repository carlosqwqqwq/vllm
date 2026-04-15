# CapsuleKV：为 exact reusable state 建立可组合胶囊抽象

## 1. 一句话 thesis

`CapsuleKV` 的核心主张是：

> 今天的 exact prefix cache 仍把可复用状态近似成“线性 token 前缀 + hash 链”，但现代输入已经包含 token、prompt embeds、多模态占位、tool/schema、scope 等异构组成因素；系统需要一个可验证、可组合、可调度的 `state capsule` 抽象，来表达 exact reusable state 的最小单位。

## 2. 学术问题定义

今天很多系统在实践中已经承认：

- prompt 不再只是纯文本 token
- cache 命中不再只取决于字符串级相等
- 可复用状态也不只来自“前缀文本”

但 runtime 内部往往仍然用很朴素的对象来表达 cacheable state：

- token block
- hash chain
- prompt module
- output step

这带来了一个更深的问题：

> exact reusable state 的最小对象到底是什么？

如果这个对象定义不清楚，系统就只能在以下几类方案之间摇摆：

- 继续坚持脆弱的线性前缀
- 提升到应用层模块化 prompt
- 退到 backend-agnostic 的输出级 reuse
- 或者做更多 ad hoc extra keys

`CapsuleKV` 研究的是：**可否在 runtime 内部定义一个既保持 exactness、又允许组合与验证的最小状态对象。**

## 3. 为什么它不是旧 idea 换皮

它不是：

- `TypedPrefixKV`
  - 那条线更关心“什么输入算同一个 exact prefix”
- `PromptABIKV`
  - 那条线更关心“应用层如何稳定地产生 cacheable prefix”

`CapsuleKV` 更底层：

- 它讨论 runtime 如何表示 reusable state
- 如何给它附带 exactness certificate
- 如何在多个 capsule 之间组合、排序、隔离和验证

换句话说：

- `TypedPrefixKV` 更像语义判等
- `PromptABIKV` 更像输入契约
- `CapsuleKV` 更像底层对象模型

## 4. 最近邻工作与边界

### 4.1 最近邻

- `Prompt Cache`
  - 用 prompt modules 做 modular attention reuse。
- `Serve Programs, Not Prompts`
  - 把 serving 对象升级为更高层的 inference programs。
- `EPIC / MEPIC`
  - 做 position-independent context caching。
- `StepCache`
  - 做 backend-agnostic 的 step-level reuse 与 selective patching。
- `Don't Break the Cache`
  - 评估 agentic workload 下 prompt caching 的稳定性与 breakage。

### 4.2 它们和 CapsuleKV 的区别

这些工作分别从：

- 应用 schema
- 编程模型
- 位置无关复用
- 输出 patching
- 生产实践评测

等角度切入 reuse 问题。

`CapsuleKV` 的边界则是：

> We ask what the minimal exact-state object should be inside the runtime, before choosing whether to modularize prompts, support PIC, or patch outputs.

它比 Prompt Cache 更 runtime-side，比 Serve Programs 更低层，比 EPIC/MEPIC 更强调 exact-state object，而不是 arbitrary-position caching。

## 5. 为什么这个问题现在值得提出

因为今天的 runtime 已经被迫承认输入与状态的异构性。

以 `vLLM` 为例，prefix cache 的 block hash 已不再只看 token：

- `prompt embeds`
- multimodal hashes
- `cache_salt`
- LoRA 相关信息

这些因素已经在进入 exactness judgement，但它们还没有被提升成正式对象，而更多像是：

- extra keys
- ad hoc metadata
- 特例化补丁

这说明：

> reusable state 的对象模型已经落后于真实输入结构

而这正是 `CapsuleKV` 的切口。

## 6. CapsuleKV 的核心机制

### 6.1 胶囊对象

每个 `state capsule` 至少包含：

- `payload`
  - 它所对应的实际 reusable state
- `exactness certificate`
  - 为什么它能被严格复用
- `scope`
  - 它的共享边界与可见范围
- `order constraint`
  - 它与前后胶囊的组合顺序
- `composition boundary`
  - 它能否与其它胶囊拼接形成更大可复用状态

### 6.2 组合规则

不是任意两个 capsule 都能拼接。

系统需要显式判断：

- 前后顺序是否一致
- scope 是否兼容
- typed carrier 是否兼容
- exactness certificate 是否可继承

### 6.3 runtime 验证

cache 命中不再只是“hash 命中即复用”，而是：

1. 找到 candidate capsules
2. 检查 certificate
3. 验证组合合法性
4. materialize 成可执行状态

这让 reusable state 从“隐式匹配结果”变成“显式可验证对象”。

## 7. 为什么它能在 vLLM 中验证

`vLLM` 很适合作为原型载体，因为它已经有明显的现实抓手：

- block hashing 里已有多种 extra keys
- prompt embeds、multimodal、cache salt 等都已进入 cache correctness 路径
- 社区也已有相关 issue，说明这不是纸上命题

因此，第一版可以很克制：

1. 只选三类 capsule carrier
   - token block
   - prompt embeds block
   - multimodal / salt annotated block
2. 定义最小 certificate
3. 实现最小组合与验证路径

这样就能验证：

- capsule abstraction 是否比 ad hoc extra keys 更清晰
- 是否能提高 APC 覆盖面或降低错误 miss / conservative fallback

## 8. 为什么它符合你的要求

### 8.1 轻量级

第一版不要求改模型，也不要求做语义缓存或近似匹配。

### 8.2 普适性

只要系统支持异构输入或复杂 prefix 组成，这条线就成立。

### 8.3 高吞吐

如果 capsule 能减少错误 miss 和过度保守禁用，就能提高 APC 覆盖率和吞吐。

### 8.4 低延迟

更稳定、更准确的 exact-state reuse 会直接改善 TTFT。

## 9. 最大风险

最大的风险是：如果写不好，它会被看成

- 更复杂的 hashing
- 更复杂的 prefix metadata
- 或者一个 prompt ABI patch

所以这条线必须坚持：

1. 研究对象是 runtime state object
2. 重点是 exactness + composition + verification
3. 不是 prompt engineering，也不是 provider caching guideline

## 10. 当前判断

截至这轮调研，`CapsuleKV` 是一个**新颖度潜力很高，但包装难度也更高**的方向。

它最吸引人的地方在于：

- 它不是继续围绕 token-linear prefix 做修修补补
- 而是直接追问“reusable state 的对象模型是不是定义错了”

如果这个命题成立，它比单纯扩展 typed prefix 更像一个新的系统抽象。

## 11. 关键资料

- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Serve Programs, Not Prompts: <https://arxiv.org/abs/2510.25412>
- EPIC: <https://arxiv.org/abs/2410.15332>
- MEPIC: <https://arxiv.org/abs/2512.16822>
- StepCache: <https://arxiv.org/abs/2603.28795>
- Don't Break the Cache: <https://arxiv.org/abs/2601.06007>
- vLLM Prefix Caching: <https://docs.vllm.ai/en/latest/design/prefix_caching/>

