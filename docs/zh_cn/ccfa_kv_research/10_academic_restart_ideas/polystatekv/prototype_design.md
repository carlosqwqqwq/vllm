# PolyStateKV：vLLM 最小原型设计

## 1. 这份设计要解决什么

这份文档不再讨论宽泛的“多状态形态很有前景”，而是把 `PolyStateKV` 第一版原型收缩成一个**能在 `vLLM` 里落地、能支撑论文主张、又不至于把实现做爆炸**的最小系统。

第一版只证明一个命题：

> 在相同显存预算下，把可复用状态固定成单一表示并不是最优默认前提；允许 runtime 在两种状态形式之间做在线选择，可以改善 `TTFT / throughput / memory` 的整体折中。

因此，这份原型设计坚持三条边界：

1. 只做两种状态形式：
   - `KV_FULL`
   - `HIDDEN_CKPT`
2. 只在 `scheduler` 层做在线选择，不把 selector 塞进各个底层热路径。
3. 只改与 `PolyStateKV` 直接相关的最小模块，不把整个 `vLLM` cache 子系统重写一遍。

## 2. 原型边界

### 2.1 第一版的状态形式

第一版建议只保留：

- `KV_FULL`
  - 使用现有 `KVCacheManager` 管理的 GPU-resident full KV。
- `HIDDEN_CKPT`
  - 存储为 host-side pinned memory 中的轻量 hidden-state checkpoint。
  - 语义上它仍然表示“同一个可复用上下文状态”，只是物理形态不再是 full KV。

这里的 `HIDDEN_CKPT` 不把目标写成一个大而全的 activation-restoration 系统。第一版更稳妥的定义是：

- 对可复用 prefix，在 prefill 完成后额外导出一份**可用于重建 KV 的 hidden-state 表示**
- 该表示放在 CPU/pinned host memory
- 当 selector 决定不继续占用 GPU full KV 时，把状态降级到 `HIDDEN_CKPT`
- 后续若该 prefix 再被命中，则通过 rematerialization 重新回到 `KV_FULL`

这里要特别防止一个常见误解：

- 第一版的 `HIDDEN_CKPT` **不是**“只存最后一层 hidden state”
- 它更接近“从若干选定层导出的 layer-scoped hidden-state bundle”

原因很简单：  
如果目标是后续重新 materialize 出可执行的 `KV_FULL`，那么只保存最终 token hidden states 在大多数模型里并不足以恢复所有层的 KV 语义。  
因此，第一版原型应把 `HIDDEN_CKPT` 明确定义为：

- 来自若干选定层的 auxiliary hidden states
- 或等价的、足以驱动 cache rebuild 的中间表示

而不是宽泛地把任何 hidden tensor 都叫做 checkpoint。

### 2.2 第一版的作用域

第一版不追求一个全局跨所有 prefix 的复杂统一对象图，而是先做**prefix-scoped reusable state object**：

- 语义身份由 prefix 本身决定
- 生命周期由 scheduler 管理
- 物理载体由 `KVCacheManager` 和 `HiddenCheckpointStore` 分别承载

这就足以证明“state form 应由 runtime 选择”，而不需要一开始就把多租户、分布式或异构设备都做完。

### 2.3 第一版明确不做什么

第一版明确不做：

- activation restoration 多形态并存
- compressed KV / low-rank KV
- 分布式状态目录服务
- learned selector
- 改模型结构或改 attention 数学语义

如果这些内容一起做，`PolyStateKV` 很容易从一篇系统论文原型，膨胀成一个无法收敛的大工程。

### 2.4 第一版实现闸门

第一版虽然不改模型数学语义，但**原型落地仍然有模型族边界**。  
从当前 `vLLM` 代码看，比较现实的落脚点不是凭空造一套 hidden capture 路径，而是优先借力现有的：

- `aux_hidden_states`
- `extract_hidden_states`
- cache-only layers

这意味着第一版最稳妥的实现边界应当是：

