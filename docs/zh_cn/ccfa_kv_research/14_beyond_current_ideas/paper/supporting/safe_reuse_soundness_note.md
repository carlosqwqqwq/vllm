# Safe Reuse 的 Soundness 说明

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

- 把 reviewer 关于“复用是否安全、是否会影响输出、理论能否证明安全”的问题，收束成一份可复用的 supporting note

它不负责：

- 重新定义 `FalseNegativeKV` 的问题空间
- 重新列举 `vLLM` / `SGLang` 的隔离 case
- 直接给出完整数学证明

它面向的使用场景是：

1. 正文中写“正确性与安全性”小节
2. rebuttal 中回答“你们是不是在冒险复用”
3. 内部统一口径：当前论文到底在证明什么、不在证明什么

从现在开始，这份文档的固定目标是：

- 明确 `FalseNegativeKV` 的安全主张边界
- 把主张写成 `soundness`，而不是 `completeness`
- 给出 `prompt_logprobs selective replay` 和 `twin-pool routing` 两类机制的安全论证框架

## 2. 先给严格结论

先给最重要的严格结论：

> 未经契约验证的复用当然可能不安全，也完全可能影响输出结果。  
> 我们的理论并不证明“已有 state 都可以复用”；  
> 它证明的是：**只有当显式 `reuse contract` 满足时，bridge 才允许发生，否则必须 fallback。**

换句话说，本文的安全主张是：

- `soundness`
  - 凡是系统判定为可 bridge 并实际执行 bridge 的 case，都必须满足显式安全条件
- 而不是 `completeness`
  - 不是说所有逻辑上可能 bridge 的 case，我们都能安全识别并复用

这一区分必须在论文中写死。

## 3. 安全不等于 hash 命中

要回答 reviewer 的问题，首先必须把“安全”拆开。

### 3.1 `identity correctness`

第一层是：

- 这份 state 是不是来自“同一个前缀”

这主要由：

- hash
- prefix key
- radix tree / family identity

来回答。

这一层只解决：

- “是不是同一个东西”

它**不能**单独推出：

- “当前请求可以安全消费这份 state”

### 3.2 `consumer legality`

第二层是：

- 当前请求在自己的 API 语义、backend capability、carrier 约束下，能不能合法消费这份 state

这正是 `reuse contract` 要回答的问题。

比如：

- `prompt_logprobs` 需要 prompt-level observation
- 某个 backend class 可能不支持消费 prefix cache
- `input_embeds` 可能不满足 token-id-sensitive consumer 的要求

### 3.3 `observational safety`

第三层才是论文真正该谈的“安全”：

- bridge 执行之后，用户可见输出是否与支持路径 baseline 一致

这里的 baseline 必须是：

- 一个系统原生支持的执行路径
- 而不是想象中的理想系统

因此：

- `hash hit` 不是安全
- `已有 cache entry` 不是安全
- `理论上能省算力` 也不是安全

更准确的说法是：

> 本文讨论的安全，是 bridge 之后的**用户可见语义**是否与 baseline 等价，或者在数值上是否具有明确、受控、可验证的误差界。

## 4. 论文到底证明什么

从现在开始，论文中的安全主张固定为下面这条：

> 本文证明的是 `safe reuse` 的充分条件，而不是所有复用机会的完全识别。

更具体地说：

- 我们允许 `false negative`
  - 本来可以安全 bridge，但系统保守地没有 bridge
- 我们不允许 `unsafe false positive`
  - 本来不应该 bridge，但系统错误地 bridge 了

因此，当前系统的设计哲学必须明确写成：

> We would rather conservatively miss a reusable case than incorrectly reuse an unsafe state.

中文固定口径为：

> 我们宁可错过一个本可安全复用的请求，也不接受一个不安全的请求被错误复用。

## 5. `SafeReuseContract`

为了让上面的 `soundness` 主张可写、可实现、可验证，本文需要一个更明确的契约定义。

对一个请求 `r`，定义：

```text
SafeReuseContract(r) := C1 ∧ C2 ∧ C3 ∧ C4 ∧ C5
```

其中五个条件的含义固定如下。

### 5.1 `C1 PrefixIdentity`

被复用的 prefix state 必须对应于：

- 与当前请求完全一致的 exact prefix
- 或当前研究口径允许的 logical-exact prefix

它排除：

- family 对齐失败
- prefix 本身已变化
- cache identity 不一致

`C1` 是必要条件，但不是充分条件。

### 5.2 `C2 ExecutionCompatibility`

producer state 必须产生于：

