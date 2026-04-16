# ExactPrefixKV P0 Instrumentation Spec

更新日期：2026-04-15

## 1. 目标

这份规格文档只服务于 **P0 instrumentation**。

它回答的问题不是：

- 完整系统最终长什么样；
- P1/P2/P3 如何实现；

而是：

> 第一批最小代码改动，究竟要加哪些字段、写什么日志、落在哪些文件里，
> 才能用最小 diff 验证 `ExactPrefixKV` 是否值得继续做。

因此，这份 spec 的设计原则是：

1. **只做观测，不改策略**
2. **优先复用现有 logger / stats 出口**
3. **尽量不改外部 API**
4. **先支持 T1 + T2，再决定是否进入更细粒度的 T3/T4**

## 2. P0 的范围

### 2.1 P0 第一批只覆盖两类信息

第一批只做下面两块：

1. **输入侧 typed tags**
   - 请求进入 engine 时，记录它属于哪类 carrier、是否带 `cache_salt`、是否带多模态等。
2. **prefix lookup outcome**
   - 请求做 prefix-cache lookup 时，记录：
   - 查询了多少 token
   - 命中了多少 token
   - 是否跳过读取
   - 如果没命中，属于哪类粗粒度 miss

### 2.2 P0 第一批明确不做

下面这些都不在第一批里：

1. post-hit graph/cascade/padding 联合 trace
2. block-level typed extra-keys 全量 trace
3. semantic breaker taxonomy 的完整归因
4. 新的 Prometheus label 体系
5. 新的用户可见请求参数
6. 新的调度或 cache 策略

原因很简单：

- 如果第一批连 drift / typed miss / read skip 都量不出来，
- 那继续做更细的 instrumentation 没有意义。

## 3. 设计原则

### 3.1 不改外部协议语义

第一批 instrumentation 不应改变：

- OpenAI 兼容接口
- 内部调度语义
- prefix cache 命中逻辑
- graph / cascade 选择逻辑

也就是说：

- **同样的请求，开关前后输出必须一致。**

### 3.2 允许新增内部可选字段

允许新增：

- 内部 request 上的只读 instrumentation tags
- 内部 stats dataclass
- 可选 observability config
- JSONL trace writer

但这些新增字段必须：

- 默认关闭
- 对热路径开销可控
- 不要求用户修改 workload

### 3.3 日志优先 JSONL，而不是先上 Prometheus

第一批最核心的输出应是 **JSONL**，原因如下：

1. request-level trace 天然适合 JSONL
2. breaker / miss_class 这类离散标签，第一轮更适合离线分析
3. 现有 Prometheus metrics 已有 prefix cache 总量统计，不缺第一批粗粒度 counters

所以第一批的 primary output 应该是：

- `input trace`
- `lookup trace`

Prometheus / 终端日志只作为 secondary output。

## 4. 第一批新增配置

### 4.1 配置目标

第一批建议只新增两个 observability 配置项。

### 4.2 新增字段建议

建议在 [observability.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/config/observability.py:18) 新增：

1. `enable_exact_prefix_instrumentation: bool = False`
   - 总开关
2. `exact_prefix_trace_dir: str | None = None`
   - JSONL 输出目录

### 4.3 为什么不用更多配置项

第一批不建议再加：

- breaker 白名单
- 采样率
- 多文件模板
- label 粒度开关

这些会让 spec 过早复杂化。

第一批只需要：

- 开或关
- 往哪里写

就够了。

## 5. 第一批新增内部对象

### 5.1 Request 级 tags

建议在 [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:59) 的 `Request` 内新增一组只读 instrumentation tags。

建议字段如下：

1. `instrument_carrier_type: str`
   - 枚举值：
   - `tokens`
   - `embeds`
   - `multimodal`
2. `instrument_has_cache_salt: bool`
3. `instrument_num_mm_items: int`
4. `instrument_prompt_tokens: int`
5. `instrument_lookup_stage: str`
   - 第一批可固定为：
   - `prefill`
   - `resumed`
   - `stream_update`

