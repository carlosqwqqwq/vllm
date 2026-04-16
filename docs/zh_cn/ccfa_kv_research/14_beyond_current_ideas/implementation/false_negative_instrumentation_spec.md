# FalseNegativeKV Instrumentation Spec

更新日期：2026-04-15

## 1. 目标

这份规格文档只服务于 `FalseNegativeKV` 的 **P0 measurement**。

它不负责：

- 实现 `TwinPathServe`
- 实现 `prompt_logprobs` selective replay
- 实现任何新的 cache / scheduler 策略

它只回答一个问题：

> 第一批最小观测改动，究竟要在 `vLLM` 里记录哪些字段、写哪些 JSONL 事件、挂在哪些代码入口，
> 才能把 `false-negative miss` 从概念变成可量化对象。

因此，这份 spec 的固定原则是：

1. **只做观测，不改策略**
2. **默认关闭**
3. **优先 request-level JSONL**
4. **每个核心指标都必须映射到明确代码入口**

## 2. 当前阶段固定接口

这一阶段固定采用下面两个内部配置名，不再重命名：

- `enable_false_negative_instrumentation`
- `false_negative_trace_dir`

这两个名字从现在开始视为锁定接口，后续实现和文档都统一用它们。

## 3. 核心术语与指标

从现在开始，`FalseNegativeKV` 的 measurement 文档统一只使用下面五个核心指标名：

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `bridgeable_tokens`
- `false_negative_reason`

并固定采用以下定义：

### 3.1 `logical_hit_tokens`

指请求在逻辑上本可复用的 prefix tokens 数量。

第一批 measurement 中，它不是直接由现有 `vLLM` 代码产出，而是通过**观测层推导**得到：

- 如果请求没有因为 contract mismatch 被跳过读取 prefix cache，
  `logical_hit_tokens == physical_hit_tokens`
- 如果请求因为已知 contract mismatch 被整段绕过，
  则需要通过同 prefix family 对照、或在后处理脚本中用 paired request 近似推导

因此第一批实现里：

- **允许 `logical_hit_tokens` 作为 derived metric 出现在离线分析阶段**
- 但必须在日志中留下足够字段支持后处理推导

### 3.2 `physical_hit_tokens`

指 `vLLM` 在 prefix cache lookup 中实际跳过计算的 tokens 数量。

这一定义直接绑定现有 prefix lookup 路径，不能改。

### 3.3 `false_negative_tokens`

固定定义为：

```text
false_negative_tokens = max(logical_hit_tokens - physical_hit_tokens, 0)
```

其含义是：

- 逻辑上应命中
- 物理上没命中

### 3.4 `bridgeable_tokens`

指属于 `false-negative_tokens` 中，理论上可被当前研究路线恢复的那一部分 tokens。

第一批 measurement 不要求 runtime 在线给出精确值，但必须预留：

- `bridgeable_class`
- `bridge_candidate`

使后处理脚本能够按 reason class 聚合出 `bridgeable_tokens`。

### 3.5 `false_negative_reason`

这是第一批最关键的分类字段。

第一批固定 reason taxonomy 为：

- `prompt_logprobs_skip`
- `pooling_skip`
- `backend_incompatibility`
- `partial_prefill_incompatibility`
- `carrier_incompatibility`
- `no_false_negative`
- `unknown`

解释如下：

- `prompt_logprobs_skip`
  - 请求因为 `prompt_logprobs` 语义而绕过 prefix cache 读取
- `pooling_skip`
  - 请求因为 pooling task 默认策略而绕过 prefix cache 读取
- `backend_incompatibility`
  - prefix cache 因 backend / batch invariance 组合而被禁用
- `partial_prefill_incompatibility`
  - 请求处在 partial prefill path，但 consumer 不支持
- `carrier_incompatibility`
  - `prompt_embeds` / multimodal / token carrier 等输入载体限制使 exact hit 不能被消费
- `no_false_negative`
  - 本请求未落入 false-negative 分类
- `unknown`
  - 现阶段无法更细分

第一批实现不允许新增新的一级 taxonomy；如果未来要扩展，只能在二级字段里做。

## 4. P0 范围

### 4.1 第一批必须覆盖的观测面

