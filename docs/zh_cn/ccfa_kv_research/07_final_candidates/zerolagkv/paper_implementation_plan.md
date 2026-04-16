# ZeroLagKV：论文与实现推进方案

更新日期：2026-04-15

## 1. 执行结论

`ZeroLagKV` 还值得继续，但必须明显收窄 thesis，不能再把它写成宽泛的“更早复用 active request 的 KV”。

当前最安全、也最有学术成立性的版本是：

> 在异步 continuous batching 的 LLM serving 中，`decode-evolved prefix` 的可见性晚于调度决策，形成了独立的 `publication lag` 瓶颈；该瓶颈可以通过 visibility-aware publication protocol 与 bounded follower waiting 进行优化。

这条线从论文角度和实现角度都还能成立，但有三个前提：

1. 必须证明 `publication lag` 不是边角料，而是在真实 workload 中可重复出现、可量化。
2. 必须把贡献写成一个通用 runtime principle，而不是一个 `vLLM async loop` 小修补。
3. 必须证明收益不只体现在单一 TTFT 指标上，而是在 `TTFT / throughput / tail latency` 上有稳定改善。

如果做不到这三点，这个方向不够支撑 `OSDI/CCF-A` 主线，只能算一个不错的工程优化。

## 2. 这条线现在真正成立的研究问题

### 2.1 最强版本

最强、也最值得继续推进的研究问题不是：

- 活跃请求的 KV 能否更早复用。

而是：

- 当 runtime 采用异步调度时，调度决策看到的共享状态是滞后的；
- 新产生的可复用前缀虽然已经在 GPU 上算出来，但还没有进入其他请求可见的共享语义；
- 因此，prefix reuse 的上界不只由 cache size、eviction 或 routing 决定，还由 `publication timing` 决定。

这比“early reuse”更窄，但学术上更干净。

### 2.2 最弱版本

下面这些说法都不该再作为主张：

- `ZeroLagKV` 让 active request 的 KV 可以提前共享。
- `ZeroLagKV` 首次提出更早复用 prefix cache。
- `ZeroLagKV` 解决了 prefix cache reuse 的普适问题。

这些表述会直接撞上工业界已经公开的近邻能力，也很容易被评审压成“工程实现细节”。

## 3. 与公开相关工作的重合风险

## 3.1 已确认的危险近邻

### NVIDIA TensorRT-LLM Early Reuse

NVIDIA 在 2024-11-08 的公开博客里已经明确提出：

- 传统 KV reuse 通常要求前一个请求结束后才可复用；
- TensorRT-LLM 已支持在前一个请求仍在生成时就开始复用其 KV。

这意味着：

- 如果 `ZeroLagKV` 还停留在“请求没结束也能复用”这个层次，重合风险非常高；
- 这部分不能再作为论文主创新。

### TensorRT-LLM KV Cache Reuse 文档

TensorRT-LLM 官方文档在 2025-09-15 更新版本中仍写到：

- 传统 reuse 语义常常要求前一个请求先终止，后一个请求才能复用。

这说明工业界对“可见性时机”已经高度敏感，但公开资料仍主要停留在能力说明，而不是系统化 thesis。

这给 `ZeroLagKV` 留下的空间不是“更早复用本身”，而是：

- 异步调度下的 `publication lag` 抽象；
- 以及围绕它设计的 correctness protocol 与 scheduling policy。

## 3.2 不同但会挤压论文空间的近邻

### vLLM 技术报告

vLLM 技术报告已经明确承认异步调度下会出现“一步滞后”现象：

- 共享前缀的新请求可能只能命中旧前缀，而看不到刚在上一步生成的新 token。

这给了 `ZeroLagKV` 很强的动机来源，但也意味着：

- 如果我们的论文最后只是在复述这件事，没有提出新的协议和调度机制，就不够。

### KVCache Cache in the Wild, USENIX ATC 2025

这篇工作把“已完成请求的 KV cache”当作主流默认语义来研究，重点在 workload characterization 与 eviction。

它的重要性在于：

- 学界主流默认前提仍然是 completed-request reuse；
- 这恰好让 `publication lag` 成为一个尚未被主流系统论文充分单独建模的缺口。

