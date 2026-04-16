# FalseNegativeKV 正式实验报告（GPU2，2026-04-15）

## 1. 实验目的

本轮实验只回答一个问题：

- 在当前已经实现的两条主机制上，`FalseNegativeKV` 到底能带来多大提升。

这里不再重复 thesis 讨论，也不把预实验和正式实验混在一起。本报告固定区分三件事：

1. `W1` 上问题出现得有多普遍；
2. `prompt_logprobs selective replay` 在同工作负载下能恢复多少命中、降低多少延迟；
3. `TwinPathServe` 在公开 `W5` workload 上能恢复多少 backend mismatch 命中、降低多少延迟。

## 2. 运行环境

- 容器：`1a17a928e473`
- Python：`/opt/conda/envs/vllm_longbench/bin/python`
- GPU：`GPU 2`
- 选择依据：`81156 MiB` 空闲显存，利用率 `0%`
- 模型：`/data/models_copy/llama/llama-2-7b-chat`
- 运行模式：`enforce_eager=true`

对应 preflight 报告：

- [MT-Bench W1 Preflight](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w1_preflight_formal_gpu2_20260415/mtbench_w1/preflight_report.md)
- [MT-Bench W5 TwinPath Preflight](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w5_twin_path_preflight_formal_gpu2_20260415/mtbench_w5_twin_path/preflight_report.md)

两份 preflight 都返回 `ready`，没有 blocker。

## 3. 实验集合

### 3.1 `W1` prevalence

用于回答：

- repeated `prompt_logprobs` 在公开 workload 上是不是稳定制造 false-negative miss。

结果目录：

- [mtbench_w1_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w1_formal_gpu2_20260415/mtbench_w1)

### 3.2 `prompt_logprobs selective replay` 同工作负载对比

用于回答：

- 在完全相同的 repeated `prompt_logprobs` workload 上，方法关闭和开启时的端到端差异到底有多大。

结果目录：

- [off baseline](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_off_formal_gpu2_20260415/summary.md)
- [on treatment](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_formal_gpu2_20260415/summary.md)

### 3.3 `W5 TwinPathServe` 公开 workload 对比

用于回答：

- 在公开 `MT-Bench W5` 上，`backend mismatch` 的命中损失有多大；
- `TwinPathServe` 能否把这部分损失恢复回来，并带来稳定的时延收益。

结果目录：

- [mtbench_w5_twin_path_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w5_twin_path_formal_gpu2_20260415/mtbench_w5_twin_path)

## 4. 核心结果

### 4.1 `W1`：问题规模已经足够大

公开 `MT-Bench W1` 结果显示：

- 原始样本数：`80`
- 最终选中请求数：`64`
- family 数：`8`
- `logical_hit_tokens = 11728`
- treatment `physical_hit_tokens = 0`
- `false_negative_tokens / logical_hit_tokens = 1.000`

这说明在当前 `W1` 构造下：

- repeated `prompt_logprobs` 请求不是偶发 miss；
- 它会把本该存在的 exact hit 全部打掉；
- `prompt_logprobs` 是一个足够强的 false-negative 来源。

对应摘要：

- [MT-Bench W1 summary](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w1_formal_gpu2_20260415/mtbench_w1/summary.md)

### 4.2 `prompt_logprobs selective replay`：命中从 0 恢复到稳定可复用

在完全相同的 32-family repeated `prompt_logprobs` workload 上：

- 方法关闭时：
  - probe 平均 `physical_hit_tokens = 0`
  - probe 平均 `effective_reuse_tokens = 0`
  - probe 平均 phase wall time = `2.5772s`
- 方法开启时：
  - probe 平均 `physical_hit_tokens = 1088`
  - probe 平均 `effective_reuse_tokens = 1072`
  - probe 平均 phase wall time = `0.9765s`

由此得到三条直接结论：

1. 命中恢复不是部分恢复，而是从 `0` 提升到了稳定的非零复用；
2. `effective_reuse_tokens / physical_hit_tokens = 1072 / 1088 = 98.5%`，
   说明 selective replay 几乎把物理命中完整兑现成了有效复用；
3. probe 阶段端到端时延从 `2.5772s` 降到 `0.9765s`，
   相当于：
   - 时延下降 `62.11%`
   - 速度提升 `2.64x`

这组结果的学术意义是：

