# FalseNegativeKV 公开 Benchmark 实例化计划

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责把
[public_benchmark_evaluation_protocol.md](public_benchmark_evaluation_protocol.md)
中的 benchmark shortlist 进一步落成可执行计划。

它回答四个问题：

1. 先跑哪些 benchmark
2. 每个 benchmark 先用哪一批公开样本
3. 如何从公开样本构造 `prefix family`
4. 当前仓库已有哪部分支持、还缺哪些脚本

它不负责：

1. 改写 `H1-H4` 的论文门槛
2. 直接替代实现文档
3. 承诺所有 benchmark 都在 P1 阶段完成恢复闭环

## 2. 固定执行顺序

从现在开始，公开 benchmark 的推进顺序固定为：

1. `LongBench / LongBench v2`
2. `MT-Bench`
3. `BFCL`
4. `MTEB`
5. `MMBench`

原因不是“这些 benchmark 最全”，而是：

- `LongBench / LongBench v2` 最适合先验证长共享前缀与 backend mismatch
- `MT-Bench` 与 `MTEB` 在当前仓库中已有局部支持，可快速起步
- `BFCL` 是 tool-rich / agentic 主战场，必须补，但脚本现状最薄
- `MMBench` 能证明公开 multimodal 场景下问题仍存在，但实现依赖最重，优先级排在最后

## 3. benchmark 级实例化总表

| benchmark | 对应 workload family | 第一批公开样本 / task | 论文角色 | 当前仓库支持状态 | 下一步脚本需求 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| `LongBench / LongBench v2` | `W1`, `W5` | 公开 released split 中可按相同 context 分组的长上下文任务 | 主 prevalence + 主 recovery | 无现成 runner | 新增 family builder + serve runner + analyzer | `P0` |
| `MT-Bench` | `W1` 辅验证 | `philschmid/mt-bench` 公共数据集 | 多轮 chat 辅验证 | 已有 dataset loader | 补 family grouping 与 prompt_logprobs 对照 runner | `P0` |
| `BFCL` | `W2` | 公共 release 中固定 tool schema 的 multi-turn / function-calling 样本 | 主 prevalence + 主 recovery | 无现成 runner | 新增 tool-schema hash、family builder、serve runner | `P1` |
| `MTEB` | `W3` | `STS12`, `NFCorpus` 起步 | prevalence 主结果 | 已有 embed / rerank 测试工具 | 补 token vs `prompt_embeds` carrier 对照 runner | `P1` |
| `MMBench` | `W4` | 公开 image-question 样本，按 image id 分组 | prevalence 补强 | 无现成 FalseNegativeKV runner | 新增 multimodal family builder 与 trace runner | `P2` |

## 4. `LongBench / LongBench v2` 实例化

## 4.1 角色定位

`LongBench / LongBench v2` 是当前最重要的公开 benchmark。

它固定承担两项任务：

1. 作为 `W1` 的主 long-context family 来源
2. 作为 `W5` 的主 backend mismatch family 来源

如果这一组 benchmark 跑不出结果，全文的公开 benchmark 主结果会明显变薄。

## 4.2 第一批公开样本选择规则

第一批不按“任务名全覆盖”推进，而按下面三条筛样：

1. context 足够长，能形成可观 prefix reuse
2. 同一 context 下能关联多个公开 question / query
3. 样本本身来自 benchmark 原始发布，而不是手工扩写

固定保留样本时只做两种过滤：

- `group_size >= 2`
- `context_token_len >= long_prefix_threshold`

这里的 `group_size` 指同一公开 context 下可形成的请求数。

## 4.3 `prefix family` 构造规则

`LongBench` family 固定按下面四元组构造：

1. `dataset_name`
2. `context_hash`
3. `system_wrapper_version`
4. `api_contract_class`

其中：

- `context_hash` 直接来自 benchmark 原始 context 文本
- `system_wrapper_version` 固定为论文统一 wrapper，不能因样本个体变化
- `api_contract_class` 只区分 `generation` 和 `prompt_logprobs`

## 4.4 第一批实验角色

`LongBench` 第一批固定拆成两组：

1. `LongBench-W1`
   - 同一 family 下 `generation` 对照 `prompt_logprobs`
   - 目标是 prevalence + selective replay recovery
2. `LongBench-W5`
   - 相同 family 下 reference backend 对照 probe backend
   - 目标是 prevalence + twin-pool route recovery

## 4.5 当前仓库支持状态

当前仓库没有现成的 `LongBench` FalseNegativeKV runner。

因此必须新增：

- `longbench_family_builder.py`
- `run_longbench_fnkv.py`
- `analyze_longbench_fnkv.py`