### BatchLLM / PAT / 2026 年 KV scheduling 工作

这类工作分别从：

- global prefix sharing,
- shared-prefix execution,
- KV-aware scheduling / theoretical scheduling

几个方向压缩了“调度论文”的空间。

因此 `ZeroLagKV` 不能写成泛化的 scheduler paper，而必须把贡献锁定在：

- state visibility,
- publication protocol,
- lag-aware scheduling。

## 3.3 关于“确保没人做过”的真实判断

截至 2026-04-15，我没有找到与下面这个命题完全同构的 peer-reviewed 系统论文：

> 在异步 continuous batching 下，将 `decode-evolved prefix` 的可见性滞后作为独立瓶颈，并给出对应 publication protocol 与 follower waiting 机制。

但是，我也不能给出“确保没人做过”的保证，原因很直接：

1. 工业界已经公开了高度接近的 `Early Reuse` 能力。
2. 相关实现可能以博客、文档、专利、产品特性或未公开内部系统存在。
3. 最近一年 prefix sharing / KV scheduling / session state 方向更新很快，存在后续撞车风险。

因此更严谨的结论是：

- `ZeroLagKV` 仍有论文空间；
- 但只能在一个更窄、更精确定义的问题上站住；
- 不能把 novelty 建立在“没有任何前人工作”这个前提上。

## 4. 从论文角度，这条线应该怎么写

## 4.1 推荐的问题定义

建议把论文的核心问题写成：

> 异步 serving runtime 会把“状态产生”和“状态可见”分离。对 `decode-evolved prefix` 而言，这种分离会导致 follower request 错过本可利用的共享前缀，从而增加 prefill 重算、拉高 TTFT，并在 bursty conversational workload 中拖累吞吐。

这个问题定义有四个好处：

1. 它不与“是否允许 active reuse”直接等价。
2. 它能覆盖 `vLLM`，但又不被 `vLLM` 绑死。
3. 它把系统瓶颈放在 runtime semantics，而不是某个 kernel 或 block size。
4. 它天然对应 correctness protocol，而不仅是启发式优化。

## 4.2 推荐的论文主张

如果继续推进，我建议把主张收敛到下面三条：

1. `Publication lag` 是异步 LLM serving 中独立且可量化的性能瓶颈。
2. 通过 `visibility-aware publication protocol`，可以安全地把可复用前缀从 `private` 提升为 `published`，而不破坏现有 cache 语义。
3. 通过 `bounded follower waiting` 或 `lag-aware scheduling`，可以把“即将可见”的共享状态纳入调度收益计算，在不明显伤害吞吐的情况下改善 `TTFT / reuse hit length / tail latency`。

这里最重要的是：

- 第一条是问题发现；
- 第二条是系统机制；
- 第三条是端到端收益。

少任何一条，论文都容易站不稳。

## 4.3 推荐的题目风格

可以考虑下列风格：

- `ZeroLagKV: Visibility-Aware Publication of Decode-Evolved Prefix State in Async LLM Serving`
- `ZeroLagKV: Closing the Publication Lag for Shared Prefix State in Continuous Batching`

不要把题目写成：

- `Early Reuse for KV Cache`
- `Active KV Reuse in vLLM`

这会把论文主动送进工业近邻的火线。

## 4.4 OSDI/CCF-A 级别的成立条件

如果目标是 `OSDI/CCF-A`，论文至少需要满足下面四条。

### 条件一：问题必须具有一般性

不能只证明：

- `vLLM` 当前某段 async 实现会漏掉一步 prefix hit。

必须证明：

- 只要 runtime 存在 `schedule-before-publish` 的控制路径，就会出现同类 visibility lag。

也就是说，`vLLM` 应该是实例，不是论文本体。

### 条件二：协议必须比 patch 更强

不能只写：

- 把 `schedule()` 和 `update_from_output()` 换个顺序；
- 或在某个点补一次 `cache_blocks()`。

必须给出清楚的 protocol：

- 哪些状态可发布；
- 谁负责发布；
- 发布后谁可见；
- 遇到 preemption / abort / speculative decode 时怎样保证正确性。

