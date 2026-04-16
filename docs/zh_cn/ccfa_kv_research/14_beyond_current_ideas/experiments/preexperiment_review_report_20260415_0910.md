# FalseNegativeKV 预实验复核报告

## 1. 报告目的

本报告回答三个问题：

1. 这轮预实验到底是怎么做的
2. 原始结果是否和汇总结论一致
3. 当前结果哪些可以直接作为 thesis 证据，哪些还只能保守表述

本报告对应的唯一有效结果目录为：

- `benchmarks/false_negative_kv/results/preexp_gpu2_20260415_0910/`

更早的 `smoke_*` 和 `0906` 之前的目录只用于调试，不作为最终结论依据。

## 2. 最终测试是怎么做的

### 2.1 执行环境

- 仓库：`/data/projects/KVresearch/vllm`
- Python：`/opt/conda/envs/vllm_longbench/bin/python`
- GPU：`CUDA_VISIBLE_DEVICES=2`
- 模型：`/data/models_copy/llama/llama-2-7b-chat`
- 执行脚本：`benchmarks/false_negative_kv/run_all_gpu2.sh`
- 结果标签：`preexp_gpu2_20260415_0910`

实际启动命令为：

```bash
cd /data/projects/KVresearch/vllm
RUN_TAG=preexp_gpu2_20260415_0910 benchmarks/false_negative_kv/run_all_gpu2.sh
```

### 2.2 三个场景分别在测什么

#### `w1_prompt_logprobs`

- 同一个 `prompt family`
- 同一个 backend class
- 同一个缓存池
- 顺序为：
  - `warmup_generation`
  - `control_generation`
  - `treatment_prompt_logprobs`

这个场景测的是：

- 当语义完全相同、逻辑上应复用时
- 仅仅因为 `prompt_logprobs=1`
- `vLLM` 是否把本来可命中的 exact prefix 变成 `skip_reading_prefix_cache`

#### `w5_backend_control`

- backend 固定为 `FLASHINFER`
- 顺序为：
  - `warmup_generation`
  - `control_generation`

这个场景不测 false negative 本身，只建立逻辑参考线：

- 如果相同 family 始终留在 `FLASHINFER` 池内
- 当前前缀到底能拿到多少稳定命中

#### `w5_backend_batch_invariant`

- backend 固定为 `FLASH_ATTN`
- `VLLM_BATCH_INVARIANT=1`
- 只发 `probe_generation`
- 不在本池内 warmup
- 逻辑参考来自 `w5_backend_control`

这个场景测的不是“单池内部 miss”，而是：

- 同一个 `prompt family`
- 若路由到另一个 backend compatibility class
- 原本在参考池中已经可以 exact reuse 的前缀
- 在 probe 池里是否变成 0 命中

因此，`W5` 当前属于 `twin-pool logical false-negative` 近似测量，而不是单引擎内部原生识别的 false-negative reason。

## 3. 我做过哪些修正，为什么只认 `0910` 这轮

### 3.1 早期 prompt 过长

早期版本把共享文本堆得过长，触发了 `Llama-2-7B` 的输入长度校验，结果无效。

最终版把 family prompt 收敛到约 `1098` tokens，保证：

- 仍然是长共享前缀
- 但不会越过上下文限制

### 3.2 早期 family 之间发生串扰

早期版本把 family anchor 放得太靠后，导致不同 family 在同一批请求里共享了过长的公共前缀。

这会带来一个严重问题：

- `w5_backend_batch_invariant` 本来应该测“跨 backend class 的 0 命中”
- 但同一 probe 批次里的其他 family 可能互相借到 prefix hit
- 从而低估真实 false-negative 比例

最终版已把 `Family anchor` 前移到共享文本前面，使每个 family 的 block hash 在开头就分叉。

只有在这个修正完成后得到的 `0910` 结果，才可作为正式基线。

## 4. 复核方法

这轮复核不依赖 `summary.md` 的文字，而是直接回到原始文件做一致性检查。

### 4.1 文件完整性

逐个场景检查以下产物是否齐全：

- `scenario_metadata.json`
- `manifest.jsonl`
- `request_outputs.jsonl`
- `traces/frontend_input.jsonl`
- `traces/engine_0_lookup.jsonl`
- `traces/engine_0_config.jsonl`

结果：

- 三个场景文件全部齐全

### 4.2 manifest / lookup / outputs 对齐性

对三个场景逐个检查：

- `manifest` 是否每条都有对应 `lookup`
- `manifest` 是否每条都有对应 `request_outputs`
- `request_outputs.num_cached_tokens`
  是否等于
  `lookup.physical_hit_tokens`

复核结果：

- `w1_prompt_logprobs`
  - `manifest=24`
  - `lookup=24`
  - `outputs=24`
  - `missing_lookup=0`
  - `missing_output=0`
  - `cached_mismatch=0`
- `w5_backend_control`
  - `manifest=16`
  - `lookup=16`
  - `outputs=16`
  - `missing_lookup=0`
  - `missing_output=0`
  - `cached_mismatch=0`
- `w5_backend_batch_invariant`
  - `manifest=8`
  - `lookup=8`
  - `outputs=8`
  - `missing_lookup=0`
  - `missing_output=0`
  - `cached_mismatch=0`

这说明：

- 汇总不是靠猜
- 每条结论都能回溯到同一条 request 的原始 trace

### 4.3 family 唯一性检查

对三个场景都检查：

- `family_id` 唯一数是否为 `8`
- `prompt_sha1` 唯一数是否为 `8`

复核结果：