1. 先只支持 `vLLM` 当前已经支持 aux hidden state 输出的模型族
2. 先固定一组 `eagle_aux_hidden_state_layer_ids` 或等价层选择
3. 先证明“统一状态对象 + 在线 form selection”成立，再讨论更普适的模型覆盖

因此，论文主张可以是普适的；但 prototype scope 应主动写成：

> 当前 prototype 仅在具备 auxiliary hidden-state extraction 支撑路径的模型族上实现。

## 3. 统一状态对象怎么落

### 3.1 语义身份

`PolyStateKV` 第一版的核心不是“多一种 cache”，而是把“可复用状态”从单一 `KV block` 提升成一等对象。

最小实现里，这个对象的身份建议直接复用 `vLLM` 已有的 prefix 语义锚点：

- `cache_salt`
- `lora_request` 或其派生标识
- `block_hashes`
- prefix 对应的 token 边界

也就是说，第一版不发明另一套 identity system，而是沿用 `Request.block_hashes` 对 prefix 的语义划分。

建议新增的最小数据结构如下：

```python
class StateForm(Enum):
    KV_FULL = "kv_full"
    HIDDEN_CKPT = "hidden_ckpt"


@dataclass(frozen=True)
class SemanticStateKey:
    cache_salt: str | None
    lora_id: int | None
    prefix_block_hashes: tuple[BlockHash, ...]
    num_tokens: int


@dataclass
class HiddenCheckpointRef:
    checkpoint_id: str
    num_tokens: int
    num_layers: int
    hidden_bytes: int
    dtype: str
    device: str  # 第一版固定为 "cpu-pinned"


@dataclass
class PolyStateEntry:
    key: SemanticStateKey
    current_form: StateForm
    kv_block_ids: tuple[list[int], ...] | None
    hidden_ref: HiddenCheckpointRef | None
    num_tokens: int
    kv_bytes: int
    hidden_bytes: int
    last_access_ts: float
    ewma_reuse_gap_s: float | None
    materialization_cost_us: float | None
    sla_class: str
    pinned: bool = False
```

### 3.2 为什么状态对象不直接塞进 `KVCacheManager`

`KVCacheManager` 现在负责：

- block 分配
- prefix hit 查找
- 物理 KV block 生命周期

它并不适合直接承担：

- 多形态状态目录
- hidden checkpoint 生命周期
- online form selection

因此第一版应该把 `PolyStateEntry` 放在一个新的 scheduler-side manager 中，而不是侵入式改造 `KVCacheManager` 内部数据结构。

最小落点建议：

- 新增 `vllm/v1/core/polystate/`
  - `types.py`
  - `store.py`
  - `selector.py`
  - `manager.py`

其中：

- `types.py`
  - 放 `StateForm / SemanticStateKey / PolyStateEntry`
- `store.py`
  - 管理 host-side hidden checkpoint 的分配、释放、bytes accounting
- `selector.py`
  - 只放 form selection 逻辑
- `manager.py`
  - 负责把 request、kv manager、selector、checkpoint store 串起来

这会比把逻辑硬塞回 `scheduler.py` 或 `kv_cache_manager.py` 更干净，而且 diff 更可控。

## 4. vLLM 里改哪些模块

下面按“第一版必须改”“第一版尽量不改”来收敛实现面。

### 4.1 必须新增的模块

| 模块 | 作用 | 第一版要求 |
|---|---|---|
| `vllm/v1/core/polystate/types.py` | 定义统一状态对象与 form 枚举 | 必须新增 |
| `vllm/v1/core/polystate/store.py` | 管理 CPU pinned hidden checkpoint | 必须新增 |
| `vllm/v1/core/polystate/selector.py` | 放最小 rule-based selector | 必须新增 |
| `vllm/v1/core/polystate/manager.py` | scheduler-side 统一入口 | 必须新增 |

### 4.2 必须修改的已有模块

#### `vllm/v1/request.py`

这里建议只补很轻的 request-level hints，不把 request 本身变成状态管理器。

建议新增字段：

- `semantic_state_key: SemanticStateKey | None`
- `polystate_sla_class: str`
- `polystate_last_form: StateForm | None`
- `polystate_hit: bool`