### 条件三：收益必须跨指标

如果只有 TTFT 好看而吞吐掉得明显，论文很难成立。

至少要证明：

- TTFT 改善；
- prefix reuse hit length 增加；
- 吞吐不明显下降，最好上升；
- p95/p99 tail latency 不恶化。

### 条件四：必须解释 workload 边界

这不是“所有请求都收益”的工作。论文需要主动写清：

- 它更适用于多轮 chat、agent、burst follow-up、session continuation；
- 对单轮独立请求、纯静态 system prompt、低并发 workload 收益有限。

这不是缺点，前提是边界解释得足够清楚。

## 4.5 论文中不能碰的 claim

下面这些 claim 很危险，不建议写：

- 我们首次提出 active request KV reuse。
- 我们首次实现 in-flight KV sharing。
- 我们系统性领先工业界最新 prefix reuse 机制。
- 我们对所有 workload 都带来稳定吞吐提升。

更稳妥的写法应该是：

- 我们识别并形式化了 `publication lag`；
- 我们证明它在一类重要 conversational workloads 中影响显著；
- 我们提出 visibility-aware runtime protocol，在开源框架中实现并验证。

## 5. 从实现角度，这条线应该怎么做

## 5.1 第一性原理

`ZeroLagKV` 的底层事实只有一句话：

> follower request 能否复用，不只取决于状态是否已经算出来，还取决于 scheduler 在做决定时是否看得见它。

因此，第一版系统不应该试图“无条件更早发布一切”，而应该做两件更可控的事：

1. 明确哪些状态在语义上已经可发布；
2. 让 scheduler 在必要时愿意短暂等待这些状态变得可见。

## 5.2 推荐的状态机

第一版建议把 block 的可见性语义显式化为四档：

- `private`
  - 仅 producer request 自己可见。
- `publishable`
  - token 已最终确定，block hash 稳定，且不包含 speculative / draft 内容。
- `published`
  - 已进入共享目录，follower 可命中并增加 `ref_cnt`，但 producer 仍持有。
- `stable`
  - 已完全纳入现有 APC 生命周期，与普通共享 block 无异。

这里的关键不在于状态名，而在于把“可发布但尚未可见”从隐式过渡变成显式语义。

## 5.3 第一版原型不要追求的东西

第一版不要同时做：

- distributed publication；
- cross-node routing；
- speculative decode 深度耦合；
- learned waiting policy；
- 新的 eviction 体系；
- 多级 host/GPU tiering。

这些内容很诱人，但都会让原型爆炸，并把主命题从 visibility protocol 拉偏。

## 5.4 第一版最小原型

建议把原型拆成三个阶段。

### P0：只做观测，不改策略

目标是回答一个决定生死的问题：

> `publication lag` 到底有多常见，多大程度影响复用长度和 TTFT？

建议在 `vLLM` 中新增轻量 instrumentation，记录：

- 当前调度轮次是否存在“只差最新 decode block 就能命中”的 follower；
- 若等待一次 output update，可额外命中的 token 数；
- 这些 missed opportunities 对应的请求类型、共享前缀长度、等待时间；
- `lost_hit_tokens`、`potential_saved_prefill_tokens`、`potential_ttft_delta_us`。

如果 P0 发现这类机会很少，或者额外命中长度太短，这条线应立即止损。

### P1：只做 publish path 显式化

在不改大调度逻辑的前提下，把 async output update 后的发布语义显式化：

- 对 finalized 的 full blocks，立即标记为 `publishable -> published`；
- 记录发布时间戳、来源 request、可见 block 数；
- 不引入等待，不改现有调度优先级。

P1 的意义不是拿最好结果，而是验证：

- 显式 publication 模型是否和现有 cache 生命周期兼容；
- correctness invariant 能否站住。

### P2：加入 bounded follower waiting

当 scheduler 发现某个新请求与某个运行中 producer 的共享前缀“只差最后一个或少数几个 publishable blocks”时，允许它在一个很小的预算内等待：

- 例如等待一次 output future；
- 或等待不超过固定微秒预算。

