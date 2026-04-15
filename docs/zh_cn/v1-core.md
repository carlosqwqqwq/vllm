# vLLM V1 核心架构

`v1` 是当前 vLLM 的主实现。它并不是简单“换个目录重写一遍”，而是把核心运行时重新分层，让调度、执行、缓存和输出的职责更清晰。

## 1. 五层结构

我更推荐把 `v1` 看成下面五层：

1. 请求适配层：`InputProcessor`、`OutputProcessor`
2. 引擎协调层：`LLMEngine`、`AsyncLLM`、`EngineCoreClient`
3. 引擎内核层：`EngineCore`
4. 调度状态层：`Scheduler`、KV Cache Manager、Structured Output Manager
5. 执行落地层：`Executor`、`Worker`、`ModelRunner`

## 2. 请求适配层

### InputProcessor

`vllm.v1.engine.input_processor.InputProcessor` 负责把“用户世界”的输入翻译成“引擎世界”的请求。

它统一处理：

- 文本 prompt、token ids、embeds
- 采样任务与池化任务
- LoRA 参数
- 多模态特征
- tracing、priority、data parallel rank 等附加元数据

可以把它看成“入口语义归一化器”。

### OutputProcessor

`vllm.v1.engine.output_processor.OutputProcessor` 则做反方向的事情：把内部执行结果整理成外部 API 需要的输出对象。

它承担的工作包括：

- 维护每个请求的 `RequestState`
- 增量 detokenize
- logprobs 整理
- 流式输出聚合
- `n > 1` 并行采样结果的父子请求映射

这层让调度器不必关心“HTTP 是怎么流式返回的”或“用户最终想看到什么格式”。

## 3. 引擎协调层

### LLMEngine / AsyncLLM

这一层负责请求生命周期管理，但不直接做底层调度细节。

它们主要做四件事：

1. 创建配置与核心组件。
2. 接收新请求并调用 `InputProcessor`。
3. 驱动 `EngineCoreClient` 与核心循环交互。
4. 调用 `OutputProcessor` 生成可返回结果。

同步 `LLMEngine` 更适合离线推理；异步 `AsyncLLM` 更适合 API 服务。

### EngineCoreClient

`EngineCoreClient` 是一个很关键的抽象。它把“引擎核心到底在当前进程，还是后台进程，还是多数据并行引擎”隐藏起来。

它至少覆盖三种运行方式：

- `InprocClient`：单进程内直接调用
- `SyncMPClient`：同步多进程
- `AsyncMPClient`：异步多进程

因此，上层代码大多不必直接关心 ZMQ、后台进程和跨进程通信细节。

## 4. 引擎内核层

`vllm.v1.engine.core.EngineCore` 可以看成真正的“运行时内核”。

它在初始化阶段完成：

- 选择并创建 `Executor`
- 分析可用显存，初始化 KV Cache
- 创建 `StructuredOutputManager`
- 创建 `Scheduler`

它在运行阶段的关键循环则非常直接：

1. `schedule()`
2. `execute_model()`
3. `sample_tokens()`
4. `update_from_output()`
5. `post_step()`

这也是整个系统最值得反复阅读的部分，因为这里把“调度决策”和“实际执行”粘合在了一起。

## 5. 调度状态层

### Scheduler

`vllm.v1.core.sched.scheduler.Scheduler` 是 v1 最核心的模块之一。

它最重要的设计思想是：不再把系统硬拆成“prefill 阶段”和“decode 阶段”，而是统一用“每个请求当前还差多少 token 需要计算”的视角来做调度。

这带来几个直接收益：

- chunked prefill 更自然
- prefix caching 更容易统一进主流程
- speculative decoding 不需要额外割裂出另一套调度框架
- 调度逻辑能围绕 token budget 统一表达

### KV Cache 相关

`v1/core` 下围绕 KV Cache 有一组密集模块：

- `kv_cache_manager.py`
- `block_pool.py`
- `single_type_kv_cache_manager.py`
- `kv_cache_coordinator.py`
- `encoder_cache_manager.py`

这说明在 vLLM 里，KV Cache 不是一个附属优化点，而是基础设施。很多性能能力最终都要落回这里。

### Structured Output / Multimodal

V1 还把结构化输出与多模态缓存都纳入核心运行时，而不是简单作为外围插件拼接：

- structured output 通过 `StructuredOutputManager` 参与调度前后的状态变化
- 多模态预算和 encoder cache 会直接影响请求调度与缓存占用

因此，vLLM 的“统一运行时”不是口号，而是代码分层上的现实。

## 6. 执行落地层

### Executor

`Executor` 负责决定如何把一个调度结果真正发到工作进程或设备上执行。

它根据配置选择不同后端：

- `UniProcExecutor`
- `MultiprocExecutor`
- `RayDistributedExecutor` / `RayExecutorV2`
- `ExecutorWithExternalLauncher`

这里的重点不是模型逻辑，而是执行编排、RPC、并行后端差异与 worker 生命周期。

### Worker

`WorkerBase` 是设备侧执行抽象，负责定义这些最底层能力：

- 初始化设备
- 加载模型
- 执行前向
- 采样
- 管理 LoRA
- 提供 KV Cache 规格

再往下的 GPU/CPU/XPU worker 和 model runner，才会真正接触具体硬件与内核实现。

## 7. 一个实用阅读建议

读 V1 时，最容易陷入的误区是“一上来钻某个 kernel 或某个模型实现”。这通常收益不高。

更高效的顺序是：

1. 先读 `InputProcessor` / `OutputProcessor`
2. 再读 `EngineCore`
3. 然后读 `Scheduler`
4. 最后再看 `Executor` 与 `Worker`

因为只有先弄清请求生命周期和状态流动，后面的设备实现细节才有上下文。