建议统一放到：

- `benchmarks/false_negative_kv/public_bench/`

## 5. `MT-Bench` 实例化

## 5.1 角色定位

`MT-Bench` 不是当前论文的 headline benchmark，但它有两个优点：

1. 当前仓库已经有数据集 loader
2. 它提供公开多轮对话结构，适合做 `W1` 的 chat 辅验证

因此它的固定角色是：

- 作为 `LongBench-W1` 的辅验证
- 不是主 headline result 来源

## 5.2 当前仓库已有支持

当前仓库已经在
[datasets.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/benchmarks/datasets/datasets.py:2890)
中提供了 `MTBenchDataset`，默认使用：

- `philschmid/mt-bench`

并且当前实现已经能把公开样本转换为单轮 prompt。

## 5.3 第一批公开样本选择规则

第一批固定使用：

- `philschmid/mt-bench`
- 公开首轮 turn

原因：

- 本仓库已有数据路径
- 能快速复用现有 tokenizer / benchmark 基础设施

但也要明确边界：

- 这一批只能证明多轮 chat 语义的辅验证
- 不能单独承担长共享前缀主结果

## 5.4 `prefix family` 构造规则

`MT-Bench` family 固定按下面三元组构造：

1. `category`
2. `system_wrapper_version`
3. `turn_index`

若后续接入完整多轮版本，再追加：

4. `conversation_id`

## 5.5 下一步脚本需求

当前最小新增脚本只需要两个：

- `run_mtbench_fnkv.py`
- `analyze_mtbench_fnkv.py`

并尽量复用现有 dataset loader，而不是再写一套下载逻辑。

## 6. `BFCL` 实例化

## 6.1 角色定位

`BFCL` 是 `W2 tool-rich / agentic` 的主 benchmark。

这一组 benchmark 的意义不是补充花样，而是决定审稿人会不会继续把论文看成：

- 一个 `prompt_logprobs` microbenchmark paper

如果没有公开 tool-rich benchmark，全文会很难支撑“mixed-API serving”这个更大的 claim。

## 6.2 第一批公开样本选择规则

第一批固定优先选择：

1. 固定 tool schema 的公开样本
2. 同一 tool catalog 可被多个 query 复用的样本
3. multi-turn 或 function-calling 结构清晰的样本

第一批不追求覆盖全部 BFCL 子任务，先保证：

- `tool_schema_hash` 稳定
- `group_size >= 2`

## 6.3 `prefix family` 构造规则

`BFCL` family 固定按下面四元组构造：

1. `tool_schema_hash`
2. `system_wrapper_version`
3. `api_contract_class`
4. `tool_mode`

其中：

- `tool_schema_hash` 是主键
- query 本身不能进入 family key，否则 family 会被打散

## 6.4 当前仓库支持状态

当前仓库没有 BFCL 的现成 ingestion / serve runner。

因此这一组 benchmark 必须新增：

- `bfcl_family_builder.py`
- `run_bfcl_fnkv.py`
- `analyze_bfcl_fnkv.py`

此外还要补一个固定 wrapper 模块，用于：

- 把 BFCL 样本转成统一 OpenAI-compatible request

## 6.5 第一批论文角色

`BFCL` 第一批固定承担：

1. `W2` prevalence 主结果
2. `prompt_logprobs` 或相关 observation contract 的 recovery 补强

若 `BFCL` 跑通，整篇论文对公开 agentic workload 的说服力会明显增强。

## 7. `MTEB` 实例化

## 7.1 角色定位

`MTEB` 不是用来回答“embedding 模型分数高不高”，而是用来回答：

- 在公开文本 benchmark 上，token carrier 与 `prompt_embeds` carrier 是否会诱发不同的复用兑现行为

因此 `MTEB` 在本文中的固定角色是：

- `W3` prevalence 主结果
- P1 阶段默认不承担主 recovery 结果

## 7.2 当前仓库已有支持

当前仓库已经有两类 MTEB 相关工具：

1. embed 测试工具
   - [mteb_embed_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/tests/models/language/pooling_mteb_test/mteb_embed_utils.py:23)
   - 当前固定任务是 `STS12`
2. rerank / score 测试工具
   - [mteb_score_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/tests/models/language/pooling_mteb_test/mteb_score_utils.py:22)
   - 当前固定任务是 `NFCorpus`

这意味着：

- `MTEB` 是当前最容易起步的 `W3` 公开 benchmark

## 7.3 第一批公开样本选择规则

第一批固定用：

1. `STS12`
2. `NFCorpus`

理由不是它们“最好”，而是：

- 当前仓库已有直接可复用的本地工具
- 能快速把 carrier mismatch 的 prevalence 跑出来