等待收益函数可以先做成简单 rule-based：

- 预计新增命中 token 数大；
- 预计等待时间短；
- 当前队列压力可接受；
- follower 属于延迟敏感类型。

这一步才是 `ZeroLagKV` 真正的系统机制主体。

### P3：可选的 lag-aware scheduling

如果 P2 证明有效，再考虑更进一步的 scheduler integration：

- 把“即将发布的共享状态”作为调度收益项；
- 在多个 follower 之间优先调度可获得更长 prefix hit 的请求；
- 或给特定 producer/follower 对做轻量配对。

这一步不是第一版必须项，只有在 P2 已经证明 headroom 足够时才值得做。

## 5.5 在 vLLM 中的建议落点

基于当前代码，最自然的改动点如下。

### `vllm/v1/core/kv_cache_manager.py`

现有 `allocate_slots()` 已经会在请求尚未结束时调用 `cache_blocks()`，这说明：

- `vLLM` 已经有活跃请求中间态进入 cache 的基础能力；
- 但发布语义仍然偏隐式。

这里适合补的是：

- publish metadata；
- publish timestamp；
- 针对 `decode-evolved full blocks` 的更清晰发布入口。

### `vllm/v1/core/block_pool.py`

`touch()` 已允许 `ref_cnt > 0` 的 block 被其他请求继续共享。

这里要做的不是大改数据结构，而是明确：

- `published-but-active` block 的生命周期与计数语义；
- 发生 eviction / abort 时哪些 block 仍合法共享。

### `vllm/v1/core/sched/async_scheduler.py`

这是第一版最关键的位置。

当前 `_update_request_with_output()` 会在 output 回来后更新 request，并调用 `kv_cache_manager.cache_blocks(...)`。

这里适合接入：

- publish event 记录；
- `publishable -> published` 过渡；
- P0/P1 所需的 lag instrumentation。

### `vllm/v1/engine/core.py`

当前 async batch queue 的控制流天然允许“先调度，再等上一轮结果”。

这正是 `publication lag` 的来源之一。

这里最适合做的是：

- P0 统计：一次调度决策是否发生在潜在 publish 之前；
- P2 等待策略：在发现高收益 follower 时，是否允许一次 bounded wait；
- 保持默认 fast path 不受影响。

### 新增模块建议

建议新增一个很薄的目录，例如：

- `vllm/v1/core/zerolag/`

里面只放：

- `types.py`
  - 定义 publication state、event、统计字段；
- `manager.py`
  - 管理 publish bookkeeping 与 lag opportunity tracking；
- `policy.py`
  - 放最小 rule-based waiting policy。

不要把逻辑直接塞满 `scheduler.py` 或 `kv_cache_manager.py`，否则后面会很难收敛。

## 5.6 正确性约束

如果没有清楚的 correctness invariant，这条线论文上会立刻失分。

第一版至少要守住下面五条：

1. 只允许 `finalized` token 对应的 full block 进入 `publishable`。
2. 含 speculative / draft token 的 block 绝不允许对 follower 可见。
3. 一旦 block 进入 `published`，其 hash 与 token 边界必须不可变。
4. producer 后续被抢占、终止或 abort，不应让已经发布且有效的 prefix block 失效。
5. publish 与 follower 命中之间的并发关系要么由单线程 scheduler 保证，要么通过显式锁/顺序保证。

这五条在论文里最好写成协议不变量，而不是实现备注。

## 6. 实验应该怎么设计

## 6.1 先做 headroom 证明

真正的第一实验不应该是端到端跑分，而应该是：

- 当前 `vLLM` async serving 中，有多少 prefix reuse 机会因为 publication lag 丢失？

这个 headroom 证明是决定是否继续的第一关。

## 6.2 建议 workload 组

至少要覆盖三类：

1. 多轮 chat / conversation continuation
2. agent / tool-use follow-up
3. bursty shared-prefix synthetic trace

同时保留一组对照：

4. 单轮独立请求或纯静态 system prompt workload

这样才能证明它的收益边界是可解释的，而不是挑 workload 讲故事。

## 6.3 核心指标