### 5.2 为什么挂在 Request，而不是 EngineCoreRequest

这是第一批最关键的最小 diff 决策。

不建议第一批直接改 [EngineCoreRequest](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/__init__.py:80) 的协议字段，原因是：

1. 这些 tags 大多可以从现有字段推导：
   - `prompt_embeds`
   - `mm_features`
   - `cache_salt`
   - `num_prompt_tokens`
2. 不改 IPC struct，风险更低；
3. `Request.__init__` 本来就是最自然的推导点。

因此第一批建议：

- **不改 `EngineCoreRequest` 结构；**
- **在 `Request` 构造时派生 instrumentation tags。**

### 5.3 新增 lookup stats 对象

建议在 [metrics/stats.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/stats.py:161) 附近新增一个 dataclass：

- `ExactPrefixLookupStats`

第一批字段建议如下：

1. `requests`
2. `queries`
3. `hits`
4. `token_requests`
5. `token_queries`
6. `token_hits`
7. `embed_requests`
8. `embed_queries`
9. `embed_hits`
10. `multimodal_requests`
11. `multimodal_queries`
12. `multimodal_hits`
13. `salted_requests`
14. `read_skipped_requests`
15. `read_skipped_prompt_logprobs`
16. `read_skipped_pooling`
17. `full_hit_requests`
18. `partial_hit_requests`
19. `zero_hit_requests`

### 5.4 为什么第一批不用 dict[str, int]

虽然 `dict` 更灵活，但第一批不建议用动态 map，原因是：

1. 固定字段更稳定，便于 logger 和 CSV 汇总；
2. 减少 Prometheus / logging 侧处理复杂度；
3. 第一批 carrier 类型是固定的三类，不需要泛化。

## 6. 第一批日志文件设计

### 6.1 输出目录

`exact_prefix_trace_dir` 应被视为目录，而不是单文件路径。

原因：

- frontend 和 engine core 不应并发写同一文件；
- DP / 多 engine core 时也不应竞争同一描述符。

### 6.2 目录结构建议

若配置：

- `exact_prefix_trace_dir=/tmp/exact_prefix_trace`

则建议输出：

1. `/tmp/exact_prefix_trace/frontend_input.jsonl`
2. `/tmp/exact_prefix_trace/engine_0_lookup.jsonl`
3. `/tmp/exact_prefix_trace/engine_1_lookup.jsonl`
   - 如果有多个 engine core

### 6.3 JSONL 基本要求

每条记录必须：

1. 单行 JSON
2. 含 `schema_version`
3. 含 `event_type`
4. 含 `request_id`
5. 含 `ts_ms`

这样后续离线分析脚本无需猜格式。

## 7. `frontend_input.jsonl` 记录格式

### 7.1 写入时机

建议在下面两个位置写：

1. [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:321)
   - 通用入口
2. [responses/serving.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/serving.py:715)
   - responses 路径可补 richer context

### 7.2 第一批字段

建议字段如下：

- `schema_version`
- `event_type`
- `ts_ms`
- `request_id`
- `route_kind`
  - `responses`
  - `generic`
- `carrier_type`
  - `tokens`
  - `embeds`
  - `multimodal`
- `prompt_tokens`
- `has_cache_salt`
- `num_mm_items`
- `is_resumable`
- `has_prompt_embeds`
- `data_parallel_rank`

### 7.3 可选字段

如果在 responses 路径能低成本拿到，再补下面这些：

- `tool_count_hint`
- `has_prev_response`
- `filtered_system_messages`
- `skipped_reasoning_items`

这些字段是 **可选增强**，不是第一批 blocker。

### 7.4 示例

```json
{"schema_version":"v1","event_type":"frontend_input","ts_ms":1713168000123,"request_id":"req_42","route_kind":"responses","carrier_type":"multimodal","prompt_tokens":1824,"has_cache_salt":true,"num_mm_items":1,"is_resumable":false,"has_prompt_embeds":false,"data_parallel_rank":null}
```

