# Prompt Logprobs Selective Replay Design

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

- 把 `prompt_logprobs selective replay` 从 paper 里的 bridge 名称，收束成 **可实现、可验证、可写进论文** 的最小原型设计

它不负责：

- 扩展到 `prompt_embeds` / multimodal
- 直接定义 `TwinPathServe` 双池的完整在线实现
- 改写 `FalseNegativeKV` 的 measurement taxonomy

从现在开始，这个 bridge 的固定定位是：

- 它是 `FalseNegativeKV` 中 `observation mismatch` 的第一个实例化
- 它服务于 `TwinPathServe` 的 `reuse_optimized_pool`
- 它必须同时对齐：
  - [../paper/supporting/paper_storyline_fnkv.md](../paper/supporting/paper_storyline_fnkv.md)
  - [twin_path_serve_minimal_architecture.md](twin_path_serve_minimal_architecture.md)
  - 下一步 `vLLM` prototype 的最小实现

## 2. 一句话设计

`prompt_logprobs selective replay` 的固定定义是：

> 当请求在逻辑上是 `logical hit`，且系统能拿到该前缀的最小 `prompt_logprobs` observation sidecar 时，
> runtime 保留大部分 prefix KV 复用，只放弃命中边界的最后一个 cache block 并从那里 replay 到 prompt 末尾，
> 从而恢复 `prompt_logprobs` 语义，而不是整段 full prefill。

这句话里的三个关键词从现在开始视为锁定：

- `observation sidecar`
- `boundary replay`
- `full-prefill fallback`

## 3. 当前问题与根因

当前 `vLLM` 在 `prompt_logprobs` 上的 conservative 行为是明确的：

- [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431)
  - 默认把 `prompt_logprobs` 请求变成 `skip_reading_prefix_cache = true`
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:259)
  - 把这类请求标记成 `api_contract_class = prompt_logprobs`
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:266)
  - 把它标成 `instrument_false_negative_hint = prompt_logprobs_skip`
- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:230)
  - 一旦 `request.skip_reading_prefix_cache == true`，prefix lookup 直接返回 `physical_hit_tokens = 0`

这说明当前 miss 的直接原因不是：

- prefix state 不存在

而是：

- runtime 无法在“复用已有 prefix KV”和“返回完整 `prompt_logprobs` observation”之间建立 `reuse contract`

更关键的是，worker 侧 prompt-logprob 生成逻辑目前默认把“KV 进度”和“observation 进度”绑在一起：

- [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:5014)
  - `_get_prompt_logprobs_dict(...)` 用 `request.num_computed_tokens` 推导 prompt-logprob slice 起点
- [logprobs.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/logprobs.py:121)
  - `LogprobsProcessor` 只是把 worker 给出的 prompt-logprob tensors 聚合成最终输出
- [output_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/output_processor.py:333)
  - 输出层默认直接消费已经完整 materialize 好的 prompt logprobs

因此，`prompt_logprobs` 被标成 skip-read，并不是一个孤立的 if 语句问题，而是一个更深的 runtime 假设：

- 只要 prefix token 没有在这次 request 中重新 forward，就没有地方产生它对应的 prompt observation

这正是 `FalseNegativeKV` 里 `observation mismatch` 的标准例子。

## 4. 固定论文口径

这条 bridge 在 paper 中的写法必须与实现边界保持一致。

### 4.1 可以怎么写

在摘要、引言和系统设计里，允许写成：

- `prompt_logprobs selective replay is a contract-aware bridge for observation mismatch`
- `the bridge combines exact-prefix KV reuse with a minimal prompt-observation sidecar and boundary replay`
- `the system avoids full-prompt recomputation for prompt-logprob requests when the reuse contract is satisfied`

### 4.2 不能怎么写

不允许写成：

- `we simply enable APC for prompt_logprobs`
- `we cache hidden activations for prompt observations`
- `we support arbitrary prompt-logprob reuse without fallback`
- `we solve prompt_logprobs false negatives with a single flag flip`

原因很明确：

- 这不是把一个开关从 `false` 改成 `true`
- 这不是 hidden cache / activation restoration
- 这也不是对所有 prompt-logprobs 请求都无条件命中

### 4.3 对齐当前 paper 叙事的固定角色

这条 bridge 在论文里固定承担三层作用：

1. 给 `W1 prompt_logprobs` 提供最直接的 runtime pathology -> recovery 闭环
2. 支撑 `paper_storyline_fnkv.md` 里的第一个 `bridge operator`
3. 为后续 `H3` 提供第一类真正恢复成功的 mismatch，而不是只停留在 measurement

