# PolyStateKV：Profiling 与实验执行计划

## 1. 目标

这份文档的目标不是再重复一遍“应该测什么”，而是把 `experiment_matrix.md` 里的实验矩阵具体化成：

- 哪些 trace 要跑
- 用仓库里哪条 benchmark 路径来跑
- 需要补哪些 wrapper script
- 指标从哪里采
- baseline 具体怎么组织

第一版 profiling 计划只服务一个核心问题：

> 在现实 serving workload 下，是否存在一块足够稳定的区域，使得 online state-form selection 同时优于 `Static-KV`、`Static-Hidden` 和 `Threshold-Demotion`。

## 2. Profiling 原则

### 2.1 先做可解释 workload，再做大规模 workload

第一版不要一上来堆很多真实数据集。  
更稳妥的顺序是：

1. 先跑可控 synthetic trace
2. 再跑 multi-turn 与 ShareGPT 风格 workload
3. 最后补长上下文高压场景

这样便于我们先定位：

- selector 是否真有用
- 收益来自哪里
- 开销出现在哪一段

### 2.2 先把观测链拉通，再追最优数字

第一轮 profiling 的第一目标不是最漂亮的 throughput，而是要回答：

- 何时发生 demotion
- 何时发生 rematerialization
- 哪些请求命中了错误 form
- selector 自身开销占多少

如果这些问题不可见，后面的实验数字很难解释。

## 3. Baseline 与运行模式

第一版固定四个系统变体：

1. `Static-KV`
   - 所有可复用状态一直保留为 `KV_FULL`
2. `Static-Hidden`
   - 所有可降级状态一律转成 `HIDDEN_CKPT`
3. `Threshold-Demotion`
   - 仅按固定阈值做 demotion
4. `PolyStateKV`
   - online selector

### 3.1 建议的最小控制接口

为了让 benchmarking 脚本可复用，建议给原型增加一组最小控制接口。  
这组接口不一定要长期暴露成公共 CLI，但 profiling 阶段最好统一起来。

建议新增配置开关：

- `--polystate-enable`
- `--polystate-policy static_kv|static_hidden|threshold|online`
- `--polystate-hidden-store cpu-pinned`
- `--polystate-low-watermark <float>`
- `--polystate-high-watermark <float>`
- `--polystate-log-dir <path>`

如果不想在 `vllm serve` 主 CLI 上加太多参数，也可以在 prototype 阶段先让 `benchmarks/polystate/run_suite.py` 通过环境变量注入：

- `POLYSTATE_ENABLE=1`
- `POLYSTATE_POLICY=online`
- `POLYSTATE_LOG_DIR=...`

但无论选哪种方式，**四个 baseline 必须共享同一套控制入口**，否则后续脚本会变得很难维护。

### 3.2 模型选择边界

第一版 profiling 不应一开始就在很多模型上铺开。  
更稳妥的策略是：

1. 先固定一个已知支持 auxiliary hidden-state 输出路径的模型族
2. 在这个模型上完成全部 trace 与 baseline 对比
3. 只有主命题成立后，再讨论跨模型泛化

原因不是为了偷懒，而是因为 `HIDDEN_CKPT` 的 capture / rebuild 本身就和模型族支持程度有关。  
如果一开始把“模型支持差异”混进实验矩阵，很容易让我们分不清：

- 收益来自 `PolyStateKV`
- 还是差异仅仅来自某个模型没有现成 hidden extraction 支撑

因此第一版建议：

- 主实验固定单一模型族
- 跨模型只做补充性 sanity check

## 4. 进入完整实验矩阵之前的 Phase-0 微基准

在跑四类 trace 之前，建议先做一个极小的 feasibility gate。  
它的目标不是比较四个 baseline，而是先判断 `HIDDEN_CKPT` 这条物理路径是否值得继续投实现成本。

### 4.1 必做的三个微基准

#### `capture_cost`

问题：

- prefill 完成后导出 hidden checkpoint 的额外时间开销有多大

至少记录：