这些字段的目标只有两个：

1. 让 request 能把 prefix 语义键带到 scheduler
2. 让 profiling 能记录“该请求是如何命中 / 恢复状态的”

#### `vllm/v1/core/sched/scheduler.py`

这是第一版最重要的接入点。

第一版建议只在三个时机调用 `PolyStateManager`：

1. **新请求入队 / prefix 匹配前**
   - 根据 `SemanticStateKey` 查询是否已有 `KV_FULL` 或 `HIDDEN_CKPT`
2. **prefill 完成且状态变得可复用时**
   - 决定该状态继续保留 `KV_FULL`，还是降级到 `HIDDEN_CKPT`
3. **显存压力升高时**
   - 从可降级状态中选出 demotion 候选

换句话说，第一版 selector 只在 scheduler 的低频决策点上运行，不进入 token-level 热路径。

建议在 `scheduler.py` 里补的最小接口：

- `self.polystate_manager = PolyStateManager(...)`
- `on_request_arrival(request)`
- `on_prefill_state_reusable(request, blocks)`
- `on_memory_pressure()`
- `on_request_finished(request)`

#### `vllm/v1/core/kv_cache_manager.py`

这里不要把 form selection 逻辑塞进来，只补两个能力：

1. 对某个 request/prefix 的当前 KV 占用给出稳定估计
2. 允许 scheduler/manager 以受控方式拿到该状态对应的 block id 集合

最小需要的辅助接口可以是：

- `estimate_kv_bytes(blocks) -> int`
- `get_block_ids_for_request(request) -> tuple[list[int], ...]`
- `free_request_blocks_for_demote(request)` 或等价 helper

核心原则是：

- `KVCacheManager` 继续管 “KV 物理块”
- `PolyStateManager` 管 “状态对象是否还用 KV 这种形式存在”

#### `vllm/v1/worker/gpu_model_runner.py`

这是第一版第二个关键改动点，因为 `HIDDEN_CKPT` 需要一个最小的 capture / rematerialize 钩子。

第一版建议只支持一个固定实现：

- 在 prefill 结束时按 prefix 导出 checkpoint
- 在命中 `HIDDEN_CKPT` 时走一条专门的 rematerialization path

这里最重要的是**把实现固定在单一 checkpoint 语义上**，而不是一开始支持多种 checkpoint 粒度。

更准确地说，第一版应把 worker 侧 checkpoint 语义钉死为：

- 从选定 auxiliary layers 抽取 hidden states
- 按 prefix token 范围组织成可持久化 bundle
- 后续以 cache rebuild 的方式重建目标 KV

这里可以明确借鉴当前 `extract_hidden_states` 相关机制，但不要在文档里把它误写成“现成就能直接复用的完整方案”。  
它更像是一个实现抓手，而不是已经现成存在的 `PolyStateKV` 子系统。

建议最小新增两个 worker 侧接口：

- `capture_hidden_checkpoint(request_id, token_range) -> HiddenCheckpointRef`
- `rebuild_kv_from_hidden(hidden_ref, request_id, slot_mapping) -> None`

这里要特别注意分层边界：

- `KVCacheBlocks` 与 block 分配仍由 scheduler / `KVCacheManager` 负责
- worker 侧只负责把已经分配好的 KV slot 重新填充出来

也就是说，`HIDDEN_CKPT -> KV_FULL` 的恢复流程应当是：

1. scheduler 先为该 request 分配目标 KV blocks
2. worker 再根据 `slot_mapping` 把 hidden checkpoint 重建进这些 blocks

设计上不要直接复用 `sequence.IntermediateTensors` 作为长期存储对象；可以借鉴它“tensor dict 容器”的形式，但持久化的 checkpoint 应该有独立的 `HiddenCheckpointRef` 和 store。

另外，这里还应明确一个 prototype 限制：

- 如果目标模型当前不支持 aux hidden states / cache-only hidden caching 路径
- 则第一版不应硬扩模型支持面
- 而应把该模型判定为“超出 prototype scope”

