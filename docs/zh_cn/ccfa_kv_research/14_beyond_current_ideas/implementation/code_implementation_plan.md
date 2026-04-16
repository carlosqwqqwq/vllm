# FalseNegativeKV 代码实现推进计划

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责把 `FalseNegativeKV / TwinPathServe` 从“有设计文档”推进到“有明确代码施工顺序”。

它回答四个问题：

1. 先写哪些代码
2. 哪些代码属于 runtime 主线，哪些属于实验支撑
3. 消融实验的开关应该怎么设计
4. 实验设置代码应该怎么组织，避免每个 runner 自己长一套参数

它不负责：

1. 替代 `prompt_logprobs_selective_replay_design.md`
2. 替代 `twin_path_serve_minimal_architecture.md`
3. 提前承诺 P1 一定有正结果

## 2. 当前实现总原则

从现在开始，代码推进固定遵守下面四条原则：

1. **先搭统一实验配置层，再写 runner**
2. **先做 measurement / prevalence 路线，再做 recovery**
3. **所有主功能都必须有显式 ablation 开关**
4. **实验设置必须代码化，不允许把主结果建立在手改脚本参数上**

第四条尤其重要。后面所有公开 benchmark runner 都必须共享一套配置 schema、override 机制和输出目录约定，否则实验会很快失控。

## 3. 代码分层

从现在开始，代码层固定分成三层：

### 3.1 runtime 层

位置：

- `vllm/`

负责：

- instrumentation
- `prompt_logprobs selective replay`
- `TwinPathServe` admission / route / bridge

### 3.2 benchmark runner 层

位置：

- `benchmarks/false_negative_kv/`

负责：

- 公共 benchmark ingestion
- family 构造
- 实验发请求
- 汇总 JSONL traces

### 3.3 experiment config 层

位置：

- `benchmarks/false_negative_kv/public_bench/`

负责：

- 统一配置 schema
- ablation 开关
- 统一 override 机制
- 输出目录 / trace 目录组织

这三层里，**experiment config 层必须先落地**，否则 runner 和 runtime 的开关会越来越乱。

## 4. 固定实现批次

## 4.1 Batch 0：统一实验配置骨架

目标：

- 新增统一 experiment config schema
- 固定 benchmark / runtime / ablation / output 四段结构
- 固定 `--config` + `--set key=value` 的覆盖方式

验收：

- 所有后续 runner 都消费同一套 config
- 不再允许每个 runner 单独长一组 argparse 参数

## 4.2 Batch 1：公开 benchmark prevalence runners

优先顺序固定为：

1. `run_mtbench_fnkv.py`
2. `run_mteb_fnkv.py`
3. `run_longbench_fnkv.py`

目标：

- 跑出第一批公开 benchmark prevalence 结果
- 统一写入 request-level JSONL traces
- 统一输出 family-level 聚合结果

验收：

- `MT-Bench`、`MTEB`、`LongBench` 都能在同一配置框架下运行

## 4.3 Batch 2：`prompt_logprobs selective replay` runtime 原型

目标：

- 落地第一个 bridge
- 保持 `W1` 路线可验证

当前状态：

- 已完成最小可运行实现
- 已打通 runtime feature gate、scheduler selective replay 和 worker sidecar cache
- 已完成 baseline / 方法版双请求冒烟对照
- 仍待补 correctness regression、memory accounting 和统一主实验 runner 摘要

验收：

- 有 feature gate
- 有 sidecar 相关内部字段
- 有 correctness regression
- 能在同一 benchmark runner 配置下打开 / 关闭

## 4.4 Batch 3：`TwinPathServe` admission / route 原型

目标：

- 先实现双池 admission
- 不做 request migration

验收：

- route key 可追踪
- family binding 可追踪
- fallback 可追踪

## 4.5 Batch 4：统一 ablation / baseline matrix

目标：

- 让所有 baseline 都能通过配置切换，而不是靠不同脚本

固定 baseline 开关组：

- vanilla
- status quo skip
- ad hoc prompt-logprobs fix
- ad hoc backend fix
- unified P1

验收：

- 同一 benchmark runner 只换 config，就能切 baseline

## 5. 固定 ablation 开关

从现在开始，下面这组字段视为主消融开关组，后续代码和文档都统一用这些名字。