## 7.4 `prefix family` 构造规则

`MTEB` family 固定按下面四元组构造：

1. `task_name`
2. `text_hash`
3. `carrier_class`
4. `system_wrapper_version`

其中 `carrier_class` 只区分：

- `tokens`
- `prompt_embeds`

## 7.5 下一步脚本需求

`MTEB` 这组 benchmark 需要的是轻量封装，而不是重写评测器。

固定新增：

- `run_mteb_fnkv.py`
- `analyze_mteb_fnkv.py`

并尽量直接调用现有本地测试工具，而不是再引入第二套 MTEB 封装。

## 8. `MMBench` 实例化

## 8.1 角色定位

`MMBench` 的固定角色是：

- 证明公开 multimodal benchmark 上也存在 false-negative prevalence

当前不要求它在 P1 阶段进入主 recovery 表。

## 8.2 第一批公开样本选择规则

第一批固定优先选：

1. 同一 image id 能对应多个问题的样本
2. image 本身稳定、文本 suffix 可变化的样本
3. 能通过官方工具链稳定下载和评测的样本

## 8.3 `prefix family` 构造规则

`MMBench` family 固定按下面四元组构造：

1. `image_id`
2. `vision_wrapper_version`
3. `text_wrapper_version`
4. `api_contract_class`

## 8.4 当前仓库支持状态

当前仓库没有现成的 `MMBench` FalseNegativeKV runner，也没有公开 multimodal family builder。

这一组 benchmark 默认依赖外部公开工具链：

- `VLMEvalKit`

因此在工程排期上，`MMBench` 固定放到 `P2`。

## 8.5 下一步脚本需求

只有在前面三组 benchmark 已经跑通后，才允许新增：

- `run_mmbench_fnkv.py`
- `analyze_mmbench_fnkv.py`

否则先不扩面。

## 9. 本仓库与 benchmark 的现状映射

为了避免继续发散，从现在开始固定采用下面的 readiness 判定。

| benchmark | readiness | 解释 |
| --- | --- | --- |
| `LongBench / LongBench v2` | `missing` | 方向最重要，但当前仓库没有现成 FalseNegativeKV runner |
| `MT-Bench` | `partial` | 已有 dataset loader，但还没有 family / FNKV runner |
| `BFCL` | `missing` | 公开 benchmark 已选定，但当前仓库完全缺 ingestion 与 runner |
| `MTEB` | `partial` | 已有本地测试工具，可快速转成 prevalence runner |
| `MMBench` | `missing` | 需要依赖外部多模态评测工具链，当前优先级最低 |

## 10. 立即执行的脚本排期

后续脚本排期固定为下面六步，不再改顺序。

1. `run_mtbench_fnkv.py`
2. `run_mteb_fnkv.py`
3. `run_longbench_fnkv.py`
4. `analyze_longbench_fnkv.py`
5. `run_bfcl_fnkv.py`
6. `run_mmbench_fnkv.py`

解释：

- `MT-Bench` 和 `MTEB` 先做，是因为当前仓库已有局部支持，能最快转出公开 benchmark 结果
- `LongBench` 虽然最重要，但需要从零补 family builder，因此排在第三步
- `BFCL` 与 `MMBench` 继续后置，避免一开始就被工具链复杂度拖死

## 11. 正文主结果的最低 benchmark 闭环

如果论文要进入“可以投稿”的状态，公开 benchmark 至少要闭环到下面这一步：

1. `LongBench-W1` 跑出 prevalence + recovery
2. `LongBench-W5` 跑出 prevalence + backend mismatch 对照
3. `BFCL-W2` 跑出 prevalence
4. `MTEB-W3` 跑出 prevalence

在这一步完成前：

- `MT-Bench` 只算辅证
- `MMBench` 只算补强，不算主线闭环

## 12. 失败条件与止损规则

公开 benchmark 路线从现在开始固定采用下面的止损规则。

1. 若 `LongBench` 无法构造出足够多 `group_size >= 2` 的 family，则必须先降级 `W1/W5` 主张，不允许继续假设长共享前缀在公开数据上天然成立。
2. 若 `BFCL` 公开 release 无法形成稳定的 tool-schema family，则 `W2` 需要降级为“公开 tool-rich prevalence 不充分”，而不是继续用自造 agent 任务硬撑。
3. 若 `MTEB` 上 token vs `prompt_embeds` carrier 差异极弱，则 `W3` 只保留为补充 family，不应继续在正文中高强度强调。
4. 若 `MMBench` 工具链成本过高，则可以把 `W4` 延后，但不能用 synthetic multimodal workload 顶替正文主结果。