第一批只覆盖下面三块信息：

1. 请求输入侧 typed tags
2. prefix cache lookup outcome
3. lookup 被跳过或被整体关闭的原因

### 4.2 第一批明确不做

下面这些都不在 P0 第一批里：

- Prometheus 主通路指标
- block-level hash / extra key 全量 trace
- graph replay / padding / cascade 完整 profile
- scheduler policy 改动
- cache policy 改动
- replay / bridge 真实执行

如果第一批无法把 false-negative miss 量出来，继续做更复杂观测没有意义。

## 5. 代码入口与字段映射

本节是这份文档最重要的部分：**每个核心指标都必须落到明确代码入口。**

## 5.1 输入侧 typed tags

目标是为每个请求打上“它属于哪类 carrier、哪类 API contract”的标签。

### 入口 A：`Request` 构造与派生字段

代码位置：

- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:59)

在这里固定派生下面这些只读 instrumentation tags：

- `instrument_carrier_type`
  - 固定枚举：
    - `tokens`
    - `embeds`
    - `multimodal`
- `instrument_has_cache_salt`
- `instrument_num_mm_items`
- `instrument_prompt_tokens`
- `instrument_api_contract_class`
  - 第一批固定枚举：
    - `generation`
    - `prompt_logprobs`
    - `pooling`
- `instrument_false_negative_hint`
  - 第一批固定枚举：
    - `none`
    - `prompt_logprobs_skip`
    - `pooling_skip`

这里的决定是：

- **第一批不改 `EngineCoreRequest` 线上的公共结构**
- 只在 `Request` 内派生只读 instrumentation tags

原因：

- 风险最小
- 不改变外部协议
- 现有字段足以推导

### 入口 B：输入处理阶段

代码位置：

- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:257)
- [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:324)

这里负责补齐 `frontend_input` 事件必须写入的 request-level 元数据：

- `request_id`
- `carrier_type`
- `prompt_tokens`
- `has_cache_salt`
- `num_mm_items`
- `has_prompt_embeds`
- `api_contract_class`

### 入口 C：OpenAI responses 路径

代码位置：

- [responses/serving.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/serving.py:715)

这里只补 richer context，不改主 schema。

允许补充的可选字段固定为：

- `route_kind`
  - `responses`
  - `generic`
- `tool_count_hint`
- `has_prev_response`
- `skipped_reasoning_items`

如果某字段不能低成本获取，就不记；第一批不为此扩路径。

## 5.2 物理命中与跳过读取

### 入口 D：prefix cache lookup 主入口

代码位置：

- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:188)

这里固定记录 `lookup` 事件的核心字段：

- `request_id`
- `query_tokens`
  - 固定等于请求用于 prefix lookup 的 token 数
- `physical_hit_tokens`
  - 固定等于 lookup 返回的命中 token 数
- `skip_reading_prefix_cache`
- `false_negative_reason`
- `bridge_candidate`
  - `true/false`

具体映射规则固定如下：

- 如果 `enable_caching == false`
  - `false_negative_reason = unknown`
  - `skip_reading_prefix_cache = false`
  - 本事件仍记录，但 `measurement_scope = disabled_cache`
- 如果 `request.skip_reading_prefix_cache == true`
  - 根据 request tags 进一步细分：
    - `prompt_logprobs` -> `prompt_logprobs_skip`
    - `pooling` -> `pooling_skip`
    - 其它 -> `unknown`
- 如果正常 lookup 执行
  - `false_negative_reason = no_false_negative`
  - `physical_hit_tokens = num_new_computed_tokens`

### 入口 E：请求级 skip 原因推导

代码位置：

- [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431)
- [pooling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/pooling_params.py:130)
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:242)

这里不直接写日志，但固定负责 reason 推导逻辑：

- `prompt_logprobs is not None`
  - `instrument_api_contract_class = prompt_logprobs`
  - `instrument_false_negative_hint = prompt_logprobs_skip`
- `pooling_params.skip_reading_prefix_cache == true`
  - `instrument_api_contract_class = pooling`
  - `instrument_false_negative_hint = pooling_skip`

## 5.3 backend incompatibility

### 入口 F：attention backend 初始化阶段

