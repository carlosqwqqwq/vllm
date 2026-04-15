# SessionKV：面向 Chat 与 Agent 的状态化 Fork-Join KV 服务

## 1. 一句话定位

`SessionKV` 的核心主张是：

> 现代 LLM 服务的真实工作负载是 stateful、branching、long-lived 的，但今天的 `vLLM` 和大多数 serving engine 仍然基本假设请求是 stateless replay。`SessionKV` 要把 KV state 提升成一等公民，让 `snapshot / fork / rollback / commit` 成为服务接口的一部分。

## 2. 为什么这是对问题定义的升级

目前几乎所有 prefix cache 工作，哪怕很强，也默认一个前提：

- 请求来了
- 带上一整段 prompt
- 引擎尝试从 cache 里“捞回”能复用的 prefix

也就是说，系统把“状态”藏在 cache 后面，把 API 继续维持为“无状态重放”。

但对 Chat / Agent 来说，真实执行结构明明是：

- 一个长期存在的会话状态
- 中间会出现工具调用暂停
- 会发生 retry / rollback
- 会发生多分支探索
- 会有 A/B 比较、规划树展开、并行子代理

这些都说明：**prefix cache 不应该只是一个被动优化，而应该升级成会话状态系统**。

## 3. 核心洞见

如果应用本身就是 stateful 的，那么最自然的服务接口不该是：

- “每轮都把整个历史重新送进来”

而应该是：

- `create_session`
- `append_user_turn`
- `snapshot`
- `fork`
- `rollback`
- `commit`

然后由引擎内部维护一个 copy-on-write 的 `KV state DAG`。

一句话说，`SessionKV` 的核心洞见是：

> 不要再把状态假装成字符串前缀；应该把状态显式建模成可分叉、可提交、可回滚的 KV object。

## 4. 系统设计

## 4.1 Session handle

每个会话由一个稳定的 `session_id` 标识。客户端不再必须在每轮请求里重放全部历史，而是通过：

- `session_id`
- 增量输入
- 可选的分叉/回滚操作

来推进状态。

## 4.2 KV state DAG

内部不再只是一棵前缀树，而是一个会话状态 DAG：

- 节点表示某个逻辑会话状态
- 边表示 append / tool result / branch / merge
- 节点持有对共享 KV blocks 的引用
- 分叉采用 copy-on-write，而不是整段复制

这使得：

- 会话主线和子分支共享大部分状态
- retry / A-B 测试 / planner branch 不会把 prefix cache 搞成一堆难以解释的近似命中

## 4.3 Fork / Join / Rollback

这是 `SessionKV` 真正和普通 APC 拉开差距的地方。

系统显式支持：

- `fork(parent_session, branch_name)`
- `rollback(session, snapshot_id)`
- `commit(branch, target)`
- `drop(branch)`

这些操作都在 KV DAG 上完成，而不是要求上层应用自己用字符串重放来模拟。

## 4.4 Tool-aware suspension

Agent workload 最常见的低效点之一，是 tool call 会把连续性打断。

`SessionKV` 不把这件事当成“cache eviction 之后再想办法命中”，而是把它当成：

- 一次显式的 session suspension
- 系统知道当前状态是“暂停、可能恢复、可能分叉”

这使得 TTL、pinning、恢复优先级这些策略都能建立在**会话语义**上，而不是孤立请求之上。

## 4.5 Session-aware scheduling

一旦 session 成为一等公民，scheduler 就多了新的优化空间：

- 同一 session 的连续恢复优先
- 来自同一根状态的分支共享调度 hint
- 对即将 resume 的 session 保持更高的 cache residency

这不是简单的 prefix-aware scheduling，而是**state-aware scheduling**。

## 5. 为什么它和已有工作不一样

和 `CachedAttention / Continuum` 不同：

- 它不只是“把多轮历史保留更久”；
- 它把“会话状态”正式变成服务抽象。

和 `SegmentKV` 不同：

- 它不复用任意 segment；
- 它只管理 exact、可验证的会话状态演化。

和 `AffinityKV / TierKV` 不同：

- 那些方向优化的是状态怎么放、怎么搬、怎么路由；
- `SessionKV` 优化的是状态本身怎么被建模和暴露。

## 6. 为什么这条线最像大论文

我认为它是三个新方案里最有 `OSDI` 气质的一个，原因在于：

- 它改变的是服务抽象，而不只是 cache policy；
- 它能同时带来性能收益和能力扩展；
- 它能让 `vLLM` 从“无状态推理引擎”更进一步，变成“状态化 LLM 运行时”。

这和很多顶会系统论文的形态更接近：

- 不是再优化一个子策略
- 而是重写系统边界与一等公民对象

## 7. 在 vLLM 里怎么落地

可能的模块包括：

- `vllm/entrypoints/openai/*`
  - 增加 session-aware API 或兼容层
- 新增 `session_manager`
  - 维护 session metadata、snapshot、branch DAG
- `vllm/v1/request.py`
  - 请求不再只是全量 token 流，也可以是 session delta
- `vllm/v1/core/kv_cache_manager.py`
  - 管理 shared state references 与 copy-on-write blocks
- `vllm/v1/core/sched/scheduler.py`
  - 用 session continuity 做 admission / preemption hint

## 8. 风险与挑战

### 8.1 风险

- 对 API 边界改动更大，集成成本高
- 需要设计兼容模式，不能要求所有用户都改写调用方式
- 评审会追问：为什么这不是 application framework 的事情

### 8.2 应对方式

论文里必须把问题讲清楚：

- 今天大量 agent / chat 系统已经天然是 stateful 的
- 但底层 serving 还在假装它们是 stateless prompt replay
- 这种抽象错位本身带来了大量无效计算和不稳定 latency

只要这层讲透，`SessionKV` 的研究价值就会很强。

## 9. 我的判断

如果你要我在“新意、系统高度、和前一轮方案的区分度”里只选一个最强的，我会把票投给 `SessionKV`。

它不一定是最容易先做出来的，但它最像一篇：

- 问题定义足够大
- 改变系统抽象
- 同时又能直接增强 `vLLM`

的系统类 CCF-A / OSDI 风格论文。