- 三个场景都是 `8` 个唯一 family
- 三个场景都是 `8` 个唯一 `prompt_sha1`

这说明最终版已经消除了不同 family 之间的 prompt 串扰。

### 4.4 summary 数值重算

不使用 `summary.md`，直接从原始 JSONL 重算：

#### W1

- 从 `control_generation` 取每个 family 的 `physical_hit_tokens`
- 作为 `logical_hit_tokens`
- 与 `treatment_prompt_logprobs` 的 `physical_hit_tokens` 配对

重算得到：

- `logical_hit_tokens = 8704`
- `physical_hit_tokens = 0`
- `false_negative_tokens / logical_hit_tokens = 1.000`

#### W5

- 从 `w5_backend_control` 的 `control_generation` 取每个 family 的 `physical_hit_tokens`
- 作为跨池逻辑参考
- 与 `w5_backend_batch_invariant` 的 `probe_generation` 配对

重算得到：

- `logical_hit_tokens = 8704`
- `physical_hit_tokens = 0`
- `false_negative_tokens / logical_hit_tokens = 1.000`

与 `summary.md` 完全一致。

## 5. 最终结果解释

### 5.1 W1 是强证据

`w1_prompt_logprobs` 的原始 trace 显示：

- `control_generation` 平均 `physical_hit_tokens = 1088`
- `treatment_prompt_logprobs` 全部 `physical_hit_tokens = 0`
- treatment 的 `false_negative_reason = prompt_logprobs_skip`
- treatment 的 `bridge_candidate = true`
- treatment 的 `bridgeable_class = prompt_logprobs_skip`

这是一个很干净的问题实例化：

- 逻辑上完全相同
- 缓存本身可命中
- 只是 API contract 改变
- 命中被直接打成 0

因此，`W1` 可以作为 `FalseNegativeKV` 的第一类主证据。

### 5.2 W5 也是强证据，但表述必须更谨慎

`w5_backend_control` 的 `FLASHINFER` 池里：

- `control_generation` 平均 `physical_hit_tokens = 1088`

`w5_backend_batch_invariant` 的 `FLASH_ATTN + batch_invariant` probe 池里：

- `probe_generation` 全部 `physical_hit_tokens = 0`

这说明：

- 同一个 family
- 在参考 backend class 中可以稳定复用
- 在另一个 backend class 中完全拿不到命中

但这里有一个重要限制：

- `w5_backend_batch_invariant` 的 engine trace 里
  `false_negative_reason` 仍然是 `no_false_negative`
- 原因不是测错
- 而是当前 runtime 站在这个 pool 内部，只能看到“本池以前没有缓存”
- 它看不到另一个 pool 已经存在的逻辑 exact hit

所以 `W5` 当前证明的是：

- `backend compatibility class mismatch` 会造成强烈的跨池 logical false negative

但它还没有证明：

- 单个 runtime 已经能原生把这种 miss 标成 `backend_mismatch`

这正是下一步 instrumentation 需要补齐的地方。

## 6. 为什么 control 只有 `1088`，不是 `1098`

最终版 prompt 长度约为 `1098` tokens，但 control 命中是 `1088`，这不是异常。

原因有两点：

1. `vLLM` 的 prefix hit 以 block 为粒度
2. 当前 engine config 里的 `block_size = 16`

因此：

- `1098` 不能完整按 16 对齐
- 当前缓存命中只统计到可复用的整块部分
- `1088 = 16 * 68`

这个现象在 `W1` 和 `W5-control` 中都一致，因此不是偶然噪声，而是当前 prefix cache 粒度的正常表现。

## 7. 复核结论

### 7.1 可以确认成立的结论

1. `prompt_logprobs` 确实会把本来可 exact reuse 的前缀打成 0 命中。
2. 该现象不是统计假象，原始 trace、request outputs 和 summary 三者一致。
3. `backend compatibility class` 的切换也会把原本可复用的前缀打成 0 命中。
4. 最终版 `0910` 结果已消除早期的 prompt 过长和 family 串扰问题。

### 7.2 当前必须保守表述的点

1. `W5` 目前是 twin-pool logical false-negative 测量，不是单池内部原生 reason 分类。
2. 因此，`W5` 在论文里现阶段更适合写成：
   - “cross-backend compatibility classes induce logical false-negative misses”
   而不是：
   - “the runtime already classifies these misses online”
3. 如果后续不补跨池逻辑 reference 或 route metadata，审稿人会质疑这是不是只是“两个独立池之间没有共享缓存”的常识现象。

## 8. 下一步建议

如果继续推进这条 thesis，优先级应为：

1. 在 instrumentation 中显式加入 `backend compatibility class`
   和 `logical reference class`
2. 让 `W5` 不仅能离线配对得到 false-negative ratio，
   还能够在线输出 `backend_mismatch` 类别
3. 之后再实现第一个 recovery prototype：
   - `prompt_logprobs selective replay`
4. 再实现第二个系统实例：
   - `TwinPathServe` 的 backend-class twin-pool route

## 9. 最终判断

这轮预实验已经足够支持一个 measurement-first 结论：

- `FalseNegativeKV` 不是空想问题
- 至少存在两类强 false-negative 实例
- 其中 `W1` 已经是原生、直接、干净的系统证据
- `W5` 已经是很强的 twin-pool 方向证据，但还需要把在线归因补完整

因此，当前最合理的研究推进方式不是继续发散新 idea，而是：

- 锁定 `W1 + W5` 作为 measurement 证据
- 继续把 `W5` 从“离线逻辑 false-negative”推进到“在线可观测 backend mismatch”