- per-request capture latency
- capture bytes
- capture 发生时的 batch size / prompt len

#### `storage_ratio`

问题：

- `hidden_bytes / kv_bytes` 的比例是否真的足够小

如果 hidden checkpoint 体积并没有显著低于 full KV，这条路线很可能没有继续推进的必要。

#### `rebuild_cost`

问题：

- 从 `HIDDEN_CKPT` 重建 `KV_FULL` 的额外时延是否小于其节省出来的显存收益所对应的吞吐回报

至少记录：

- rebuild latency
- rebuild tokens/s
- rebuild 期间的 GPU memory usage

### 4.2 Phase-0 判死条件

如果 Phase-0 出现下面任一情况，我建议不要直接进入完整实验矩阵：

1. `hidden_bytes` 相对 `kv_bytes` 没有显著下降
2. capture 开销已经明显抬高 prefill 热路径
3. rebuild 开销在短 reuse-distance 场景下基本抵消任何潜在收益

也就是说，完整 trace 实验之前，必须先确认：

> `HIDDEN_CKPT` 至少在物理层面是“更省存储但可接受地可恢复”的。

## 5. Trace 矩阵落成什么

第一版建议固定四类 trace。它们分别覆盖不同的失败模式。

### 5.1 `prefix_hotspot`

用途：

- 测 prefix 高复用、短 reuse gap
- 验证 `Static-Hidden` 是否在本该保留 `KV_FULL` 的热点前缀上吃亏

建议优先使用的现有路径：

- 快速 sanity check：
  - `benchmarks/benchmark_prefix_caching.py`
- 正式表格与结构化结果：
  - `vllm bench serve --dataset-name prefix_repetition`

建议参数维度应按两条入口分别写清：

- 对 `benchmark_prefix_caching.py`
  - `prefix_len`: `256 / 512 / 1024`
  - `repeat_count`: `4 / 16`
  - `input_length_range`: `512:1024 / 1024:2048`
- 对 `vllm bench serve --dataset-name prefix_repetition`
  - `--prefix-repetition-prefix-len`: `256 / 512 / 1024`
  - `--prefix-repetition-suffix-len`: `256 / 512 / 1024`
  - `--prefix-repetition-num-prefixes`: `8 / 32`
  - `request_rate`: `inf / 8 / 32`

期望产出：

- `TTFT p50/p95/p99`
- prefix hit rate
- selector 是否倾向保留热点 `KV_FULL`

### 5.2 `mixed_reuse_distance`

用途：

- 这是 `PolyStateKV` 最重要的主场景
- 让热、温、冷三类 prefix 混在同一条 trace 里
- 观察静态单一形式为何在混合分布下系统性吃亏

现有 benchmark 还没有直接提供这类 trace，因此建议新增：

- `benchmarks/polystate/generate_trace.py`
- `benchmarks/polystate/replay_trace.py`

其中：

- `generate_trace.py`
  - 输出 JSONL trace
  - 每条记录至少包含：
    - `request_id`
    - `arrive_ts_ms`
    - `prompt_len`
    - `output_len`
    - `prefix_group`
    - `reuse_class` (`hot / warm / cold`)
    - `sla_class`
- `replay_trace.py`
  - 按给定到达时间回放请求到 OpenAI-compatible server
  - 输出详细请求级时延与命中信息

建议参数维度：

- 热前缀占比：`20% / 40%`
- warm reuse gap：`5s / 20s`
- cold reuse gap：`60s / 180s`
- 长短上下文比例：`short:long = 7:3 / 5:5`

这是第一版最关键的 trace，因为它决定论文主命题是否成立。

### 5.3 `multi_turn_session`

用途：

- 观察会话式 workload 中的跨轮状态复用
- 验证 prefix 不是一次性热点，而是会经历“热 -> 温 -> 冷”的生命周期

现有路径可以直接复用：

- `benchmarks/multi_turn/benchmark_serving_multi_turn.py`
- `benchmarks/multi_turn/generate_multi_turn.json`