这比为了“看起来普适”而把实现复杂度推高要稳得多。

#### `vllm/v1/metrics/stats.py`

第一版需要一个单独的 `PolyStateStats`，不要把所有指标揉进现有 `PrefixCacheStats`。

建议新增：

```python
@dataclass
class PolyStateStats:
    selector_calls: int = 0
    selector_time_us: float = 0.0
    demotions_to_hidden: int = 0
    restorations_from_hidden: int = 0
    retained_kv_states: int = 0
    retained_hidden_states: int = 0
    hidden_bytes: int = 0
    wrong_form_decisions: int = 0
```

并在 `SchedulerStats` 中增加一个可选字段：

- `polystate_stats: PolyStateStats | None`

#### `vllm/v1/metrics/loggers.py`

这里只做两件事：

1. 把 `PolyStateStats` 暴露到 logging / prometheus
2. 保持与现有 `prefix_cache_stats / kv_connector_stats / cudagraph_stats` 的并列关系

第一版不需要重做 metrics framework，只在现有 logger 注册路径上追加即可。

### 4.3 第一版尽量不改的模块

#### `vllm/v1/simple_kv_offload/manager.py`

第一版**尽量不把 hidden checkpoint 直接塞进 `simple_kv_offload` 协议**。原因很简单：

- `simple_kv_offload` 当前抽象的是 block-level KV
- `HIDDEN_CKPT` 不是 block-level KV 的另一种摆放位置
- 如果强行复用，会把“状态形态切换”误写成“另一种 KV offload backend”

更稳妥的做法是：

- 借鉴其 host-side capacity accounting 方式
- 但 hidden checkpoint 仍由 `PolyStateManager + HiddenCheckpointStore` 自己维护

#### `vllm/v1/kv_offload/spec.py`

第一版也不建议在 `OffloadingSpec` 层做统一多形态抽象。

原因是论文第一版要证明的是：

- online state-form selection 的系统价值

而不是：

- `vLLM` 所有 offload 接口的统一重构

后者会把工程面放大很多，但对论文主张增益有限。

## 5. 最小状态生命周期

第一版建议把状态生命周期收成下面五步：

1. `DISCOVER`
   - 新请求到达，scheduler 依据 `SemanticStateKey` 查状态目录
2. `ATTACH`
   - 若命中 `KV_FULL`，走现有 prefix/KV 快路径
   - 若命中 `HIDDEN_CKPT`，登记一次待 rematerialize
3. `MATERIALIZE`
   - 如果当前状态是 `HIDDEN_CKPT`，在真正入执行前恢复成 `KV_FULL`
4. `DEMOTE`
   - 如果状态已经可复用且 GPU 压力升高，则将其降级为 `HIDDEN_CKPT`
5. `RECLAIM`
   - 如果 hidden store 也达到容量上限，按 reuse horizon 或 LRU 进一步删除

第一版有一个非常重要的约束：

- `HIDDEN_CKPT` 不是执行态
- 真正执行 decode 前必须回到 `KV_FULL`

这样可以大幅降低对下游执行器和 attention kernel 的侵入性。

## 6. selector 怎么最小实现

### 6.1 不做 learned selector

第一版 selector 必须是 rule-based，可解释、可消融、好实现。

它不该回答复杂问题，只需要回答两个小问题：

1. 一个可复用状态在当前是否应该继续占用 `KV_FULL`
2. 当它再次命中时，恢复后是否继续倾向保留 `KV_FULL`

### 6.2 selector 输入

第一版输入信号建议只保留四类：

1. `reuse_horizon`
   - 同一 `SemanticStateKey` 的历史复用间隔 EWMA
   - 若没有历史，则使用保守默认值
2. `memory_pressure`
   - `kv_cache_usage`
   - free block 水位
3. `materialization_cost`
   - 从 `HIDDEN_CKPT` 恢复到 `KV_FULL` 的估计开销
