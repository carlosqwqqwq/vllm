# `prompt_logprobs selective replay` 实现状态

更新日期：2026-04-15

## 1. 当前结论

`FalseNegativeKV` 的第一个主方法已经完成最小可运行实现：`prompt_logprobs selective replay` 现在不再只是设计文档，而是已经在 `vLLM` runtime 中打通了从请求建模、调度决策到 worker sidecar 构建与回填的完整链路。

当前实现目标不是一次把整篇论文的系统都做完，而是先把主 thesis 的第一条恢复机制变成可运行、可开关、可做实验的 prototype。按照这个标准，当前状态已经达到“可以开始做主方法实验验证”。

## 2. 已落地的运行时能力

### 2.1 请求与调度侧

已经实现以下内部元数据：

- `original_skip_reading_prefix_cache`
- `prompt_logprobs_sidecar_key`
- `prompt_logprobs_use_sidecar`
- `prompt_logprobs_replay_start_token`

调度器已经具备以下行为：

- 对默认会 `skip_reading_prefix_cache` 的 `prompt_logprobs` 请求，先根据 `sidecar key` 判断是否具备恢复条件
- 若具备恢复条件，则临时允许 prefix lookup
- 对命中的物理前缀执行 `one-block boundary replay`
- 把 `physical hit` 裁剪为 `effective reuse`
- 将 `replay_start_token` 通过 `NewRequestData` 传给 worker

### 2.2 worker 侧

已经实现以下能力：

- 完整 `prompt_logprobs` 结果自动固化为 CPU sidecar
- sidecar 以 `prompt_logprobs_sidecar_key` 为键缓存
- consumer 请求进入 worker 时，若被调度器标记为 selective replay，则先把 sidecar clone 到 `in_progress_prompt_logprobs_cpu`
- 后续 replay 区间继续沿用现有 `_get_prompt_logprobs_dict` 逻辑覆盖写回，不需要另起一套 prompt-logprobs 聚合路径

### 2.3 benchmark / ablation 开关

已经打通以下环境开关：

- `VLLM_ENABLE_PROMPT_LOGPROBS_SELECTIVE_REPLAY`
- `VLLM_ENABLE_PROMPT_LOGPROBS_SIDECAR`
- `VLLM_ENABLE_BOUNDARY_REPLAY`
- `VLLM_MAX_PROMPT_LOGPROBS_REPLAY_TOKENS`

同时，`public_bench/common.py` 已经把这些字段与统一 experiment config 对齐，`preflight_mtbench_fnkv.py` 和 `run_mtbench_fnkv.py` 也不再拒绝这三个开关。

### 2.4 新增 request-level replay trace

现在除了已有的 `engine_*_lookup.jsonl` 之外，还会额外输出：

- `engine_*_replay.jsonl`

这个 trace 专门记录恢复决策后的字段，当前已经包含：

- `physical_hit_tokens`
- `effective_reuse_tokens`
- `replay_start_token`
- `prompt_logprobs_use_sidecar`
- `sidecar_key_ready`
- `prefix_family`
- `backend_compatibility_class`
- `false_negative_risk_class`

`public_bench/common.py` 的 `join_request_rows()` 也已经把这份 replay trace 合并进 joined rows，后续 runner 和分析脚本可以直接消费。

同时，frontend trace 和 lookup / replay trace 现在已经共享第一版 route-key 元数据：

- `prefix_family`
- `api_contract_class`
- `backend_compatibility_class`
- `false_negative_risk_class`

这一步还不是 `TwinPathServe` 的在线路由本身，但它已经把后续 `family binding`、`route / fallback` 和 `backend mismatch` 第二机制需要依赖的统一字段正式接进了 runtime trace。

## 3. 本轮真实验证结果

### 3.1 验证环境

- 容器：`1a17a928e473`
- Python：`/opt/conda/envs/vllm_longbench/bin/python`
- 模型：`/data/models_copy/llama/llama-2-7b-chat`
- GPU：`GPU 2`
  - 选择依据：空闲显存最高（约 `81156 MiB`），利用率 `0%`

### 3.2 先澄清：`W1 prevalence` 不是主方法触发实验