两种输入方式都要留：

1. synthetic conversations
2. ShareGPT 转换后的真实多轮会话

重点关注：

- `ttft_ms`
- `tpot_ms`
- `latency_ms`
- `input_num_turns`
- `approx_cached_percent`

这类 trace 对 `PolyStateKV` 特别重要，因为它天然包含不同 reuse distance 的状态切换。

### 5.4 `long_context_pressure`

用途：

- 测高内存压力下的极端场景
- 验证 `Static-KV` 是否会过早失去状态承载能力

建议优先使用：

- `vllm bench throughput`
- `vllm bench latency`
- `vllm bench serve --dataset-name random`

建议参数维度：

- `input_len`: `4k / 8k / 16k`
- `output_len`: `64 / 128 / 256`
- `num_prompts`: `128 / 512`
- `request_rate`: `2 / 8 / 16`

这类 trace 的目标不是追最优时延，而是观察：

- 显存承压后各策略如何退化
- rematerialization 开销是否把收益吃掉

## 6. 脚本规划

为了把实验矩阵变成真正可反复执行的 suite，建议补一个很薄的 `benchmarks/polystate/` 目录。

建议包含四个脚本：

### 6.1 `generate_trace.py`

职责：

- 生成 `mixed_reuse_distance` 的 JSONL trace
- 固定随机种子
- 输出 trace 元数据

输入：

- `--trace-type mixed_reuse_distance`
- `--num-requests`
- `--hot-prefix-ratio`
- `--warm-gap-s`
- `--cold-gap-s`
- `--seed`

输出：

- `traces/<trace_name>.jsonl`
- `traces/<trace_name>.meta.json`

### 6.2 `run_suite.py`

职责：

- 统一拉起 server
- 统一注入 baseline 配置
- 根据 trace 类型调用现有 benchmark 入口

它应该屏蔽下面这些差异：

- `vllm bench serve`
- `vllm bench throughput`
- `vllm bench latency`
- `benchmark_prefix_caching.py`
- `benchmark_serving_multi_turn.py`

建议最小输入：

- `--model`
- `--variant static_kv|static_hidden|threshold|online`
- `--trace prefix_hotspot|mixed_reuse_distance|multi_turn_session|long_context_pressure`
- `--memory-budget low|mid|high`
- `--seed`
- `--output-dir`

### 6.3 `collect_metrics.py`

职责：

- 收 benchmark JSON
- 抓 `/metrics`
- 拉取 `polystate_events.jsonl`
- 汇总成统一结果目录

输出目录建议长这样：

```text
results/
  <trace>/
    <variant>/
      seed_<n>/
        benchmark.json
        prometheus.txt
        polystate_events.jsonl
        profiler/
        summary.json
```

### 6.4 `summarize_results.py`

职责：

- 聚合多 seed 结果
- 输出论文表格所需的 CSV / Markdown
- 标记 kill criteria 是否被触发

第一版 profiling 要尽量从一开始就让结果目录结构稳定，否则后面很难批量比较。

## 7. 指标从哪里采

### 7.1 现有 benchmark 指标

仓库里已经能直接拿到一批很重要的端到端指标。

#### 来自 `vllm bench serve`

直接使用：

- request throughput
- request goodput
- output throughput
- total token throughput
- `TTFT`
- `TPOT`
- `ITL`
- `E2EL`
- `max_concurrent_requests`

这些已经足够覆盖论文里最关键的 serving 指标。

#### 来自 `vllm bench throughput`

适合离线高负载吞吐对比，且已有：

- `start_profile / stop_profile`
- output JSON

#### 来自 `vllm bench latency`

适合做单批次延迟与 profiler trace：

- average latency
- percentile latency
- warmup / stable iters

### 7.2 现有 runtime 指标采集点

以下采集点应该直接复用，不要另起炉灶：

#### `vllm/v1/metrics/stats.py`

已经有：

- `PrefixCacheStats`
- `PrefillStats`
- `PromptTokenStats`
- `SchedulerStats`

