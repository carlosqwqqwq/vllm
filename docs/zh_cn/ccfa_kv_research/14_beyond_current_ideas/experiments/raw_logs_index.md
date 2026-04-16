# FalseNegativeKV 原始实验日志索引

更新日期：2026-04-15

## 1. 原始实验脚本入口

- Benchmark 目录：
  [benchmarks/false_negative_kv](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv)
- 单场景执行脚本：
  [run_preexperiment.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/run_preexperiment.py)
- 汇总分析脚本：
  [analyze_preexperiment.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/analyze_preexperiment.py)
- GPU2 串行运行脚本：
  [run_all_gpu2.sh](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/run_all_gpu2.sh)

## 2. 当前正式结果目录

正式结果与调试结果统一保留在：

- [results](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results)

当前与论文主线直接相关的正式目录如下：

- [mtbench_w1_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w1_formal_gpu2_20260415)
  - `MT-Bench W1` prevalence 正式结果
- [prompt_logprobs_replay_validation_off_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_off_formal_gpu2_20260415)
  - `prompt_logprobs selective replay` 关闭时的同工作负载基线
- [prompt_logprobs_replay_validation_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_formal_gpu2_20260415)
  - `prompt_logprobs selective replay` 开启后的正式验证结果
- [mtbench_w5_twin_path_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w5_twin_path_formal_gpu2_20260415)
  - `MT-Bench W5` TwinPathServe 正式结果
- [mtbench_w1_preflight_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w1_preflight_formal_gpu2_20260415)
  - `W1` 正式实验前检查
- [mtbench_w5_twin_path_preflight_formal_gpu2_20260415](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/mtbench_w5_twin_path_preflight_formal_gpu2_20260415)
  - `W5` 正式实验前检查
- `smoke_*`
  - 调试用途，不作为论文结论依据

## 3. 当前研究口径

- 论文与主线文档当前优先引用 `2026-04-15` 这轮 formal 结果
- 更早 `preexp_*` 目录保留为预实验与复核参考，不再作为最新主结果
- 如果后续新增实验，统一在本文件继续补充正式结果目录索引