代码位置：

- [attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/attention.py:319)
- [mla_attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/mla_attention.py:375)

这里不做 request-level trace，而是固定写 **engine-level config snapshot**。

必须记录的字段：

- `engine_id`
- `attn_backend`
- `batch_invariant_enabled`
- `prefix_caching_disabled_by_backend`
- `backend_incompatibility_reason`

第一批 reason 固定为：

- `flashinfer_batch_invariant_apc_unsupported`
- `triton_mla_batch_invariant_apc_unsupported`

输出文件固定为：

- `engine_<idx>_config.jsonl`

原因：

- 这是实例级配置，不是请求级事件
- 后处理时用 `engine_id` join 到 request traces

## 5.4 partial prefill incompatibility

### 入口 G：pooling path 的 partial prefill 约束

代码位置：

- [methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/seqwise/methods.py:43)
- [methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/seqwise/methods.py:67)
- [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:678)

第一批不在 assert 处直接打日志，避免污染异常语义。

固定做法是：

- 记录 request 的 `pooling_task_class`
- 记录 engine 的 `chunked_prefill_enabled`
- 记录 request 是否实际进入 chunked schedule path

后处理按下面规则派生：

- 如果：
  - request 是 `seqwise` pooling
  - engine 开启 `chunked_prefill`
  - 请求进入 partial prefill 相关路径
- 则标为：
  - `false_negative_reason = partial_prefill_incompatibility`

## 5.5 carrier incompatibility

### 入口 H：`prompt_embeds` 与 carrier 约束

代码位置：

- [gpu_input_batch.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_input_batch.py:68)
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:471)

第一批只做观测，不做判责扩张。

必须记录：

- `has_prompt_embeds`
- `prompt_token_ids_known`
- `carrier_type`

后处理规则固定为：

- 如果请求携带 `prompt_embeds`
- 且下游 task 需要 token-id-level observation 或 token carrier
- 则归入：
  - `carrier_incompatibility`

第一批不尝试在线判断更细粒度的 multimodal carrier mismatch。

## 6. JSONL 输出设计

### 6.1 目录结构

`false_negative_trace_dir` 固定视为目录，不允许配置成单文件。

固定输出结构为：

- `frontend_input.jsonl`
- `engine_<idx>_lookup.jsonl`
- `engine_<idx>_config.jsonl`

如果未来新增文件，只能在这个目录下扩展，不能改变已有文件名。

### 6.2 通用字段

所有 JSONL 事件都必须包含：

- `schema_version`
  - 固定为 `fnkv_v1`
- `event_type`
- `ts_ms`
- `request_id`
  - 引擎级 config event 允许写 `null`

## 6.3 `frontend_input.jsonl`

### 写入时机

- 请求完成内部 input processing 之后、进入 engine request 构造之前或之后均可
- 但必须保证每个 request 只写一次

### 固定字段

- `schema_version`
- `event_type = frontend_input`
- `ts_ms`
- `request_id`
- `route_kind`
- `carrier_type`
- `prompt_tokens`
- `has_cache_salt`
- `num_mm_items`
- `has_prompt_embeds`
- `api_contract_class`
- `tool_count_hint`
- `has_prev_response`

### 示例

```json
{"schema_version":"fnkv_v1","event_type":"frontend_input","ts_ms":1713168000123,"request_id":"req_42","route_kind":"responses","carrier_type":"multimodal","prompt_tokens":1824,"has_cache_salt":true,"num_mm_items":1,"has_prompt_embeds":false,"api_contract_class":"generation","tool_count_hint":3,"has_prev_response":true}
```

## 6.4 `engine_<idx>_lookup.jsonl`

### 写入时机

- 每次进入 prefix cache lookup 时写一次

### 固定字段

- `schema_version`
- `event_type = prefix_lookup`
- `ts_ms`
- `request_id`
- `engine_id`
- `carrier_type`
- `api_contract_class`
- `query_tokens`
- `physical_hit_tokens`
- `skip_reading_prefix_cache`
- `false_negative_reason`
- `bridge_candidate`
- `measurement_scope`
  - 固定枚举：
    - `normal_lookup`
    - `skip_read`
    - `cache_disabled`