- 当前 consumer 支持的 execution / backend compatibility class

如果 backend class 不兼容，则：

- 不得直接消费该 state
- 必须 route 到兼容的执行池，或者 fallback

### 5.3 `C3 CarrierCompatibility`

当前 consumer 所需的输入载体语义必须与 producer 发布的 state 相容。

这意味着：

- token-id-sensitive consumer 不能在仅有 embeds carrier 时假定自己能正确复用
- multimodal carrier mismatch 不能靠“prefix identity 一致”直接掩盖

### 5.4 `C4 ObservationCompleteness`

当前 consumer 需要的 observation 必须被完整覆盖。

例如：

- 如果请求需要 `prompt_logprobs`
- 那么裸 KV 不足以直接返回完整 prompt observations
- 系统必须有 sidecar 或 replay 方案保证 observation 完整

不能接受：

- 只返回一部分 observation
- 只返回“近似可用”的 observation
- 不透明的部分降级

### 5.5 `C5 VerifiedFallback`

只要前四个条件中任一条：

- 不成立
- 无法在线验证
- 成本超预算

系统就必须：

- 回退到 baseline 支持路径

不允许：

- 模糊降级
- 部分 bridge
- 猜测性继续执行

## 6. 这套理论为什么是 `soundness` 而不是 `completeness`

有了 `SafeReuseContract` 以后，本文的主张可以写成一个条件性命题：

> 如果一个请求满足 `SafeReuseContract`，并通过对应 bridge 执行，那么 bridge 的用户可见输出与 baseline 支持路径等价；  
> 如果 `SafeReuseContract` 不能被建立，则系统不得 bridge，必须 fallback。

这个命题天然是单向的。

它没有说：

- 所有 logical hit 都能被系统识别出来
- 所有 bridgeable miss 都一定能被系统恢复

它只说：

- 一旦系统决定 bridge
- 这个 bridge 必须已经满足安全前提

因此，论文里必须主动强调：

1. 我们证明的是 `soundness`
2. 我们不证明 `completeness`
3. 当前系统可以故意保守

## 7. `prompt_logprobs selective replay` 的安全论证

这是当前最适合写成准证明的 case。

### 7.1 baseline 是什么

对 `prompt_logprobs` 而言，baseline 不是：

- 任意理论上的最优系统

而是：

- 当前 `vLLM` 原生支持的 `full prefill + full prompt observation` 路径

这条 baseline 很重要，因为它明确了：

- 我们比对的是一个已有、可运行、可验证的支持路径

### 7.2 为什么“直接读 prefix cache”不安全

如果只恢复 prefix cache read：

1. 命中的 prefix tokens 不会在本次 request 中重新 forward
2. worker 就不会为这些位置 materialize prompt logprobs
3. 返回结果会缺失整段 prompt observation

因此：

- “裸 KV + prompt_logprobs” 本身不是安全组合

### 7.3 为什么必须有 sidecar

如果想跳过重算 prefix，又想保持 prompt observations 完整，就必须有一个额外对象来补齐：

- 命中边界之前的 prompt observations

这就是 `observation sidecar` 的作用。

sidecar 的安全要求不是“看起来像”，而是：

- 它在覆盖区间上与 full-prefill baseline 的 observation 一致

### 7.4 为什么必须 replay 边界

即便 sidecar 覆盖了命中 prefix 的已有 observation，仍然还有一个关键难点：

- 跨命中边界的第一项 prompt observation

它依赖于：

- 边界前一个 token 的 hidden state

因此只读 sidecar 仍然不够。  
这就是为什么 `boundary replay` 不是优化技巧，而是 correctness requirement。

### 7.5 `prompt_logprobs` 的安全命题

对一个 token-only 请求 `r`，若满足：

1. 当前请求与 producer 请求共享 exact prefix
2. prefix state 来自与当前 consumer 兼容的 execution class
3. sidecar 在 `[0, replay_start)` 区间与 baseline 的 prompt observations 完全一致
4. 从 `replay_start` 到 prompt 末尾，系统执行与 baseline 相同的 prefill / logits / logprob path
5. 若上述任一条件不成立，则回退到 full prefill

则 bridge execution 返回的 prompt logprobs 与 full-prefill baseline 在用户可见语义上等价。

### 7.6 这个安全主张在工程上怎么验证

至少应验证下面四类指标：

1. prompt-logprob 条目数完全一致
2. 各位置 top-k token ids 一致
3. 各位置 rank 一致
4. logprob 数值在可接受误差 `ε` 范围内一致

如果使用 deterministic mode 或相同 kernel setting，则可以进一步要求：