建议至少记录：

- `TTFT`
- `TPOT`
- `throughput`
- `p95/p99 latency`
- `prefix hit length`
- `lost_hit_tokens`
- `saved_prefill_tokens`
- `scheduler overhead`
- `wait_budget_consumption`

其中 `lost_hit_tokens` 和 `saved_prefill_tokens` 是这篇论文最关键的自定义指标。

## 6.4 baseline 设计

最低限度建议有下面四组：

1. 原生 `vLLM async scheduler`
2. 只做 P1 publication bookkeeping，不做等待
3. `ZeroLagKV` P2：bounded follower waiting
4. Oracle 上界
   - 假设 follower 总能在最优时刻看到刚发布的新前缀，用来衡量理论 headroom

如果后面资源允许，再加：

5. `vLLM` 非异步/更保守调度配置
6. 其他公开系统作为外部参考

Oracle 很重要，因为它决定这条线到底值不值得深挖。

## 6.5 论文里最关键的一张图

我认为最关键的一张图不是传统吞吐柱状图，而应该是：

- 横轴：follower 等待预算
- 纵轴：额外 prefix hit token / TTFT 改善 / 吞吐变化

这张图能直接回答评审最关心的问题：

- 你到底是在拿多少等待换多少复用？

## 7. 值不值得继续做

我的判断是：

- 值得继续；
- 但必须以 `P0 instrumentation` 为第一优先级；
- 在 P0 数据出来前，不应该继续膨胀机制设计。

更直接一点说：

- 如果 P0 显示 `publication lag` 只贡献很小的 missed reuse，`ZeroLagKV` 不适合作为主线；
- 如果 P0 显示在 conversational / agent trace 中存在稳定且可观的 lost-hit headroom，这条线就值得继续推进到 P2。

## 8. 接下来建议怎么做

建议按下面顺序推进。

### 第一周

1. 在 `async_scheduler.py` 和 `engine/core.py` 加 P0 instrumentation。
2. 产出一版 trace-level 统计，确认 lost-hit headroom。
3. 明确 `publishable` 的最小定义，只允许 finalized full blocks。

### 第二周

1. 实现 P1 publish bookkeeping，不改策略。
2. 验证 correctness invariant，不让 speculative 路径穿透。
3. 产出第一版 publication timeline 可视化。

### 第三周

1. 实现 P2 的 rule-based bounded waiting。
2. 跑多轮 chat / agent / synthetic 三类 workload。
3. 看 TTFT 改善是否能在吞吐基本不掉的条件下成立。

### 第四周

1. 决定是否继续做 P3 lag-aware scheduling。
2. 同时开始准备论文摘要、问题定义和实验图模板。
3. 若 headroom 不足，及时转向，不要继续追加机制复杂度。

## 9. 当前最推荐的推进判断

如果现在必须做一个选择，我的建议是：

- 不放弃 `ZeroLagKV`；
- 但立刻把它从“泛化 early reuse”重写为“async publication lag”；
- 先做 P0，再决定是否把它当作 `OSDI/CCF-A` 主线。

这既是最学术上稳妥的写法，也是最工程上不浪费时间的路径。

## 10. 参考来源

- NVIDIA TensorRT-LLM Early Reuse blog, 2024-11-08  
  https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/
- TensorRT-LLM KV Cache Reuse docs, last updated 2025-09-15  
  https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html
- vLLM Technical Report, EECS-2025-192  
  https://www2.eecs.berkeley.edu/Pubs/TechRpts/2025/EECS-2025-192.pdf
- KVCache Cache in the Wild: Characterizing and Optimizing KVCache Cache at a Large Cloud Provider, USENIX ATC 2025  
  https://arxiv.org/abs/2506.02634
- BatchLLM: Optimizing Large Batched LLM Inference with Global Prefix Sharing and Throughput-oriented Token Batching  
  https://arxiv.org/abs/2412.03594
- Competitive Non-Clairvoyant KV-Cache Scheduling for LLM Inference, 2026  
  https://arxiv.org/abs/2601.22996
- PAT, ASPLOS 2026  
  https://arxiv.org/abs/2511.22333
