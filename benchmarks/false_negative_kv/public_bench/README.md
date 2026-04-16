# FalseNegativeKV Public Benchmark Scaffold

这个目录保存 `FalseNegativeKV` 公开 benchmark 路线的统一代码骨架。

当前固定职责：

- 统一 experiment config schema
- 固定 ablation 开关命名
- 为后续 `MT-Bench` / `MTEB` / `LongBench` / `BFCL` runner 提供共享配置入口

当前不负责：

- 实现 runtime bridge
- 取代 `benchmarks/false_negative_kv/` 下已有的预实验脚本

## 当前文件

- `config_schema.py`
  - 统一实验配置 dataclass、校验和 override 逻辑
- `config_template.json`
  - `MT-Bench W1` 的第一版可运行配置模板
- `config_template_longbench.json`
  - `LongBench W1` 的第一版可运行配置模板
- `config_template_bfcl.json`
  - `BFCL W2` 的第一版可运行配置模板
- `config_template_mtbench_w5_twin_path.json`
  - `MT-Bench W5` 双池 `TwinPathServe` 的第一版可运行配置模板
- `common.py`
  - 共享配置加载、输出目录、trace 环境变量和 JSONL 汇总工具
- `data/mtbench_train_rows_payload.json`
  - `MT-Bench train` 的本地公开快照，保证容器离线时也能跑 `W1`
- `data/bfcl/`
  - `BFCL v3 multi-turn` 官方本地快照与 `multi_turn_func_doc`，保证容器离线时也能跑 `W2`
- `bfcl_family_builder.py`
  - `BFCL W2` 的 tool-schema family builder、tool doc registry 和 prompt 构造逻辑
- `preflight_mtbench_fnkv.py`
  - `MT-Bench W1` 的实验前检查，验证依赖、模型路径、数据集访问、tokenizer 和 GPU
- `preflight_bfcl_fnkv.py`
  - `BFCL W2` 的实验前检查，验证本地 BFCL 快照、tool doc 覆盖、tokenizer 选样和 GPU
- `preflight_longbench_fnkv.py`
  - `LongBench W1` 的实验前检查，验证本地 `data.zip`、模型上下文上限、
    tokenizer 选择逻辑和 GPU
- `preflight_mtbench_w5_twin_path.py`
  - `MT-Bench W5` 双池实验前检查，额外验证 `TwinPath` 配置与双池显存预算
- `run_mtbench_fnkv.py`
  - 第一版公开 benchmark prevalence runner，当前固定用于 `MT-Bench W1`
- `run_bfcl_fnkv.py`
  - 第一版公开 benchmark prevalence runner，当前固定用于 `BFCL W2`
- `run_longbench_fnkv.py`
  - 第一版公开 benchmark prevalence runner，当前固定用于 `LongBench W1`
- `run_mtbench_w5_twin_path.py`
  - 第一版公开 benchmark twin-path runner，当前固定用于 `MT-Bench W5`
- `analyze_mtbench_fnkv.py`
  - 对已跑出的 `MT-Bench W1` 结果目录重新聚合并生成 `summary.md`
- `analyze_bfcl_fnkv.py`
  - 对已跑出的 `BFCL W2` 结果目录重新聚合并生成 `summary.md`

## 固定约定

- 后续主 runner 必须支持 `--config <json>`
- 后续主 runner 必须支持 `--set dotted.key=value`
- 后续主 runner 启动时必须把最终配置落盘

## 当前可执行入口

当前已经可以直接执行：

```bash
cd /data/projects/KVresearch/vllm
PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/preflight_mtbench_fnkv.py \
  --config benchmarks/false_negative_kv/public_bench/config_template.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/run_mtbench_fnkv.py \
  --config benchmarks/false_negative_kv/public_bench/config_template.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/preflight_bfcl_fnkv.py \
  --config benchmarks/false_negative_kv/public_bench/config_template_bfcl.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/run_bfcl_fnkv.py \
  --config benchmarks/false_negative_kv/public_bench/config_template_bfcl.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/preflight_mtbench_w5_twin_path.py \
  --config benchmarks/false_negative_kv/public_bench/config_template_mtbench_w5_twin_path.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/run_mtbench_w5_twin_path.py \
  --config benchmarks/false_negative_kv/public_bench/config_template_mtbench_w5_twin_path.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/preflight_longbench_fnkv.py \
  --config benchmarks/false_negative_kv/public_bench/config_template_longbench.json

PYTHONPATH=/data/projects/KVresearch/vllm \
python3 benchmarks/false_negative_kv/public_bench/run_longbench_fnkv.py \
  --config benchmarks/false_negative_kv/public_bench/config_template_longbench.json
```

当前已经有四条固定 runner：

- `run_mtbench_fnkv.py`
  - 只支持 `benchmark_name=mt_bench`
  - 只支持 `workload_family=W1`
  - 支持 `W1` 的 prevalence 测量与 `prompt_logprobs selective replay` 公共 runner 对照
  - 不支持 `TwinPathServe` / `ad hoc fix` 开关被打开
- `run_mtbench_w5_twin_path.py`
  - 只支持 `benchmark_name=mt_bench`
  - 只支持 `workload_family=W5`
  - 固定用于 `backend mismatch` 双池恢复验证
  - 要求开启：
    - `enable_twin_path_serve`
    - `enable_family_binding`
    - `enable_backend_mismatch_route`
  - 不支持混合 `prompt_logprobs selective replay`
- `run_longbench_fnkv.py`
  - 只支持 `benchmark_name=longbench`
  - 只支持 `workload_family=W1`
  - 支持公开长上下文 `W1` prevalence 测量与 `prompt_logprobs selective replay` 对照
  - 当前依赖本地 `LongBench data.zip`
- `run_bfcl_fnkv.py`
  - 只支持 `benchmark_name=bfcl`
  - 只支持 `workload_family=W2`
  - 支持公开 tool-rich / agentic `W2` prevalence 测量与 `prompt_logprobs selective replay` 对照
  - 当前固定基于 BFCL v3 `multi-turn` 官方本地快照，不依赖容器在线拉数

当前还没有落地的公开 runner：

- `W3`
  - 需要 `MTEB / BEIR` 的 token / `prompt_embeds` 双 carrier 对照 runner
- `W4`
  - 需要 `MMBench` 的 multimodal family builder 与 trace runner

两条 runner 当前都默认优先使用仓库内本地快照；若快照不适用，再尝试 Dataset Viewer；最后才回退到本地 `datasets` 路径。

推荐顺序固定为：

1. 先选定 `W1` 或 `W5` 路线
2. 先跑对应 `preflight` 脚本
3. 确认 `preflight_report.md` 为 `ready`
4. 再执行对应 `run` 脚本