- 完全一致的输出 token

## 8. `twin-pool routing` 的安全论证

这个 case 的安全性逻辑与 `prompt_logprobs` 不同。

### 8.1 我们不证明什么

我们**不**证明：

- 跨不兼容 backend class 直接共享 state 是安全的

这条 claim 当前不应出现。

### 8.2 我们证明什么

`TwinPathServe` 的安全性来自：

- 不在不兼容 backend 上强行消费既有 state
- 而是通过 route 把请求送到支持该 contract 的执行池

更具体地说：

1. `throughput_optimized_pool`
   - 可以保留 aggressive fast path
   - 但不要求它无条件支持 prefix reuse
2. `reuse_optimized_pool`
   - 承担 exact-prefix reuse 友好的执行路径
3. route/fallback
   - 如果请求需要命中保护，就送到兼容 pool
   - 如果兼容 pool 不可用，就 fallback

因此，这里的安全结论是：

> 我们通过避免 unsupported cross-backend consumption 来保证安全，而不是证明任意跨 backend 共享都安全。

### 8.3 这类安全性的固定写法

最稳的写法是：

- routing safety
- execution compatibility safety
- no unsafe cross-backend reuse

最不该写的说法是：

- we prove cross-backend state sharing is safe
- we directly reuse incompatible backend state

## 9. 当前不能声称已被理论证明安全的 case

下面这些 case 当前最多只能说：

- 它们证明问题存在
- 它们支持 `reuse contract` 抽象
- 但当前还不能声称已有安全 bridge

这些 case 包括：

1. `prompt_embeds` / multimodal carrier mismatch
2. 更一般的 `token_embed` / `token_classify`
3. `partial prefill + CLS/MEAN pooling`
4. 更复杂的 cross-backend / cross-carrier 在线 bridge

原因通常至少缺一样：

- 明确 sidecar 定义
- 明确 composition rule
- 明确在线验证 predicate
- 明确 fallback 边界
- 明确 baseline

因此，论文必须把 claim 边界写清：

- 当前 `P1` 只对 `prompt_logprobs selective replay` 和 `twin-pool routing` 做安全主张
- 其他 case 只作为问题广泛性的 supporting evidence

## 10. reviewer 问“是否会影响输出”，最稳的回答方式

最稳的回答不是：

- “不会”

而是：

> 未经契约验证的复用当然可能影响输出，这正是现有系统保守禁用这些 path 的原因。我们的工作不是否认这种风险，而是把“何时安全、何时不安全”显式化，并只在满足充分条件时执行 bridge；否则系统保持原始行为。

这句话有三个作用：

1. 承认 reviewer 的担心是合理的
2. 不和现有系统保护逻辑对着干
3. 把工作重新定位成“安全边界建模”

## 11. 可直接复用的 reviewer 回答模板

可以直接这样回答：

> 我们并不主张“已有 state 就应该被复用”，更不是“只要命中 hash 就安全”。未经契约验证的复用当然可能影响输出，这也是现有系统在若干路径上保守禁用复用的原因。我们的工作不是取消这些保护，而是把“何时可以安全复用”显式化为 `reuse contract`。  
> 对当前论文的 `P1` 范围，我们只对两类机制做安全主张。第一类是 `prompt_logprobs selective replay`：只有当 exact-prefix identity、execution compatibility、observation sidecar 完备性以及 boundary replay 前提同时满足时，系统才允许 bridge；否则回退到 full prefill。因此这里证明的是与原生 full-prefill baseline 的 observational equivalence，而不是任意启用 cache read。第二类是 `twin-pool routing`：我们并不声称跨不兼容 backend 直接共享 state 是安全的，恰恰相反，我们通过显式 route 把高复用风险请求送入支持复用的执行池，从而避免不安全的 cross-backend consumption。  
> 因而，我们的理论证明不是一个“所有 logical hit 都能安全复用”的完备性结论，而是一个 `soundness` 结论：凡是被系统判定为可 bridge 并实际执行 bridge 的请求，都满足显式契约条件；若条件不满足或无法在线验证，系统一律 fallback。换句话说，我们宁可错过可复用机会，也不接受不安全复用。

## 12. 一句固定总结

如果只保留一句最该复用的话，这句应该固定为：

> `FalseNegativeKV` 证明的不是“已有 state 都可以复用”，而是“安全复用的充分条件”：只有当显式 `SafeReuseContract` 成立时，bridge 才允许发生；否则系统必须 fallback，因此当前安全主张是 `soundness`，不是 `completeness`。