它们分别回答：

- prefix 命中情况
- prefill token 组成
- local compute vs cached tokens
- scheduler 层总体状态

#### `vllm/v1/core/kv_cache_metrics.py`

已经有：

- block lifetime
- idle time
- reuse gaps
- eviction events

这些对分析“为什么 selector 做错决定”非常关键。

#### `vllm/v1/worker/gpu_model_runner.py`

已经有：

- `CUDAGraphStat`

它可以帮助判断 `PolyStateKV` 是否意外改变了 batch padding / cudagraph 命中行为。

### 7.3 第一版必须新增的指标

为了让 `PolyStateKV` 的收益可解释，建议新增三类指标。

#### `PolyStateStats`

放到 `SchedulerStats` 里，至少包括：

- `selector_calls`
- `selector_time_us`
- `demotions_to_hidden`
- `restorations_from_hidden`
- `retained_kv_states`
- `retained_hidden_states`
- `hidden_bytes`
- `wrong_form_decisions`

#### `polystate_events.jsonl`

这是第一版最重要的调试日志，不建议只靠聚合指标。

每次 selector 决策都记一条：

- `ts`
- `request_id`
- `state_key`
- `action`
- `old_form`
- `new_form`
- `kv_bytes`
- `hidden_bytes`
- `pressure`
- `ewma_reuse_gap_s`
- `estimated_materialization_cost_us`
- `reason`

这个日志会在论文写作阶段非常有价值，因为它能支持 case study，而不仅仅是表格数字。

#### 请求级恢复路径标记

建议在请求结果里额外记录：

- 是否命中 `KV_FULL`
- 是否从 `HIDDEN_CKPT` 恢复
- 恢复是否发生在 admission 前还是执行前

否则后面只能看到总时延，却很难解释具体哪条路径导致了时延变化。

## 8. Baseline 运行方案

### 8.1 统一的内存预算

四个 baseline 必须在同一内存预算下比较。  
推荐优先使用两种方式之一：

1. `--kv-cache-memory-bytes`
2. `--num-gpu-blocks-override`

如果只需要粗粒度压测，也可以使用：

3. `--gpu-memory-utilization`

但为了论文可复现性，我更建议：

- 主结果使用 `--kv-cache-memory-bytes`
- 辅助结果使用 `--gpu-memory-utilization`

### 8.2 三档预算

建议每条 trace 固定三档预算：

- `high`
  - full `KV` 基本够用
- `mid`
  - 已经需要做 form selection 才有明显空间
- `low`
  - 逼出极端压力与错误 form 决策

最简单的确定方式是：

1. 先用 stock `vLLM` 跑一轮 `Static-KV`
2. 记录能稳定完成 workload 的最小 `kv_cache_memory_bytes`
3. 以此为基准取：
   - `high = 1.0x`
   - `mid = 0.75x`
   - `low = 0.5x`

### 8.3 每组实验的固定流程

每个 `trace x variant x budget x seed` 建议都走同样流程：

1. 冷启动 server
2. warmup 一轮
3. benchmark 正式运行
4. benchmark 结束后拉 Prometheus 与 `polystate_events.jsonl`
5. 若该组被抽中做深度分析，再补 profiler trace

这样才能避免把 warmup、残留状态和历史命中混进结果里。

### 8.4 最小执行矩阵

第一轮先跑这个矩阵就够：

- trace:
  - `prefix_hotspot`
  - `mixed_reuse_distance`
  - `multi_turn_session`
- variant:
  - `Static-KV`
  - `Static-Hidden`
  - `Threshold-Demotion`
  - `PolyStateKV`
- budget:
  - `mid`
  - `low`
- seed:
  - `0 / 1 / 2`

只要这轮结果已经能回答主命题，就先不要扩 matrix。

## 9. 建议的 benchmark 命令模板

下面给出推荐模板。它们的作用是固定 profiling 入口，而不是要求现在立刻把所有参数都实现完。

### 9.1 在线 serving

先起服务：