## 5. P1 最小设计原则

原型必须同时满足下面四条：

1. `prompt_logprobs` 语义正确
2. 尽量复用已有 prefix KV
3. 尽量复用已有 `LogprobsTensors -> LogprobsProcessor -> RequestOutput` 管线
4. 对现有 scheduler / worker 的侵入尽可能小

据此，P1 的固定原则是：

- **必须引入最小 sidecar**
- **必须回放命中边界的最后一个 cache block**
- **不能为了这个 bridge 重写通用 prefill 内核**

## 6. 为什么必须是 “sidecar + boundary replay”

这是这份设计文档最关键的判断。

### 6.1 只“允许读 prefix cache”是不够的

如果请求命中了 `k` 个 prefix tokens，而我们只是恢复 cache read：

- 这次 request 不会重新 forward 命中的 prefix tokens
- worker 也就不会为这些位置产生 prompt logprobs
- 最终返回的 `prompt_logprobs` 会缺失一大段 prefix observation

这正是 [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431) 目前默认 skip-read 的原因。

### 6.2 只“读取 sidecar”也不够

即便 sidecar 覆盖了所有命中 prefix 的已有 observation，仍然还差一个跨边界位置：

- 第一个未命中 token 的 prompt logprob

它依赖的是：

- 命中边界前一个 token 的 hidden state

也就是说，**跨命中边界的第一项 prompt observation 无法直接由 sidecar 给出**，因为它已经依赖当前 request 的 suffix。

### 6.3 最小安全做法：回放最后一个命中 block

因此，P1 的最小安全设计固定为：

1. sidecar 负责补齐“命中边界之前”的 prompt observations
2. runtime 主动放弃命中边界的最后一个 cache block
3. 从这个 block 的起点 replay 到 prompt 末尾

这样做的好处是：

- 语义正确
- replay 范围固定、连续、易实现
- 保留了绝大多数 prefix KV 复用
- 可以继续使用现有 block-aligned `allocate_slots` / prefill 路径

从现在开始，`selective replay` 的固定含义就是：

> `sidecar-covered prefix` + `one-block boundary replay` + `normal uncached suffix prefill`

而不是：

- replay 整段 hit prefix
- 维护 hidden-state cache
- 引入 arbitrary-position reuse

## 7. 固定系统抽象

## 7.1 `reuse contract`

对 `prompt_logprobs` 而言，P1 的 `reuse contract` 固定为：

一个 request 只有同时满足下面四条时，才允许把 `logical hit` 恢复成带 observation 的 `physical hit`：

1. 输入 carrier 是 `tokens`
2. 存在 exact prefix hit
3. 存在可消费的 prompt-logprobs sidecar
4. boundary replay 的额外成本在预算内

否则统一 fallback 到 full prefill。

## 7.2 `observation sidecar`

P1 的 sidecar 不是新的通用 cache 层，只是 bridge 所需的最小伴生状态。

它固定只存下面这些内容：

- `num_prompt_logprobs`
- `covered_tokens`
- `logprob_token_ids`
- `logprobs`
- `selected_token_ranks`

也就是说，sidecar 的内部布局直接对齐现有 `LogprobsTensors` 的 CPU 形态，而不是发明新的输出表示。

P1 固定不存：

- hidden states
- attention outputs
- activation cache
- 任意中间层 side state

## 7.3 `boundary replay`

P1 固定把 replay 起点定义为：

```text
replay_start_token = align_down(max(physical_hit_tokens - 1, 0), block_size)
```

对应的固定语义是：

- sidecar 覆盖 `[0, replay_start_token)` 对应的 prompt observations
- runtime 从 `replay_start_token` 开始重新 prefill 到 prompt 末尾

据此再定义两个内部量：

```text
effective_reuse_tokens = replay_start_token
replay_tokens = num_prompt_tokens - replay_start_token
```

这意味着：

- `physical_hit_tokens` 用来描述本来能命中的上界
- `effective_reuse_tokens` 用来描述 P1 原型实际保留的 KV 复用量

## 8. 原型范围锁定

P1 原型只支持下面这个子问题：

- token-only request
- `sampling_params.prompt_logprobs is not None`
- 本地 prefix cache 可命中
- 同 prefix family 已经建立 sidecar

P1 原型明确不做：