本轮先按 `public_bench/MT-Bench W1` 模板各跑了一次 baseline 和 selective-replay 版。

结果是：

- baseline 版：
  - `logical_hit_tokens=11728`
  - treatment `physical_hit_tokens=0`
  - `false_negative_tokens / logical_hit_tokens = 1.000`
- selective-replay 版：
  - treatment `physical_hit_tokens=0`
  - `false_negative_tokens / logical_hit_tokens = 1.000`

这并不说明主方法无效，而是说明当前 `W1` public-bench runner 固定在测：

- `generation -> prompt_logprobs` 的 false-negative prevalence

而当前 P1 `prompt_logprobs selective replay` 的真实触发前提是：

- 先有一次相同 `prompt_logprobs` 请求建立 sidecar
- 再有第二次相同 `prompt_logprobs` 请求消费 sidecar 并执行 boundary replay

也就是说，`W1 prevalence` 回答的是“问题是否存在”，不是“bridge 是否已经恢复成功”。这两个实验边界必须严格分开。

### 3.3 方法版真实触发验证

本轮新增了一个独立验证脚本：

- `benchmarks/false_negative_kv/run_prompt_logprobs_replay_validation.py`

它固定执行两阶段：

- `seed_prompt_logprobs`
- `probe_prompt_logprobs`

并分别在 `baseline off` 与 `selective replay on` 下运行。

#### baseline off

结果目录：

- `benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_off_gpu2_20260415_210200`

关键结果：

- `probe physical_hit_tokens = 0`
- `probe effective_reuse_tokens = 0`
- `prompt_logprobs_use_sidecar = 0 / 8`
- `sidecar_key_ready = 0 / 8`
- `skip_reading_prefix_cache = true (8 / 8)`
- `false_negative_reason = prompt_logprobs_skip (8 / 8)`

#### selective replay on

结果目录：

- `benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_on_gpu2_20260415_210240`

关键结果：

- `probe physical_hit_tokens = 8704`
- `probe effective_reuse_tokens = 8576`
- `prompt_logprobs_use_sidecar = 8 / 8`
- `sidecar_key_ready = 8 / 8`
- `skip_reading_prefix_cache = false (8 / 8)`
- `false_negative_reason = no_false_negative (8 / 8)`
- probe 平均 phase wall time 从 `0.7521s` 降到 `0.2021s`
  - 相对下降约 `73.1%`

这说明主方法已经在真实 runtime 中完成了我们要的闭环：

- 默认路径下第二次 `prompt_logprobs` 请求仍是 false-negative miss
- 开启主方法后，同一类请求被恢复成 exact-prefix reuse
- 恢复不只是 trace 字段变化，而是同时体现在 `physical_hit_tokens`、`effective_reuse_tokens` 和端到端 phase 时间上

### 3.4 机制级消融

为了回答“是不是随便开几个开关都能得到收益”，本轮又补跑了四组机制级消融，并生成统一报告：

- `benchmarks/false_negative_kv/results/prompt_logprobs_replay_ablation_gpu2_20260415/report.md`

固定 workload 为同一批 repeated `prompt_logprobs` 请求，核心事实如下：

- `baseline_off`
  - probe `physical_hit_tokens = 0`
  - probe `effective_reuse_tokens = 0`
  - probe 平均 phase wall time: `0.7521s`
- `sidecar_only`
  - `selective_replay=false`
  - `sidecar=true`
  - probe `effective_reuse_tokens = 0`
  - probe 平均 phase wall time相对 baseline 变化 `+5.8%`
- `full_replay_budget_25`
  - `sidecar_key_ready = 8 / 8`
  - probe `physical_hit_tokens = 8704`
  - 但 probe `effective_reuse_tokens = 0`
  - probe 平均 phase wall time相对 baseline 变化 `+0.3%`
- `full_replay_budget_256`
  - probe `physical_hit_tokens = 8704`
  - probe `effective_reuse_tokens = 8576`
  - `prompt_logprobs_use_sidecar = 8 / 8`
  - probe 平均 phase wall time相对 baseline 变化 `-73.1%`

这里最关键的两个结论是：

1. 只建立 sidecar 不会自动带来收益，bridge 不是“有 sidecar 就行”。
2. `max_prompt_logprobs_replay_tokens` 是真实约束，不是装饰性开关。

