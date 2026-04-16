# `TwinPathServe` backend mismatch 状态

更新日期：2026-04-15

## 1. 本轮新增了什么

本轮不再只做 measurement，也不再只在单引擎调度器里打 annotation，而是补上了
`backend mismatch` 的第一版真实恢复路径。

当前新增的核心实现有三块：

1. `vllm/v1/core/twin_path_policy.py`
   - 把 `TwinPathServe` 的最小 route policy 从调度器里抽成独立模块
   - 固定支持：
     - `observation_mismatch -> reuse_optimized_pool`
     - `backend_mismatch + known family + reuse_preferred -> reuse_optimized_pool`
     - family binding
     - backend cold-start 回退到 `throughput_optimized_pool`
2. `benchmarks/false_negative_kv/twin_path_runtime.py`
   - 实现第一版真实双池 runtime
   - 由两个独立 `LLM` pool 组成：
     - `reuse_optimized_pool`
     - `throughput_optimized_pool`
   - 外部 runtime 使用同一套 `TwinPathPolicy` 做 admission 决策
3. `benchmarks/false_negative_kv/run_backend_mismatch_twin_path_validation.py`
   - 新增 backend mismatch 专用验证脚本
   - 它不是 measurement-only 预实验，而是主方法的第二条机制验证
4. `benchmarks/false_negative_kv/public_bench/run_mtbench_w5_twin_path.py`
   - 新增 `MT-Bench W5` 公开 benchmark runner
   - 把真实双池 `TwinPathRuntime` 接进统一 `public_bench` 配置入口
5. `benchmarks/false_negative_kv/public_bench/preflight_mtbench_w5_twin_path.py`
   - 新增 `W5` 公开 benchmark preflight
   - 在正式跑双池公开实验前先检查 `TwinPath` 配置和显存预算

## 2. 当前代码状态应该如何诚实描述

到这一步，最准确的说法已经不是：

- `backend mismatch` 只有测量证据，还没有恢复实现

而应该更新为：

- `backend mismatch` 已经具备**最小双池恢复原型**
- 这个原型已经同时具备：
  - validation runtime
  - 第一版 public benchmark runner
- 但它还没有变成常驻服务形态，也还没有接进全部公开 workload family

因此，当前系统状态可以分成三层：

1. core runtime 内
   - 已有统一 route metadata
   - scheduler 内已有共享 policy 的 annotation 路径
   - 但单引擎内仍然不会真的切到双池
2. validation runtime
   - 已有真实双池执行面
   - 已能把 `backend mismatch` 从“可测”推进到“可恢复”
3. public benchmark 主线
   - `MT-Bench W5` 已经接入 `TwinPathRuntime`
   - 但 `LongBench / BFCL / MTEB` 仍未接入

## 3. 本轮 smoke validation 证明了什么

本轮在 `GPU 2` 上跑了一次最小双池 smoke validation：

- 结果目录：
  - `benchmarks/false_negative_kv/results/twin_path_backend_validation_smoke_gpu2_20260415/`
- 模型：
  - `/data/models_copy/llama/llama-2-7b-chat`
- pool 配置：
  - reuse pool: `FLASHINFER`
  - throughput pool: `FLASH_ATTN + batch_invariant=true`
- family 数：
  - `2`

验证流程固定为五步：

1. `reuse_seed`
   - 强制进入 reuse pool
   - 建立 prefix cache
2. `reuse_reference_probe`
   - 再次强制进入 reuse pool
   - 证明 reference pool 内确实存在 exact hit
3. `baseline_throughput_probe`
   - 强制进入 throughput pool
   - 模拟 status quo 的 backend mismatch 路径
4. `twin_path_backend_probe`
   - 用 `TwinPathPolicy` 做第一次真实 backend route
5. `twin_path_family_binding_probe`
   - 再次用 `TwinPathPolicy`
   - 验证 family binding 已经接管后续同 family 请求

## 4. 本轮关键结果

本轮最关键的结果不是绝对耗时，而是路由语义和命中恢复是否成立。

### 4.1 throughput baseline 仍然完全拿不到 hit

`baseline_throughput_probe` 的结果是：

- 平均 `physical_hit_tokens = 0`
- `selected_pool = throughput_optimized_pool`
- `route_reason = forced_throughput_optimized_pool`

这说明：

- 同一个 family 即便已经在 reuse 参考池里形成了 hit 证据，
  throughput-oriented backend class 里仍然不会自然继承这份复用收益

### 4.2 TwinPath 的第一次 backend route 已经恢复命中

`twin_path_backend_probe` 的结果是：

- 平均 `physical_hit_tokens = 1088`
- `route_reason = backend_mismatch`
- `selected_pool = reuse_optimized_pool`

这说明：

