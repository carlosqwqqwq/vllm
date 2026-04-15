# 建议阅读顺序

下面给出一条偏“工程理解优先”的阅读路径。目标不是一次读完所有细节，而是尽快建立完整心智模型。

## 第一阶段：先确认主线入口

1. `docs/usage/v1_guide.md`
2. `vllm/__init__.py`
3. `vllm/entrypoints/llm.py`
4. `vllm/engine/llm_engine.py`
5. `vllm/engine/async_llm_engine.py`

这一阶段要回答的问题只有两个：

- 当前外部 API 最终是不是都落到 `v1`？
- 离线入口和在线入口分别是谁？

## 第二阶段：理解请求如何进出引擎

1. `vllm/v1/engine/input_processor.py`
2. `vllm/v1/engine/output_processor.py`
3. `vllm/v1/engine/llm_engine.py`
4. `vllm/v1/engine/async_llm.py`
5. `vllm/entrypoints/openai/api_server.py`

这一阶段关注的是“请求生命周期”，不是底层执行细节。

读完后最好能自己复述：

- 输入在什么时候被标准化？
- 输出在什么时候被 detokenize 和聚合？
- 同步和异步接口的差别主要在哪里？

## 第三阶段：抓住最核心循环

1. `vllm/v1/engine/core.py`
2. `vllm/v1/core/sched/scheduler.py`

这是最关键的一步。

要重点看三件事：

- `EngineCore` 初始化了哪些核心组件。
- `step()` 的调度、执行、回写顺序。
- `Scheduler` 如何用统一 token budget 处理 prefill、decode、缓存和恢复执行。

如果这一段读通，vLLM 的主干就基本通了。

## 第四阶段：补执行后端

1. `vllm/v1/executor/abstract.py`
2. `vllm/v1/executor/uniproc_executor.py`
3. `vllm/v1/executor/multiproc_executor.py`
4. `vllm/v1/worker/worker_base.py`
5. `vllm/v1/worker/gpu_worker.py`
6. `vllm/v1/worker/gpu/model_runner.py`

这里的目标是看清：

- 调度结果如何被送往 worker
- worker 如何持有模型与设备状态
- 单进程、多进程、Ray 等后端差异在哪里

## 第五阶段：按兴趣深入专题

如果你的目标不同，后续可以分支阅读：

- 性能优化：`attention/`、`v1/core/kv_cache_*`、`v1/worker/gpu/*`
- 分布式：`distributed/`、`v1/executor/ray_*`
- 多模态：`multimodal/`、`v1/worker/gpu/mm/*`
- 工具调用与结构化输出：`tool_parsers/`、`reasoning/`、`v1/structured_output`
- 模型接入：`model_executor/models/`、`transformers_utils/`

## 一个阅读原则

先读“状态如何流动”，再读“算子如何优化”。

原因很简单：不知道请求状态、调度语义和缓存生命周期时，直接钻 kernel 或模型实现，很容易只看到局部技巧，看不到系统设计。
