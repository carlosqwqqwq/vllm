# FalseNegativeKV 预实验

这组脚本服务于 `FalseNegativeKV` 的第一轮预实验，目标不是直接实现
`TwinPathServe`，而是先回答两个最关键的问题：

1. `prompt_logprobs` 是否会稳定制造 `false-negative hit`
2. `backend mismatch` 是否会稳定制造 `false-negative hit`

当前只实现两条场景：

- `W1`：同一个 prompt family 下，`generation` 对照 `prompt_logprobs`
- `W5`：同一个 prompt family 下，`FLASHINFER` 参考池对照
  `FLASH_ATTN + VLLM_BATCH_INVARIANT=1` probe 池

另外补了一个主方法验证脚本：

- `run_prompt_logprobs_replay_validation.py`
  - 专门验证 repeated `prompt_logprobs` 请求下，`prompt_logprobs selective replay`
    是否真的把第二次请求恢复成可复用命中
- `run_backend_mismatch_twin_path_validation.py`
  - 专门验证真实双池 `TwinPathServe` runtime
  - 检查 `backend mismatch` family 是否能被 route policy 重新送回 reuse pool
    并恢复原本在 throughput pool 中丢失的 exact-prefix hit

## 目录

- `run_preexperiment.py`
  - 跑单个场景并写出 manifest、request outputs 和 JSONL traces
- `analyze_preexperiment.py`
  - 汇总多个场景结果，生成 `summary.md`
- `run_prompt_logprobs_replay_validation.py`
  - 跑 `seed_prompt_logprobs -> probe_prompt_logprobs` 两阶段方法验证
- `run_backend_mismatch_twin_path_validation.py`
  - 跑 `reuse_seed -> reuse_reference_probe -> baseline_throughput_probe -> twin_path_probe`
    的 backend mismatch 双池验证
- `twin_path_runtime.py`
  - 第一版真实双池 runtime 封装，内部维护 `reuse_optimized_pool`
    与 `throughput_optimized_pool`
- `run_all_gpu2.sh`
  - 在 `vllm_longbench` conda 环境、GPU2 上串行跑首轮预实验
- `public_bench/`
  - 公开 benchmark 路线的统一配置层，当前已包含：
    - `MT-Bench W1` prevalence runner
    - `MT-Bench W5` TwinPathServe runner
- [experiments/preexperiment_plan.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/experiments/preexperiment_plan.md)
  - 当前主线中的预实验计划文档

## 输出

每次运行都会在 `results/<timestamp>/` 下生成：

- `<scenario>/scenario_metadata.json`
- `<scenario>/manifest.jsonl`
- `<scenario>/request_outputs.jsonl`
- `<scenario>/traces/frontend_input.jsonl`
- `<scenario>/traces/engine_0_lookup.jsonl`
- `<scenario>/traces/engine_0_config.jsonl`
- `summary.md`

## 运行方式

容器内推荐命令：

```bash
cd /data/projects/KVresearch/vllm
CUDA_VISIBLE_DEVICES=2 \
PYTHONPATH=/data/projects/KVresearch/vllm \
benchmarks/false_negative_kv/run_all_gpu2.sh
```

## 当前实验边界

- `run_preexperiment.py` / `public_bench/` 只做 measurement-first 预实验
- `run_preexperiment.py` / `public_bench/` 不实现 selective replay
- 不实现双池 runtime
- 不覆盖 `prompt_embeds` / multimodal 在线恢复

当前目录里的两类脚本职责不同：

- prevalence 脚本：
  - 回答 false-negative 是否存在、出现多频繁
- validation 脚本：
  - 回答某条 bridge / route 是否已经把 miss 真正恢复成命中

目前已经有两条 validation 路线：

- `run_prompt_logprobs_replay_validation.py`
  - 验证第一条 bridge：`prompt_logprobs selective replay`
- `run_backend_mismatch_twin_path_validation.py`
  - 验证第二条机制：`TwinPathServe` backend mismatch twin-pool route

其中 `W5` 目前采用 twin-pool 近似测量：

- `w5_backend_control` 在 `FLASHINFER` 池内先 warmup 再 probe，建立可复用参考
- `w5_backend_batch_invariant` 在 `FLASH_ATTN + batch_invariant` 池内直接 probe
- 这一步测的是跨 backend compatibility class 的潜在 false negative，而不是单池内部 miss

这意味着当前目录现在同时承担两类职责：

- measurement-first 预实验
  - 证明问题是否真实存在、出现是否稳定
- method validation
  - 证明某条恢复机制是否已经真正跑通

但也必须明确：

- `public_bench/` 现在已经同时包含：
  - measurement-first `W1` runner
  - twin-path `W5` runner
- 但 `LongBench / BFCL / MTEB` 仍然还没有接入 `TwinPathRuntime`