- `prompt_embeds`
- multimodal
- cross-request sidecar merge
- 细粒度到 token 级别的非 block 对齐 replay
- sidecar 持久化到磁盘
- 双池路由联调

这意味着下一步 prototype 的目标不是“完整 `TwinPathServe`”，而是：

- 先在单个 cache-friendly 执行路径内，把第一个 bridge 做闭环

## 9. 建议的内部字段与接口

当前阶段不引入任何用户可见 API 变更，只定义内部字段。

### 9.1 feature gate

建议锁定两个内部配置名：

- `enable_prompt_logprobs_selective_replay`
- `max_prompt_logprobs_replay_tokens`

前者决定是否允许把 `prompt_logprobs` 请求从 unconditional skip 改成 bridge candidate。

后者决定：

- 如果 `replay_tokens` 超过预算，则直接 full-prefill fallback

### 9.2 request-level bridge plan

建议在 `Request` 上增加下面这些内部字段：

- `prompt_logprobs_bridge_active`
- `prompt_logprobs_physical_hit_tokens`
- `prompt_logprobs_effective_reuse_tokens`
- `prompt_logprobs_replay_start_token`
- `prompt_logprobs_replay_tokens`
- `prompt_logprobs_sidecar_covered_tokens`
- `prompt_logprobs_bridge_fallback_reason`

这些字段的职责必须分清：

- `physical_hit_tokens`
  - measurement / diagnosis 视角
- `effective_reuse_tokens`
  - scheduler / worker 真正执行的复用边界
- `sidecar_covered_tokens`
  - worker 预填 observation slice 的边界

## 10. 代码落点与最小实现路径

## 10.1 `SamplingParams`：把 unconditional skip 改成 conditional bridge candidate

入口：

- [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431)

当前行为：

- `prompt_logprobs is not None` 时默认 `skip_reading_prefix_cache = true`

P1 目标：

- 当 `enable_prompt_logprobs_selective_replay = false` 时，保持现状
- 当该 gate 打开时，不再让 `prompt_logprobs` 自动变成 unconditional skip
- 改由后续 lookup + bridge planner 决定：
  - 走 selective replay
  - 或 fallback 到 full prefill

固定要求：

- 不能破坏显式传入的 `skip_reading_prefix_cache`
- 不能改变没有开启 feature gate 时的默认行为

## 10.2 `Request`：派生 bridge 标签并承载 plan

入口：

- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:169)
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:259)
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:279)

P1 目标：

- 保留现有 instrumentation tags
- 新增 bridge-plan 内部字段
- 允许请求在 lookup 后同时拥有：
  - `prompt_logprobs_physical_hit_tokens`
  - `prompt_logprobs_effective_reuse_tokens`

固定要求：

- 不改变 `instrument_api_contract_class`
- 不改变 `instrument_false_negative_hint`
- bridge plan 只作为内部执行态，不进入外部 API

## 10.3 sidecar registry：新增一个最小 companion-state 管理器

建议新增一个小模块，而不是把 sidecar 逻辑塞进输出层。

建议文件：

- `vllm/v1/core/prompt_logprobs_sidecar.py`

它只负责：

1. 以 prefix-family key 查询 sidecar
2. 返回 `covered_tokens`
3. 写入 `LogprobsTensors` 对齐格式的 CPU sidecar
4. 跟随 prefix-cache reset 一起清理

P1 的固定 key 选择建议为：

- `request.block_hashes[:num_hit_blocks]`
- 再加上 `num_prompt_logprobs`

这样做的原因是：

- 直接复用现有 exact-prefix 哈希边界
- 不引入第二套复杂前缀指纹协议

## 10.4 `KVCacheManager`：先量出物理命中，再交给 bridge planner 决定是否保留最后一个 block

入口：

- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:230)

当前行为：

- 要么直接 skip-read
- 要么直接返回完整 `physical_hit_tokens`

P1 目标：

- 对 `prompt_logprobs` candidate 先照常做 lookup，拿到 `physical_hit_tokens`
- 然后由 bridge planner 基于：
  - `physical_hit_tokens`
  - `block_size`
  - `sidecar coverage`
  - `max_prompt_logprobs_replay_tokens`
  计算 `effective_reuse_tokens`

固定要求：

- `physical_hit_tokens` 的 measurement 语义不能变
- selective replay 不能改写 `false_negative_instrumentation` 的核心字段定义

## 10.5 `Scheduler`：把“物理命中上界”裁成“实际复用边界”

入口：

- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:627)