4. `sla_class`
   - `interactive / default / batch`
   - 第一版可以由 request 长度和调用来源粗略推断，不必引入复杂 SLO 框架

### 6.3 selector 输出

第一版 selector 只输出三种动作：

- `KEEP_KV`
- `DEMOTE_TO_HIDDEN`
- `DELETE_STATE`

其中 `DELETE_STATE` 只在 hidden store 也到水位线时触发。

### 6.4 最小决策规则

可以把第一版写成一个很简单的两级规则：

```python
def choose_form(entry, pressure, now):
    if pressure.kv_usage < LOW_WATERMARK:
        return KEEP_KV

    if entry.sla_class == "interactive" and \
       entry.materialization_cost_us > INTERACTIVE_BUDGET_US:
        return KEEP_KV

    expected_penalty = reuse_probability(entry.ewma_reuse_gap_s, now) \
        * entry.materialization_cost_us

    memory_saving = entry.kv_bytes - entry.hidden_bytes

    if memory_saving * pressure_factor(pressure) > expected_penalty:
        return DEMOTE_TO_HIDDEN

    return KEEP_KV
```

这个 selector 的价值不在于它“聪明”，而在于它能准确支撑论文问题：

- 为什么固定单一形式不对
- 为什么 runtime 需要一个在线 form selector
- 为什么收益来自统一成本比较，而不是硬编码一个阈值

### 6.5 为什么 selector 不应直接嵌进 eviction policy

如果把 selector 写成 `KVCacheManager` 的 eviction policy 变体，论文叙事会迅速变成：

- “又一个 cache eviction heuristic”

这会削弱 `PolyStateKV` 的学术命题。  
因此第一版要明确区分：

- eviction 是物理 block 管理问题
- form selection 是 reusable state 的表示选择问题

两者相关，但不是同一个问题。

## 7. 最小实现步骤

### 7.1 阶段 A：只加状态对象与观测，不改行为

先做：

- `SemanticStateKey`
- `PolyStateEntry`
- `PolyStateStats`
- scheduler 中的目录记录

但不真的做 demotion。  
这样可以先回答一个基本问题：

- 不同 prefix 的 reuse gap 分布是否真的存在明显分层

### 7.2 阶段 B：实现两个静态基线

然后只实现：

- `Static-KV`
- `Static-Hidden`

先不加在线 selector。  
这一步的意义是确认：

- 两个极端策略是否真的各自有明显短板

### 7.3 阶段 C：加固定阈值 selector

在确认两边都各有短板后，再加：

- `Threshold-Demotion`

它是 `PolyStateKV` 前的中间控制组。

### 7.4 阶段 D：再加 online selector

最后才上：

- `PolyStateKV` 的最小 online selector

这条顺序非常重要，因为论文最后需要回答的不是：

- “我们写了个 selector”

而是：

- “为什么 online selection 比单一形式和固定阈值都更合理”

## 8. 风险与对应约束

### 8.1 最大实现风险

第一版最大的实现风险其实不在 scheduler，而在 worker 侧的 checkpoint capture / rematerialization。

所以必须守住两个约束：

1. checkpoint 语义固定
2. `HIDDEN_CKPT` 永远不是执行态，只是过渡态

这样可以把 model runner 改动控制在一个新增路径里，而不是把正常 decode 路径全面改写。

### 8.2 最大论文风险

最大的论文风险是被理解成：

- “又一个 hybrid cache”

所以在原型实现层也要主动体现这一点：

- selector 独立存在
- 状态对象独立存在
- 不把 `HIDDEN_CKPT` 伪装成另一种 KV offload

## 9. 当前建议

如果现在就进入实现，我建议按下面顺序推进：

1. 先新增 `v1/core/polystate/` 四个文件，建立对象层和 selector 层
2. 再在 `scheduler.py` 中接一条只记录不改行为的观测链
3. 然后补 `Static-KV / Static-Hidden`
4. 最后才做 online selector

这条路径的好处是：

- 每一步都有可测结果
- 每一步都能支撑论文证据链
- 不会一开始就把 `vLLM` 改成不可控的大 diff