## 8. `engine_<idx>_lookup.jsonl` 记录格式

### 8.1 写入时机

建议主写点放在：

- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:176)

因为这里天然拿得到：

- request
- queried tokens
- hit tokens
- skip read 逻辑

### 8.2 第一批字段

建议字段如下：

- `schema_version`
- `event_type`
- `ts_ms`
- `engine_idx`
- `request_id`
- `carrier_type`
- `prompt_tokens`
- `queried_tokens`
- `hit_tokens`
- `hit_ratio`
- `has_cache_salt`
- `num_mm_items`
- `preempted`
- `read_skipped`
- `skip_reason`
- `hit_class`
- `lookup_stage`

### 8.3 `skip_reason` 枚举

第一批建议只支持下面几个值：

- `NONE`
- `PREFIX_CACHING_DISABLED`
- `SKIP_READING_PREFIX_CACHE`
- `PROMPT_LOGPROBS`
- `POOLING_ALL`

这里不追求最终 breaker taxonomy，只做最小可解释归因。

### 8.4 `hit_class` 枚举

第一批建议只支持：

- `FULL_HIT`
- `PARTIAL_HIT`
- `ZERO_HIT`
- `READ_SKIPPED`

### 8.5 示例

```json
{"schema_version":"v1","event_type":"cache_lookup","ts_ms":1713168000456,"engine_idx":0,"request_id":"req_42","carrier_type":"multimodal","prompt_tokens":1824,"queried_tokens":1823,"hit_tokens":1536,"hit_ratio":0.8426,"has_cache_salt":true,"num_mm_items":1,"preempted":false,"read_skipped":false,"skip_reason":"NONE","hit_class":"PARTIAL_HIT","lookup_stage":"prefill"}
```

## 9. 第一批聚合统计格式

### 9.1 目的

JSONL 用于离线分析，但第一批仍应把核心统计接到现有 `SchedulerStats -> StatLoggerManager` 出口。

### 9.2 新增聚合对象

建议在 [SchedulerStats](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/stats.py:170) 中新增：

- `exact_prefix_lookup_stats: ExactPrefixLookupStats | None = None`

然后在 [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:1959) 构造 `SchedulerStats` 时附带出去。

### 9.3 第一批终端日志格式

建议只在 `LoggingStatLogger` 中追加一段简洁摘要，不做大改版式。

建议格式：

```text
ExactPrefix: req=128 q=145920 hit=101376 hr=69.47% token_hr=71.2% embed_hr=63.5% mm_hr=58.1% skipped=7 full=29 partial=66 zero=26
```

### 9.4 第一批 Prometheus 策略

第一批不建议新增太多 label。

建议最多新增以下 counters：

1. `vllm:exact_prefix_requests_total`
2. `vllm:exact_prefix_queries_total`
3. `vllm:exact_prefix_hits_total`
4. `vllm:exact_prefix_read_skipped_total`

carrier breakdown 和 hit_class breakdown 先放 JSONL，不急着进 Prometheus。

## 10. 最小 diff 文件清单

第一批建议只动下面这些文件。

### 10.1 配置与工具

1. [observability.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/config/observability.py:18)
   - 新增配置字段。
2. 新增一个小工具模块
   - 建议路径：
   - `vllm/v1/metrics/exact_prefix_trace.py`
   - 负责 JSONL writer 和 record schema。

### 10.2 request 与输入侧

3. [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:59)
   - 派生 request instrumentation tags。
4. [input_processor.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/engine/input_processor.py:321)
   - 写 `frontend_input` 记录。
5. [responses/serving.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/serving.py:715)
   - 可选补 richer request hints。

### 10.3 lookup 与 stats

6. [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:176)
   - 计算并写 `cache_lookup` 记录；
   - 同时更新 `ExactPrefixLookupStats`。
7. [metrics/stats.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/stats.py:170)
   - 定义 `ExactPrefixLookupStats`
   - 扩展 `SchedulerStats`