这是 prototype 真正决定最小 diff 的地方。

P1 的固定做法是：

1. 先获取 `physical_hit_tokens`
2. 如果 bridge precondition 满足：
   - 计算 `replay_start_token`
   - 把实际保留的本地命中裁成 `effective_reuse_tokens`
3. 只把 `effective_reuse_tokens` 之前的 blocks 当成“已计算”
4. 让常规 prefill 从 `replay_start_token` 开始继续跑

这意味着 scheduler 不需要学会新的执行模式，只需要：

- 在已命中的 blocks 里主动丢掉最后一个 block

这是 P1 里最重要的“最小化侵入”选择。

## 10.6 `GPUModelRunner`：先填 sidecar，再复用现有 prompt-logprob materialization

入口：

- [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:5014)

当前行为：

- `_get_prompt_logprobs_dict(...)` 假设 prompt logprobs 全部来自本次 prefill 产生的 hidden states

P1 目标：

1. 若 `prompt_logprobs_bridge_active = true`
   - 先创建完整长度的 `LogprobsTensors.empty_cpu(...)`
   - 再把 sidecar 对应的前缀 slice 拷进去
2. 随后沿用现有 `_get_prompt_logprobs_dict(...)`
   - 只让 replay 段覆盖剩余 slice
3. 最终仍返回完整的 `prompt_logprobs_dict`

P1 尽量不改的东西：

- `compute_logits(...)`
- `sampler.compute_logprobs(...)`
- `sampler.gather_logprobs(...)`

也就是说，prototype 不要重写 prompt-logprob 的计算逻辑，而是：

- 改“输入 slice 从哪开始”
- 改“结果 tensor 的前缀由谁填”

## 10.7 `LogprobsProcessor` 与 `OutputProcessor`：尽量不改协议

入口：

- [logprobs.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/logprobs.py:121)
- [output_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/output_processor.py:333)

P1 固定目标是：

- 不改最终 `RequestOutput.prompt_logprobs` 的外部语义
- 不改 `DELTA` 模式下 `pop_prompt_logprobs()` 的行为
- 只保证 worker 交上来的 tensor 已经是“sidecar prefix + replay suffix”的完整拼装结果

如果这里需要大改，说明前面的原型边界定错了。

## 11. 固定执行流程

P1 的固定执行流程如下：

1. request 进入系统，且 `prompt_logprobs is not None`
2. 若 feature gate 关闭，走现有 full-prefill baseline
3. 若 gate 打开，先做 prefix lookup，拿到 `physical_hit_tokens`
4. 查询 sidecar：
   - 若没有 sidecar，fallback
   - 若 sidecar 覆盖不足，fallback
5. 计算 `replay_start_token` 与 `replay_tokens`
6. 若 `replay_tokens` 超预算，fallback
7. scheduler 丢掉命中边界最后一个 block，仅保留 `effective_reuse_tokens`
8. worker 先把 sidecar slice 拷入 `LogprobsTensors`
9. 常规 prefill 从 `replay_start_token` 跑到末尾，补齐剩余 prompt observations
10. `LogprobsProcessor` / `OutputProcessor` 按原逻辑返回结果

这个流程里唯一允许的 fallback 是：

- 留在当前 pool / 当前执行路径做 full prefill

不允许：

- 中途切到另一种 API 语义
- 降级成只返回部分 `prompt_logprobs`

## 12. fallback 与 failure mode

P1 固定只允许下面几类 fallback reason：

- `sidecar_missing`
- `sidecar_coverage_insufficient`
- `replay_budget_exceeded`
- `hit_too_short_for_boundary_replay`
- `non_token_carrier`
- `request_preempted_before_sidecar_materialization`

这些 reason 必须和 measurement / report 对齐，不能在代码里再发明另一套名字。

## 13. 正确性要求

原型是否成立，首先看 correctness，不先看性能。

### 13.1 输出完整性

对任意 bridge 成功的 request，必须满足：

- 返回的 `prompt_logprobs` 条目数与 full-prefill baseline 一致
- `prompt_logprobs` 的位置对齐关系一致
- `DELTA` 输出时机一致

### 13.2 数值一致性

在固定模型、固定 prompt、固定采样配置下：

- selective replay 返回的每个 prompt position 的 top-k token id、rank、logprob
- 必须与 full-prefill baseline 在允许数值误差范围内一致

### 13.3 运行时不变量

P1 原型从现在开始固定满足下面四条不变量：