- `prompt_logprobs` 并不是“天然无法复用”；
- 它本质上是 observation mismatch；
- 一个轻量级 sidecar + selective replay 机制，已经足以把这类 false-negative miss 恢复成接近完整的有效复用。

### 4.3 `W5 TwinPathServe`：在公开 workload 上恢复全部 backend mismatch 命中

公开 `MT-Bench W5` 结果显示：

- 原始样本数：`80`
- 最终选中请求数：`64`
- family 数：`8`
- `logical_hit_tokens = 11728`
- baseline throughput `physical_hit_tokens = 5952`
- baseline `false_negative_tokens / logical_hit_tokens = 0.492`
- twin-path backend `physical_hit_tokens = 11728`
- twin-path family-binding `physical_hit_tokens = 11728`
- twin-path backend recovery ratio = `1.000`
- twin-path family-binding recovery ratio = `1.000`

按 phase 平均值看：

- baseline throughput probe
  - 平均 `physical_hit_tokens = 93.00`
  - 平均 phase wall time = `0.6208s`
- twin-path backend probe
  - 平均 `physical_hit_tokens = 183.25`
  - 平均 phase wall time = `0.0910s`
- twin-path family-binding probe
  - 平均 `physical_hit_tokens = 183.25`
  - 平均 phase wall time = `0.0896s`

直接换算后：

- `physical_hit_tokens` 从 `93.00` 提升到 `183.25`
  - 提升 `97.04%`
  - 约 `1.97x`
- backend probe 的 phase wall time 从 `0.6208s` 降到 `0.0910s`
  - 时延下降 `85.34%`
  - 速度提升 `6.82x`
- family-binding probe 的 phase wall time 从 `0.6208s` 降到 `0.0896s`
  - 时延下降 `85.57%`
  - 速度提升 `6.93x`

这说明当前 `TwinPathServe` 的第一版双池 runtime 已经不只是“解释问题”，而是真的把公开 workload 上由于 backend mismatch 丢掉的 exact-prefix hit 恢复回来了。

## 5. 现阶段最重要的结论

如果只保留最重要的三句话，那么本轮正式实验已经证明：

1. `FalseNegativeKV` 指向的问题不是边角现象。
   在 `MT-Bench W1` 上，当前构造下 repeated `prompt_logprobs`
   会造成 `100%` 的 logical-hit 丢失。
2. `prompt_logprobs selective replay` 是有效的主方法，而不是 tracing trick。
   在同 workload 对比里，它把 probe 时延压低了 `62.11%`，
   并恢复了接近完整的有效复用。
3. `TwinPathServe` 的 backend mismatch 路线已经具备公开 workload 证据。
   在 `MT-Bench W5` 上，它把 baseline 中 `49.2%` 的 false-negative
   hit 全部恢复，并把 probe 时延压低了约 `85%`。

## 6. 论文写作时必须保留的边界

这些结果已经足够强，但写论文时仍然必须保留边界，不能夸大：

1. `W1` 的方法收益目前来自 dedicated validation workload，
   而不是已经接进 public runner 的 unified treatment runner。
   因此这条结果应当写成：
   - “同工作负载验证证明方法有效”
   而不是：
   - “公开 benchmark runner 已完成 full-method 闭环”
2. `W5` 的公开结果已经可以作为系统主结果的一部分，
   但 workload family 仍然只覆盖 `MT-Bench`，
   还没有扩展到 `LongBench / BFCL / MTEB`。
3. 当前结论最强的是：
   - 两条主机制都已被证明“有效”
   但还不是：
   - 整个最终论文系统已经在所有公开基准上闭环完成

## 7. 对下一步实验的直接建议

基于这轮结果，下一步最值得做的不是继续发散新 idea，而是继续做两件事：

1. 把 `LongBench` 接进可运行 runner，
   把“频率足够高”与“端到端收益显著”放到同一个公开 benchmark 上闭环；
2. 把 `W1 selective replay` 接进统一 public runner，
   让方法收益不再只停留在 validation runner，而能直接进入论文主评测矩阵。

在这两步完成之前，最稳的论文表述应当是：

- `FalseNegativeKV` 已经有强问题证据；
- 两条主机制都已经被实验验证有效；
- `TwinPathServe` 已经在公开 workload 上展示出强恢复收益；
- 但更大规模的 benchmark 闭环仍在推进中。