8. [scheduler.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/sched/scheduler.py:1959)
   - 把聚合 stats 带出。
9. [metrics/loggers.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/metrics/loggers.py:99)
   - 在 `LoggingStatLogger` 中追加摘要；
   - 可选在 `PrometheusStatLogger` 中加四个简单 counters。

## 11. 第一批字段的推导规则

### 11.1 `carrier_type`

推导规则：

1. `prompt_embeds is not None` -> `embeds`
2. `mm_features` 非空 -> `multimodal`
3. 否则 -> `tokens`

### 11.2 `prompt_tokens`

直接使用：

- `request.num_prompt_tokens`

不要重新计算。

### 11.3 `queried_tokens`

沿用现有口径：

- `request.num_tokens`
- 但在 full-hit 情况下，lookup 逻辑本身仍受 “最后一个 token 需要重算” 影响；
- 第一批直接记录：
  - `queried_tokens = request.num_tokens - 1`
  - 与当前 `max_cache_hit_length` 对齐

### 11.4 `hit_tokens`

直接使用 `get_computed_blocks()` 当前返回的：

- `num_new_computed_tokens`

### 11.5 `lookup_stage`

推导规则：

1. `request.num_preemptions > 0` -> `resumed`
2. streaming update 场景 -> `stream_update`
3. 否则 -> `prefill`

第一批允许先只区分：

- `prefill`
- `resumed`

## 12. 第一批开销控制

### 12.1 默认关闭

默认必须关闭。

### 12.2 JSONL 仅在 trace_dir 非空时写

如果：

- `enable_exact_prefix_instrumentation=False`
  - 什么都不做
- `enable_exact_prefix_instrumentation=True`
  - 只做聚合统计
- `exact_prefix_trace_dir` 非空
  - 再额外写 JSONL

### 12.3 不在热路径做复杂字符串拼接

record 对象应先组 dict，再走一次 `json.dumps` 或等价序列化。

不要在热路径反复构造大文本。

## 13. 与 measurement plan 的对应关系

这份 spec 主要覆盖 [measurement_plan.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/12_combined_main_thesis/measurement_plan.md) 中的：

- `T1` 输入侧 typed trace
- `T2` prefix-hit 与 breaker trace

它**不**覆盖：

- `T3` block-level typed composition trace
- `T4` post-hit execution trace

换句话说，第一批的目的只是回答：

1. drift / typed miss 是否足够明显；
2. 现有 prefix-cache 查询结果是否足够可解释；
3. 是否值得继续做更深 instrumentation。

## 14. 实现顺序建议

建议严格按下面顺序做。

1. 先加配置与 JSONL writer
2. 再在 `Request` 构造处加派生 tags
3. 再在 `kv_cache_manager.get_computed_blocks()` 接聚合 stats
4. 再接 `SchedulerStats -> LoggingStatLogger`
5. 最后再补 `frontend_input.jsonl`

这个顺序的理由是：

- 先把 core-side lookup 统计跑通，最能验证价值；
- frontend richer context 是加分项，不是第一 blocker。

## 15. 完成条件

第一批 P0 instrumentation 只有满足下面四条，才算真正完成。

1. 可以稳定输出 `lookup` JSONL
2. 可以稳定输出 `frontend_input` JSONL
3. 终端日志能看到 exact-prefix 聚合摘要
4. 至少能离线算出下面四个数字：
   - token / embed / multimodal 三类 carrier 的 hit ratio
   - read skipped 请求占比
   - full / partial / zero hit 分布
   - salted 请求占比

## 16. 当前建议

这份 spec 刻意把第一批范围压得很小。

原因不是保守，而是：

- 只要第一批数据已经显示：
  - typed carrier 差异明显；
  - read skipped / partial hit 足够多；
  - workload family 间差异清晰；

那 `ExactPrefixKV` 就有很强的继续价值。

如果连这些最基础的观测都不成立，就不值得继续做更复杂的 post-hit instrumentation。