1. `physical_hit_tokens` 永远表示 lookup 阶段真实观测到的物理命中上界，不能被 replay 裁剪结果覆盖。
2. `effective_reuse_tokens <= physical_hit_tokens` 必须始终成立；若不成立，说明 scheduler 裁剪语义出错。
3. `prompt_logprobs_sidecar_covered_tokens <= prompt_logprobs_replay_start_token` 必须成立；sidecar 不允许越过 replay 起点。
4. 一旦 `prompt_logprobs_bridge_active = true`，最终输出的 prompt-logprobs 长度必须与 full-prefill baseline 一致；不允许返回“部分 observation”。

这些不变量的意义在于：它们把“桥接成功”从一个模糊说法收束成可检查的运行时事实。

## 14. 触发条件矩阵

为了让实现方案能够与论文中的“发生条件”一一对应，`prompt_logprobs selective replay` 的触发必须显式满足下面这张表，而不是隐含在 scattered if-else 中。

| 条件 | 判定来源 | 不满足时的固定行为 |
| --- | --- | --- |
| 请求为 token carrier | `Request` typed tags | `full-prefill fallback`，原因记为 `non_token_carrier` |
| 请求要求 `prompt_logprobs` | `SamplingParams` / `Request` | 不进入该 bridge |
| feature gate 打开 | runtime config | 保持现有 unconditional skip 行为 |
| 存在非零 `physical_hit_tokens` | `KVCacheManager` lookup 结果 | `full-prefill fallback`，原因记为 `hit_too_short_for_boundary_replay` 或普通 miss |
| sidecar 可用且覆盖充分 | sidecar registry | `full-prefill fallback`，原因记为 `sidecar_missing` 或 `sidecar_coverage_insufficient` |
| `replay_tokens` 不超过预算 | bridge planner | `full-prefill fallback`，原因记为 `replay_budget_exceeded` |

这里要特别强调：`prompt_logprobs selective replay` 不是“命中了就桥接”，而是“命中 + sidecar + replay 预算”三者同时成立时才桥接。

## 15. 最小代码改动批次

从可执行性的角度，P1 不应一次性展开为大补丁，而应拆成四个顺序固定的最小批次。

### 15.1 Batch A：无行为变化的字段与 trace 打底

目标：

- 加 feature gate
- 加 request-level bridge plan 字段
- 不改变任何默认执行行为

验收：

- 关闭 gate 时所有现有测试与基准结果不变
- trace 中可以看到 `prompt_logprobs` candidate 的诊断字段

### 15.2 Batch B：sidecar 建立与读取

目标：

- 在 full-prefill baseline 路径上生成 sidecar
- 在后续请求中能够按 prefix-family key 读回 sidecar

验收：

- sidecar 生命周期跟随 prefix cache reset 一起清理
- sidecar 命中不会改变现有输出

### 15.3 Batch C：scheduler 裁剪 + worker 拼装

目标：

- 根据 `replay_start_token` 裁剪 `effective_reuse_tokens`
- worker 先写 sidecar slice，再用 replay 段补齐 observation

验收：

- correctness tests 通过
- `physical_hit_tokens` 与 `effective_reuse_tokens` 语义未混淆

### 15.4 Batch D：性能验证与回退保护

目标：

- 跑预实验 workload
- 打开 replay budget 和 fallback logs
- 验证 TTFT 与吞吐变化

验收：

- `W1` 上出现显著 TTFT 改善
- 吞吐回退在预算内
- fallback rate 可解释

## 16. 原型测试与验收矩阵

`prompt_logprobs selective replay` 从现在开始固定采用四层测试，而不是只跑 end-to-end benchmark。

### 16.1 单元测试

需要至少覆盖：

- `replay_start_token` 的 block 对齐计算
- `effective_reuse_tokens` 与 `physical_hit_tokens` 的关系
- 各类 fallback reason 的分支选择

### 16.2 语义回归测试

需要至少覆盖：

- 相同 prompt 下 full prefill 与 selective replay 的 `prompt_logprobs` 长度一致
- top-k token id、rank、logprob 数值一致
- `DELTA` 输出模式一致

### 16.3 运行时 trace 测试

需要至少覆盖：

- bridge 成功时的 request trace
- sidecar 缺失时的 fallback trace
- replay 预算不足时的 fallback trace

### 16.4 端到端性能测试

固定优先跑：

- `w1_prompt_logprobs`
- 长共享 system prompt 的多轮 chat
- 至少一个非命中对照 workload

固定核心指标：