```bash
vllm serve ${MODEL} \
  --enable-prefix-caching \
  --kv-cache-memory-bytes ${KV_CACHE_BYTES} \
  --polystate-enable \
  --polystate-policy ${POLICY} \
  --polystate-log-dir ${RUN_DIR}
```

再压测：

```bash
vllm bench serve \
  --backend openai \
  --base-url http://127.0.0.1:8000 \
  --endpoint /v1/completions \
  --dataset-name prefix_repetition \
  --prefix-repetition-prefix-len 512 \
  --prefix-repetition-suffix-len 512 \
  --prefix-repetition-num-prefixes 16 \
  --num-prompts 1000 \
  --request-rate 8 \
  --save-result \
  --result-dir ${RUN_DIR}
```

### 9.2 离线 throughput

```bash
vllm bench throughput \
  --model ${MODEL} \
  --dataset-name random \
  --input-len 4096 \
  --output-len 128 \
  --num-prompts 512 \
  --kv-cache-memory-bytes ${KV_CACHE_BYTES} \
  --enable-prefix-caching
```

### 9.3 单批 latency 与 profiler

```bash
vllm bench latency \
  --model ${MODEL} \
  --input-len 4096 \
  --output-len 128 \
  --batch-size 8 \
  --kv-cache-memory-bytes ${KV_CACHE_BYTES} \
  --profile \
  --output-json ${RUN_DIR}/latency.json
```

### 9.4 Prefix hotspot 补充测试

```bash
python benchmarks/benchmark_prefix_caching.py \
  --model ${MODEL} \
  --enable-prefix-caching \
  --num-prompts 64 \
  --repeat-count 16 \
  --input-length-range 512:1024 \
  --prefix-len 512
```

### 9.5 Multi-turn session

```bash
python benchmarks/multi_turn/benchmark_serving_multi_turn.py \
  --model ${MODEL} \
  --served-model-name ${SERVED_NAME} \
  --input-file benchmarks/multi_turn/generate_multi_turn.json \
  --num-clients 4 \
  --max-active-conversations 16
```

## 10. 判死标准如何映射到 profiling

profiling 结果要直接支持 kill criteria，而不是跑完一堆图表不知道该怎么看。

### 10.1 如果 `Static-KV` 几乎总赢

需要检查：

- `mid / low` 预算是否压得不够
- `mixed_reuse_distance` 是否不够混合
- hidden checkpoint 是否根本没有释放出足够空间

如果这些都排除了，说明 `PolyStateKV` 主命题很可能站不住。

### 10.2 如果 `Static-Hidden` 几乎总赢

需要检查：

- `materialization_cost` 估计是否过高
- `KV_FULL` 保留是否没有带来足够 TTFT 收益
- trace 的热点比例是否太低

如果依然如此，说明“保留热态 full KV”这个子假设可能不成立。

### 10.3 如果 `Threshold-Demotion` 和 `PolyStateKV` 差不多

需要重点检查：

- selector 输入信号是否太弱
- online selector 是否只是在模拟一个固定阈值
- `mixed_reuse_distance` 中是否缺乏足够的状态异质性

这是第一版最需要警惕的失败模式。

## 11. 当前建议的推进顺序

我建议 profiling 按下面顺序推进：

1. 先做 Phase-0 微基准
2. 再打通 `PolyStateStats + polystate_events.jsonl`
3. 先跑 `prefix_hotspot`
4. 再跑 `mixed_reuse_distance`
5. 只有当前两者给出明确信号后，再做 `multi_turn_session`
6. 最后再补 `long_context_pressure`

原因很简单：

- Phase-0 最先回答物理路径是否值得继续
- `prefix_hotspot` 最容易暴露 `Static-Hidden` 的短板
- `mixed_reuse_distance` 最能验证 `PolyStateKV` 的主命题
- `multi_turn_session` 最接近真实服务
- `long_context_pressure` 最适合做边界与极限分析

只要 Phase-0 或前两类 trace 已经不能支撑主命题，就不应继续扩大实验面。