这组实验还额外证明了一个论文里必须讲清楚的点：

- `physical hit` 不等于 `effective reuse`

在 `budget=25` 这组中，调度器已经探测到同样的物理前缀命中，但由于真实需要的 replay token 数是 `26`，最终：

- `physical_hit_tokens > 0`
- `effective_reuse_tokens = 0`
- `prompt_logprobs_use_sidecar = false`

也就是说，系统已经看到了“逻辑上可复用”的物理命中，但由于 bridge operator 的预算不足，仍然不能把这次命中转化为可消费的 observation recovery。这一点对论文里的机制解释非常重要，因为它说明我们的系统不是简单地“允许读 cache”，而是在满足 sidecar 与 replay 预算两个条件时，才把 `physical hit` 变成真正的 `effective reuse`。

### 3.5 方法版冒烟

测试方式：

- 在同一 `LLM` 实例内，顺序发送两次相同的 `prompt_logprobs` 请求
- 方法版开启：
  - `enable_prompt_logprobs_selective_replay=true`
  - `enable_prompt_logprobs_sidecar=true`
  - `enable_boundary_replay=true`

结果：

- 第一次请求：
  - `skip_reading_prefix_cache=true`
  - `physical_hit_tokens=0`
  - `false_negative_reason=prompt_logprobs_skip`
- 第二次请求：
  - `skip_reading_prefix_cache=false`
  - `physical_hit_tokens=1152`
  - `false_negative_reason=no_false_negative`
- 两次请求都成功返回 `prompt_logprobs`

这说明主方法已经成功把“默认必须跳过前缀缓存”的第二次请求，转换成“可以恢复 exact prefix reuse 的请求”。

### 3.6 baseline 对照冒烟

对照方式：

- 关闭上述三个主方法开关
- 其他条件保持不变

结果：

- 第一次请求：
  - `skip_reading_prefix_cache=true`
  - `physical_hit_tokens=0`
- 第二次请求：
  - `skip_reading_prefix_cache=true`
  - `physical_hit_tokens=0`

这与当前 `vLLM` status quo 一致，说明这次观察到的恢复行为来自新实现，而不是测试误判。

### 3.7 本轮回归验证

本轮额外补了一个最小 CPU 回归测试文件：

- `tests/v1/core/test_prompt_logprobs_selective_replay.py`

覆盖三件事：

- `sidecar key` 的生成边界
- `one-block boundary replay` 的规划结果
- 调度器是否真的输出 `engine_*_replay.jsonl`，并把 `physical_hit_tokens` 裁成 `effective_reuse_tokens`

在当前容器里，仓库标准 `pytest` 环境并不完整：

- `vllm_longbench` 环境缺 `pytest`
- 基础 `conda` 环境有 `pytest`，但缺 `torch/tblib`

因此本轮采用等价自检脚本，在真实 `torch + vllm` 环境中逐条执行新增断言；三条断言全部通过。

## 4. 当前实现边界

这次实现是 P1 prototype，不是最终论文系统，当前边界明确如下：

- 只覆盖 token-based `prompt_logprobs`
- 不支持 `prompt_embeds`
- 不支持 `prompt_logprobs = -1` 的 full-vocab sidecar
- 还没有实现 sidecar eviction / memory budget
- 还没有实现 `TwinPathServe` 双池 route
- 还没有实现 backend mismatch 恢复
- 还没有形成完整 correctness regression 测试集

## 5. 现在可以做什么

基于当前状态，后续可以直接进入下面三类工作：

1. 在 `W1 prevalence` 之外，单独跑 bridge-trigger validation
2. 做 `max_prompt_logprobs_replay_tokens`、`boundary replay` 等消融
3. 在此基础上继续扩展 `TwinPathServe` 第二条机制实例化

## 6. 当前最需要补的工程项

如果目标是尽快把实验跑扎实，优先顺序建议固定为：

1. 增加 sidecar memory accounting
2. 增加标准 `pytest` 可执行环境或等价轻量 test harness
3. 把 `W1 prevalence runner` 与 `bridge validation runner` 的结果统一汇总
4. 开始跑公开 workload 上的主实验和消融实验
