# 审稿意见驱动修订记录（2026-04-15）

本文档记录本轮围绕 reviewer 核心质疑所做的主文稿修订，目标不是简单“降格保守”，而是在不虚增 claim 的前提下，把论文主线抬升为更稳的 `correctness-aware reuse control plane` 叙事。

## 1. 已完成修订

### 1.1 “`logical_hit_tokens` 不是 ground truth” 的回应

- 在 [03_FalseNegativeExactHit的测量.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/03_FalseNegativeExactHit的测量.md) 中，把 `logical_hit_tokens` 明确改写为“围绕 compatible reference execution 构造的 `counterfactual reuse oracle`”。
- 在同章中新增配对语义说明，明确 `false-negative ratio` 表示“相对兼容参考执行的丢失比例”，而不是脱离执行条件的全局客观损失。
- 在 [04_契约感知复用抽象.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/04_契约感知复用抽象.md) 中同步把 `logical hit` 的定义从“理论应命中”收紧为“compatible-reference oracle”。

### 1.2 “安全恢复没有被真正证明” 的回应

- 在 [04_契约感知复用抽象.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/04_契约感知复用抽象.md) 的 `SafeReuseContract` 与命题 1 后，显式区分：
  - 理论命题证明的是 `soundness-first` 判定原则。
  - 当前 prototype 还没有完成所有 top-k / rank / logprob 的完整 output-equivalence audit。
- 文稿因此不再把命题 1 写成“现有实现已经被完整证明安全”，而是写成“若谓词被正确、完备、在线验证，则 bridge 决策在逻辑上 sound”。

### 1.3 “统一系统还是两个 patch” 的回应

- 在 [00_题目与摘要.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/00_题目与摘要.md)、[01_引言.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/01_引言.md) 与 [05_TwinPathServe系统设计.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/05_TwinPathServe系统设计.md) 中，把主贡献重新组织为 `correctness-aware reuse control plane`。
- 在 [04_契约感知复用抽象.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/04_契约感知复用抽象.md) 与 [06_实验评估.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/06_实验评估.md) 中补写“共享控制链”的表述：
  - 同一套 request-level instrumentation
  - 同一套 `reuse contract`
  - 同一条 `reuse oracle -> contract gate -> bridge / route -> fallback`
- 这样可以把“两个实现实例”与“一个统一控制面”明确区分开，而不再让读者误以为正文只是在拼两个 feature patch。

### 1.4 “外部效度与 framing 不匹配” 的回应

- 已把正式 `LongBench W1` 结果纳入主文稿：
  - 在 [00_题目与摘要.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/00_题目与摘要.md) 与 [01_引言.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/01_引言.md) 中纳入 `483504 logical-hit -> 0 physical-hit` 的公开长上下文证据。
- 在 [03_FalseNegativeExactHit的测量.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/03_FalseNegativeExactHit的测量.md) 与 [06_实验评估.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/06_实验评估.md) 中，把 `W1` 的公开证据口径从“仅 MT-Bench”扩展为“MT-Bench + LongBench”。

### 1.5 “W5 有 post-hoc labeling 风险” 的回应

- 在 [03_FalseNegativeExactHit的测量.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/03_FalseNegativeExactHit的测量.md) 中明确说明：`W5` 当前是 cross-backend logical evidence，不是 vanilla runtime 的原生 `backend_mismatch` 分类。
- 在 [06_实验评估.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/06_实验评估.md) 中明确写出：`route_reason = backend_mismatch` 是控制面基于独立 reference phase 与 backend metadata 作出的 admission diagnosis，而不是 probe 自己在线报告的 ground truth。

## 2. 当前仍未闭环的硬缺口

- `prompt_logprobs selective replay` 仍缺完整 output-equivalence audit：
  - 需要覆盖 top-k token、rank、logprob 数值一致性，以及长尾 fallback path。
- 仍缺同台 baseline matrix：
  - `vanilla vLLM`
  - `ad hoc per-feature fix`
  - unified `TwinPathServe`
- 仍缺多 benchmark 的统一主实验：
  - `W1` 的 full-method public runner
  - `W5` 之外的更多 workload family
  - `p50/p95/p99 TTFT`、steady-state throughput、overload / fallback rate

## 3. 这轮修订后的主叙事

- 不是“我们已经完整证明当前实现绝对安全”。
- 也不是“我们只有两个零散 patch”。
- 当前最稳的写法应当是：
  - 我们首先提出并测量了 `false-negative exact hit`；
  - 然后把它提升为一层 `correctness-aware reuse control plane`；
  - 最后用两类 prototype bridge 证明这层控制面不是空抽象，而是已经能恢复两类强 mismatch 的第一版系统实例。