- `TTFT`
- `p95 TTFT`
- throughput
- bridge success rate
- fallback reason breakdown

## 17. kill criteria 与降级路径

为了防止实现方案继续膨胀，P1 从现在开始固定采用下面的 kill criteria。

1. 若无法在不引入 hidden-state cache 的前提下保证 `prompt_logprobs` correctness，则停止这一路线，明确降级为“需要更重状态”的后续方向。
2. 若 `W1` 上 bridge 成功率很高但 `TTFT` 改善仍不明显，则说明 selective replay 的收益不足以支撑 thesis，只能作为 patch 线保留。
3. 若吞吐回退持续高于目标预算，则必须优先收缩 replay 范围或收紧触发条件，而不是继续加复杂优化。

换句话说，这条 bridge 是否成立，不取决于“是否能 work”，而取决于它能否在保持轻量边界的同时给出可写进论文的收益。

### 13.3 生命周期一致性

sidecar 的生命周期固定跟 prefix-cache reset 绑定：

- reset prefix cache 时同时清理 sidecar
- 不允许 sidecar 比 prefix-family 生命周期更长

## 14. 预实验与评测对齐

这条 bridge 的实验必须和 [../experiments/paper_evaluation_matrix.md](../experiments/paper_evaluation_matrix.md) 对齐。

### 14.1 correctness 实验

至少做三组：

1. 无 cache baseline：当前 `prompt_logprobs` full prefill
2. selective replay：sidecar + boundary replay
3. fallback case：命中但 sidecar 缺失，退回 full prefill

固定检查：

- prompt_logprobs 长度
- 每个位置 top-k 一致性
- 首个跨边界 token 的正确性

### 14.2 性能实验

主战场固定是：

- `W1` 长 system prompt 多轮 chat
- `W2` tool-rich / agentic

固定指标：

- `TTFT`
- `p95/p99 TTFT`
- replay token 占比
- `effective_reuse_tokens / physical_hit_tokens`
- fallback rate

### 14.3 关键 ablation

至少做下面三组 ablation：

1. `full prefill`
2. `sidecar only` 假设性分析
3. `sidecar + boundary replay`

这组 ablation 的作用是证明：

- 真正必要的是 `boundary replay`
- 真正带来收益的是“只丢最后一个 block”，而不是整段重算

## 15. 对摘要和引言的直接影响

这份设计文档落地后，paper 里关于第一个 bridge 的写法应固定收敛为：

- 它不是“支持 `prompt_logprobs + APC`”
- 它是“用最小 prompt-observation sidecar 和 boundary replay 恢复 false-negative exact hit”

更适合放进摘要/引言的方法句可以写成：

> We instantiate the first bridge with `prompt_logprobs` selective replay, which augments exact-prefix KV reuse with a minimal prompt-observation sidecar and a one-block boundary replay to recover prompt-level observations without full-prompt recomputation.

中文工作口径则固定为：

> 我们不是让 `prompt_logprobs` 无条件读 cache，而是给 exact-prefix reuse 补上最小 observation sidecar，并仅回放命中边界的最后一个 block 与未命中 suffix，以恢复完整 prompt observation。

## 16. 最小实现清单

如果下一步开始写 prototype，最小 diff 应按下面顺序展开：

1. 在 `SamplingParams` 加 feature gate，去掉 `prompt_logprobs => unconditional skip` 的硬绑定
2. 在 `Request` 上增加 bridge-plan 字段
3. 新增 sidecar registry 小模块
4. 在 scheduler 中实现“命中上界 -> 实际复用边界”的裁剪
5. 在 worker 中实现“sidecar prefix copy + replay suffix fill”
6. 保持 `LogprobsProcessor` / `OutputProcessor` 外部协议不变

如果第 4 步做不成，就不要硬推在线 prototype，而应先停下来改设计。

## 17. 成功标准

这条设计在工程和论文上都成立，至少要同时满足下面三条：

1. 它把 `prompt_logprobs_skip` 从 measurement 问题变成可恢复机制
2. 它的最小实现不需要重写通用 prefill / output 协议
3. 它在 paper 中能被清楚地表述为 `observation mismatch` 的 bridge，而不是单点 feature patch

如果最后只能做到“有 cache 命中时少量特殊判断”，但说不清：

- 为什么需要 sidecar
- 为什么要 replay 最后一个 block
- 为什么这属于统一 `reuse contract`

那这条 bridge 就还不够稳，不应直接写进主论文贡献。