- `TwinPathPolicy` 没有只做 tracing
- 它真的把 backend mismatch family 从 throughput 路径重新送回了 reuse pool
- 送回之后，系统确实重新拿回了 exact-prefix hit

### 4.3 family binding 已经成为后续请求的稳定归属

`twin_path_family_binding_probe` 的结果是：

- 平均 `physical_hit_tokens = 1088`
- `route_reason = family_binding`
- `selected_pool = reuse_optimized_pool`

这说明：

- 第一次 `backend_mismatch` 路由不是一次性的硬编码
- family binding 已经能够让后续同 family 请求稳定留在 reuse pool

### 4.4 延迟趋势与命中恢复一致

这次 smoke run 的平均 phase wall time 为：

- `baseline_throughput_probe`: `0.2411s`
- `twin_path_backend_probe`: `0.0881s`
- `twin_path_family_binding_probe`: `0.0725s`

这组数字只能作为方向性证据，不能直接当论文主结果，因为：

- 样本数只有 `2 families`
- 还不是公开 benchmark
- 也还没有统一吞吐测量

但它已经足够证明一个更重要的事实：

- 恢复命中后，时间收益方向与命中恢复是一致的

## 5. 本轮顺手修掉的真实 bug

双池 smoke run 还暴露出一个真实前端 bug：

- `vllm/v1/engine/input_processor.py`
  在构建 route key 时假定 `mm_features` 一定可迭代
- 对普通文本请求，这个字段可能是 `None`

本轮已修复为：

- 文本请求走 frontend route-key 路径时，`mm_identifiers` 正确视为空列表

同时补了一个最小回归测试，避免后续 public benchmark 再次被这个空值问题打断。

## 6. 新增的公开 benchmark 结果

在把 `TwinPathRuntime` 接入 `public_bench` 之后，本轮又补跑了一次
`MT-Bench W5` 小样本公开 benchmark：

- 结果目录：
  - `benchmarks/false_negative_kv/results/mtbench_w5_twin_path_smoke_gpu2_20260415/mtbench_w5_twin_path/`
- 样本规模：
  - `4 requests / 2 families`
- 路由配置：
  - reuse pool: `FLASHINFER`
  - throughput pool: `FLASH_ATTN + batch_invariant=true`

这次公开 benchmark 的关键结果是：

- `reuse_reference_probe`
  - 平均 `physical_hit_tokens = 148`
- `baseline_throughput_probe`
  - 平均 `physical_hit_tokens = 64`
- `twin_path_backend_probe`
  - 平均 `physical_hit_tokens = 148`
- `twin_path_family_binding_probe`
  - 平均 `physical_hit_tokens = 148`

这说明在真实公开 workload 上，throughput 路径并不是“完全零命中”，
而是只能拿到一部分命中；`TwinPathServe` 的 route 和 family binding
则把它重新拉回到了 reuse reference 的命中水平。

按配对汇总统计：

- `logical_hit_tokens = 592`
- baseline throughput `physical_hit_tokens = 256`
- `baseline false_negative ratio = 0.568`
- twin-path backend `physical_hit_tokens = 592`
- twin-path family-binding `physical_hit_tokens = 592`

这比合成 smoke workload 更接近论文里应该呈现的事实：

- backend mismatch 在真实公开请求上可能表现为“部分命中退化”
- `TwinPathServe` 的贡献不是把 `0` 变成非零，而是把退化后的 hit 恢复到
  reference-compatible 路径的水平

## 7. 当前还差什么

虽然第二条机制已经有了最小可运行实现，但还不能直接把它当作论文最终方法完成版。

当前还差三步：

1. 把 `TwinPathRuntime` 继续接入更多公开 benchmark runner
   - 让 `LongBench / BFCL / MTEB` 能直接切换到双池模式
2. 把 baseline matrix 接到同一配置层
   - status quo
   - ad hoc backend fix
   - unified twin-path
3. 把第二机制扩成公开数据集实验
   - 现在只有 `MT-Bench W5` 小样本 smoke run
   - 还没有更大规模 workload-level prevalence-to-recovery 闭环

## 8. 当前可下的结论

截至本轮结束，可以下一个比之前更强、但仍然诚实的结论：

> `FalseNegativeKV` 的第二条主机制已经不再只是 architecture 文档或测量假设。
> 通过第一版真实双池 runtime，我们已经在 `vLLM` 上验证：
> backend mismatch family 可以被 `TwinPathServe` 的 route policy 重新送回 reuse pool，
> 并恢复原本在 throughput 路径中丢失的 exact-prefix hit。

但同时也必须保留边界：

> 当前这已经是带有 `MT-Bench W5` 公开 runner 的最小论文系统原型，
> 但还不是覆盖多 workload family 的最终论文系统。