### 5.1 measurement 开关

- `enable_false_negative_instrumentation`

### 5.2 `prompt_logprobs` bridge 开关

- `enable_prompt_logprobs_selective_replay`
- `enable_prompt_logprobs_sidecar`
- `enable_boundary_replay`
- `max_prompt_logprobs_replay_tokens`

### 5.3 `TwinPathServe` 路由开关

- `enable_twin_path_serve`
- `enable_family_binding`
- `enable_backend_mismatch_route`

### 5.4 ad hoc baseline 开关

- `enable_ad_hoc_prompt_logprobs_fix`
- `enable_ad_hoc_backend_fix`

### 5.5 实验模式开关

- `benchmark_name`
- `workload_family`
- `attention_backend`
- `batch_invariant`
- `max_samples`
- `min_group_size`
- `long_prefix_threshold`

这里的原则是：

- runtime 行为用 `enable_*`
- benchmark 选择用 `benchmark_*` 或任务字段
- 数值预算统一用显式数值字段，不用布尔开关代替

## 6. 实验设置代码的固定要求

从现在开始，实验设置代码必须满足下面五条。

1. 主 runner 必须支持 `--config <json>`
2. 主 runner 必须支持多个 `--set dotted.key=value` 覆盖
3. 所有输出目录必须带 `run_tag`
4. trace 目录、聚合目录、日志目录必须由 config 统一派生
5. runner 启动时必须把最终生效配置落盘

不允许：

1. 只靠 shell 环境变量拼出主实验配置
2. 只靠手改 Python 常量控制主实验
3. 同一 baseline 需要换脚本而不是换配置

## 7. 固定文件布局

后续代码文件从现在开始按下面布局推进。

### 7.1 实验配置层

- `benchmarks/false_negative_kv/public_bench/config_schema.py`
- `benchmarks/false_negative_kv/public_bench/config_template.json`
- `benchmarks/false_negative_kv/public_bench/README.md`

### 7.2 benchmark runners

- `benchmarks/false_negative_kv/public_bench/run_mtbench_fnkv.py`
- `benchmarks/false_negative_kv/public_bench/run_mteb_fnkv.py`
- `benchmarks/false_negative_kv/public_bench/run_longbench_fnkv.py`
- `benchmarks/false_negative_kv/public_bench/run_bfcl_fnkv.py`

### 7.3 family builders / analyzers

- `benchmarks/false_negative_kv/public_bench/family_builders.py`
- `benchmarks/false_negative_kv/public_bench/analyze_public_bench_fnkv.py`

### 7.4 runtime 主线

- `vllm/v1/core/prompt_logprobs_sidecar.py`
- `vllm/v1/core/...` 中的 selective replay planner
- `core_client.py` / `request.py` / `scheduler.py` 中的最小 route / bridge 改动

## 8. 代码级验收标准

## 8.1 配置层验收

- 同一 JSON config 能驱动至少两个 runner
- `--set` 覆盖后会把最终配置正确写回磁盘

## 8.2 runner 层验收

- 所有 runner 输出统一目录结构
- 所有 runner 都复用同一 trace schema
- 所有 runner 都支持 baseline / ablation 切换

## 8.3 runtime 层验收

- 每个新功能都有 feature gate
- 关闭 gate 时默认行为不变
- 打开 gate 时输出 trace 足够支持归因与消融

## 9. 当前最优先的下一步

如果现在开始真正写代码，固定优先顺序是：

1. 先落地统一 experiment config schema
2. 再补 `MT-Bench` / `MTEB` 两个公开 benchmark runner
3. 然后再补 `LongBench` runner
4. 之后才进入 selective replay runtime 原型

原因很简单：

- 没有统一 experiment config，后面所有 benchmark runner 都会各写一套
- 没有公开 benchmark runner，后面的 recovery 代码也很难快速进入正文主结果

## 10. 结论

现在不只是“有代码实现计划”，而且这份计划已经固定成：

- 先统一实验配置层
- 再统一公开 benchmark runner
- 再做 runtime bridge
- 最后统一 baseline / ablation matrix

如果后续实现不按这个顺序推进，最可能出的问题不是“写不出来”，而是：

- 开关乱
- baseline 乱
- 实验不可复现
- 消融做不干净