### `bridge_candidate` 固定规则

第一批在线判定规则固定为：

- `prompt_logprobs_skip` -> `true`
- `backend_incompatibility` -> `true`
- `pooling_skip` -> `true`
- `partial_prefill_incompatibility` -> `false`
- `carrier_incompatibility` -> `false`
- 其它 -> `false`

原因：

- 第一批只为 P1 两类强实例化服务
- 不提前承诺所有 reason 都可桥接

### 示例

```json
{"schema_version":"fnkv_v1","event_type":"prefix_lookup","ts_ms":1713168000456,"request_id":"req_42","engine_id":0,"carrier_type":"tokens","api_contract_class":"prompt_logprobs","query_tokens":2048,"physical_hit_tokens":0,"skip_reading_prefix_cache":true,"false_negative_reason":"prompt_logprobs_skip","bridge_candidate":true,"measurement_scope":"skip_read"}
```

## 6.5 `engine_<idx>_config.jsonl`

### 写入时机

- engine 初始化完成后写一次

### 固定字段

- `schema_version`
- `event_type = engine_config`
- `ts_ms`
- `request_id = null`
- `engine_id`
- `attn_backend`
- `batch_invariant_enabled`
- `prefix_caching_enabled`
- `prefix_caching_disabled_by_backend`
- `backend_incompatibility_reason`

### 示例

```json
{"schema_version":"fnkv_v1","event_type":"engine_config","ts_ms":1713167000000,"request_id":null,"engine_id":0,"attn_backend":"FLASHINFER","batch_invariant_enabled":true,"prefix_caching_enabled":false,"prefix_caching_disabled_by_backend":true,"backend_incompatibility_reason":"flashinfer_batch_invariant_apc_unsupported"}
```

## 7. 离线派生规则

第一批 measurement 允许部分核心指标在离线阶段派生，但派生规则必须固定。

### 7.1 `logical_hit_tokens`

第一批不要求 runtime 在线准确写出 `logical_hit_tokens`。

固定后处理规则：

- 对 `measurement_scope = normal_lookup`
  - `logical_hit_tokens = physical_hit_tokens`
- 对 `measurement_scope = skip_read`
  - 如果 `bridge_candidate = true`
    - `logical_hit_tokens` 通过 paired trace 或 same-family baseline 近似估计
  - 如果 `bridge_candidate = false`
    - 标记为 `logical_hit_tokens = unknown`

### 7.2 `false_negative_tokens`

后处理脚本必须按：

```text
if logical_hit_tokens is known:
    false_negative_tokens = max(logical_hit_tokens - physical_hit_tokens, 0)
else:
    false_negative_tokens = unknown
```

### 7.3 `bridgeable_tokens`

第一批固定后处理规则：

- 仅统计 `bridge_candidate = true` 的 requests
- 按 reason class 汇总

## 8. 最小实现边界

为了保证 P0 是最小 diff，这里固定不做下面这些事：

- 不新增新的用户 API 参数
- 不修改 cache lookup 语义
- 不修改 scheduler token budget 语义
- 不修改 prefix cache hash 逻辑
- 不修改 `prompt_logprobs` 行为

如果某个观测需要上述改动之一，直接视为超出 P0。

## 9. 验收标准

`false_negative_instrumentation_spec.md` 的验收标准固定如下：

1. 五个核心指标都有固定定义
2. 每个核心指标都能追溯到明确代码入口
3. 三类 JSONL 文件的 schema 已锁死
4. `false_negative_reason` taxonomy 已锁死
5. 允许离线派生的字段与派生规则已明确

只要这五条里有一条做不到，就不进入实现。

## 10. 后续衔接

这份 spec 完成后，后续文档固定按下面顺序衔接：

1. [../experiments/paper_evaluation_matrix.md](../experiments/paper_evaluation_matrix.md)
2. [twin_path_serve_minimal_architecture.md](twin_path_serve_minimal_architecture.md)
3. [../paper/supporting/paper_storyline_fnkv.md](../paper/supporting/paper_storyline_fnkv.md)

也就是说：

- **先测出 false-negative miss 是否够大**
- **再决定 `TwinPathServe` 是否值得做**
- **最后再把它包装成论文**
