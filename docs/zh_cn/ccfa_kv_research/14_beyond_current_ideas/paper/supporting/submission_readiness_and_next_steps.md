# 论文准备度与下一步

更新日期：2026-04-16

## 1. 这份文档的作用

这份文档只负责一件事：

- 判断当前文稿到底已经能回答哪些 reviewer 问题，以及还有哪些问题不能只靠写作解决

它不负责：

- 代替主文稿
- 给出完整 rebuttal
- 虚构尚未完成的实验闭环

## 2. 先给严格结论

当前这版论文文档已经能够稳定回答下面五类高频 reviewer 问题：

1. 这是不是两个工程 patch 的拼接
2. 这是不是 `hash` / cache key 没打通
3. `vLLM` 为什么会主动隔离这些路径，你们为什么还要复用
4. 这样的复用是否不安全、会不会影响输出、理论是否能证明安全
5. 这是不是 `vLLM` 独有的工程问题，`SGLang` 有没有类似现象

但当前文稿**还不能**诚实声称下面三件事已经被完整证明：

1. `prompt_logprobs selective replay` 的完整 output-equivalence 与统一 public runner 闭环已经完成
2. `W5` 的 `backend_mismatch` 已经在 vanilla runtime 中完成在线原生归因
3. 本文已经完成跨框架统一实验验证

因此，当前状态最准确的判断不是“万无一失、所有问题都已证毕”，而是：

- 文稿层的高频质疑已经基本封口
- 证据层仍有少量必须靠后续实验而不是靠措辞补齐的缺口

## 3. 当前已经锁住的问题

### 3.1 问题表征已经足够稳

主文稿已经把核心问题稳定写成：

- `logical hit` 与 `physical hit` 的脱节
- `false-negative exact hit`
- 不是 hash 失效，而是 consumer contract 没有被安全兑现

这意味着 reviewer 再把论文压回“已有缓存没用上”或“只是 cache key 工程问题”，正文已经有足够强的主回答。

### 3.2 安全边界已经足够清楚

当前主文稿已经明确写出：

- 本文证明的是 `soundness`
- 不是 `completeness`
- `SafeReuseContract` 是 bridge 的前置条件
- 任一条件不成立、无法验证或代价超预算，就必须 fallback

这意味着 reviewer 再问“你们是不是在冒险复用”，正文已经可以给出明确边界：我们不证明“已有 state 都可复用”，只证明“满足契约的 bridge 是安全的”。

### 3.3 一般性回答已经有 supporting 兜底

`SGLang` supporting 文档已经提供了多类 capability boundary 证据，包括：

- `input_embeds`
- multimodal reranker
- speculative / logprob path
- deterministic / backend-specific radix compatibility

这足以支撑一个谨慎但有力的回答：

- 本文的问题不是 `vLLM` 独有的一条工程 if-branch，而更像是现代 serving runtime 的共同结构

## 4. 当前仍然不能写满的地方

### 4.1 `W1` 已经同时具备公开 prevalence 证据与强恢复证据，但还没有 full-method 最终闭环

当前 `W1` 不再只是问题存在的证据。

截至 `2026-04-15`：

- `MT-Bench W1` 已有正式 prevalence 结果
- `LongBench W1` 也已有正式 prevalence 结果
- `prompt_logprobs selective replay` 已在 dedicated validation 中给出强恢复结果

因此现在可以稳妥写成：

- bridge I 已经在真实 runtime 中恢复高 `effective reuse`
- 并带来显著时延收益
- 且该问题在公开短 prompt workload 与公开长上下文 workload 上都已被观察到

但它仍然**不能**被写成：

- 统一 public runner 上的 full-method 闭环已经完成
- 完整 output-equivalence 审计已经证毕

### 4.2 `W5` 仍然要守住 logical evidence 边界

当前 `W5` 在 vanilla runtime 侧的证据类型仍然是：

- cross-backend logical false-negative evidence

而不是 vanilla runtime 已经在线完成：

- runtime 已经在线完成 `backend_mismatch` 分类

但与此同时，`TwinPathServe` 原型侧已经有更强的正式结果：

- `MT-Bench W5` 公开 workload 上的 twin-pool 恢复证据

因此现在最稳的写法不是“`W5` 只能写 logical evidence”，而是：

- vanilla runtime 侧：logical evidence
- twin-path prototype 侧：公开 workload recovery evidence

这条边界如果被写松，reviewer 很容易直接抓住“claim 超过 instrumentation”的问题；但如果把第二条机制继续只写成小样本 smoke，也会低估当前已有证据强度。

### 4.3 cross-framework 仍然是 supporting，而不是主证据

`SGLang` 的 case 很有价值，但当前最稳的写法仍然只能是：

- supporting evidence for generality

不能写成：

- 跨框架主实验已经闭环

## 5. 最优先的下一步

如果目标是继续把论文从“已能答 reviewer”推进到“证据也尽量闭环”，下一步优先级应该固定为：

1. 补 `prompt_logprobs selective replay` 的 output-equivalence 审计，把 top-k token / rank / logprob 数值一致性补齐
2. 把 `W1` 的方法收益接进统一 public runner，并在 `MT-Bench + LongBench` 上做统一 baseline matrix
3. 为 `TwinPathServe` 继续补在线 `route metadata` 与 `backend_mismatch` 原生归因
4. 视投稿目标再决定是否补跨框架实验，而不是过早把 `SGLang` 写成主证据

为避免下一轮推进再次停留在口头约定，这两件最高优先级工作现在已经分别被固化成执行型方案文档：

- [prompt_logprobs_output_equivalence_audit_plan_20260416.md](prompt_logprobs_output_equivalence_audit_plan_20260416.md)
- [../../experiments/unified_baseline_matrix_design_20260416.md](../../experiments/unified_baseline_matrix_design_20260416.md)

## 6. 一句最该复用的判断

如果只保留一句最该复用的话，这句应该固定为：

> 当前论文文稿已经能够稳住“为什么这是一个独立系统问题”以及“为什么我们的复用主张是安全的”这两条主线；真正还需要后续工作补齐的，主要是恢复收益与在线归因的证据闭环，而不是论文叙事本身。
