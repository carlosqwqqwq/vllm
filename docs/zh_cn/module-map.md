# vLLM 模块地图

`vllm/` 顶层目录很多，第一次看容易被淹没。这一篇的目标不是逐文件解释，而是先建立“功能分区”。

## 1. 对外入口与接口层

| 目录 | 主要职责 |
| --- | --- |
| `entrypoints/` | 对外暴露的离线接口、OpenAI 兼容服务、CLI、Anthropic/MCP/SageMaker 等入口 |
| `engine/` | 兼容层入口；当前主要把公开 `LLMEngine` / `AsyncLLMEngine` 指向 `v1` |
| `inputs/` | prompt、token、embeds、多模态输入等统一输入表示 |
| `outputs.py`、`sampling_params.py`、`pooling_params.py` | 对外 API 常见的数据结构与参数对象 |

## 2. 核心运行时层

| 目录 | 主要职责 |
| --- | --- |
| `v1/engine/` | 请求处理、引擎协调、核心循环、异步客户端、输出整理 |
| `v1/core/` | 调度器、KV Cache 管理、encoder cache、调度输出结构 |
| `v1/executor/` | 单进程、多进程、Ray、外部 launcher 等执行后端 |
| `v1/worker/` | 设备侧 worker、model runner、采样、多模态执行、spec decode 等 |

如果你是为了理解“vLLM 为什么快”，这四个目录是主战场。

## 3. 模型与算子层

| 目录 | 主要职责 |
| --- | --- |
| `model_executor/` | 模型注册、模型执行相关实现、层与模型适配 |
| `attention/` | Attention 相关逻辑与后端适配 |
| `triton_utils/` | Triton 相关辅助 |
| `csrc/` | C++/CUDA 等原生扩展源码 |
| `vllm_flash_attn/` | FlashAttention 相关包装 |

这一层更接近“算子和模型怎么真正跑起来”。

## 4. 并行、分布式与缓存扩展

| 目录 | 主要职责 |
| --- | --- |
| `distributed/` | 张量并行、流水并行、数据并行、KV 传输、专家通信等分布式能力 |
| `lora/` | LoRA 请求与加载相关 |
| `plugins/` | 插件系统 |
| `ray/` | Ray 集成辅助 |

这部分体现了 vLLM 不只是单机推理库，而是一个可扩展的服务运行时。

## 5. 模态与任务扩展

| 目录 | 主要职责 |
| --- | --- |
| `multimodal/` | 多模态输入、预算、缓存和处理器适配 |
| `tokenizers/` | tokenizer 适配 |
| `renderers/` | 渲染与输入预处理桥梁 |
| `tool_parsers/` | tool calling 相关解析 |
| `reasoning/` | reasoning parser 相关逻辑 |
| `pooling/`、`tasks.py` | embedding / pooling / 分类 / 打分等任务语义 |

这部分说明 vLLM 已经不再只是“文本生成引擎”。

## 6. 配置与平台层

| 目录 | 主要职责 |
| --- | --- |
| `config/` | 模型、缓存、并行、观测、编译等配置对象 |
| `platforms/` | 不同硬件平台能力判断与差异封装 |
| `utils/` | 各类通用工具 |
| `tracing/`、`usage/`、`logging_utils/` | 观测、上报、日志能力 |

如果你想追“某个参数最终在哪里起作用”，通常会从 `config/` 开始一路追到 `v1` 或 `worker`。

## 7. 建议怎么用这张地图

第一次读仓库时，不建议顺着目录树硬读。更有效的方法是带着问题走：

- 想看“一个请求怎么执行”：读 `entrypoints` + `v1/engine` + `v1/core`
- 想看“为什么吞吐高”：读 `v1/core` + `v1/worker` + `attention`
- 想看“怎么支持多卡多机”：读 `executor` + `distributed`
- 想看“怎么接入新模型”：读 `model_executor` + `transformers_utils` + 官方贡献文档

把目录地图和问题绑定起来，阅读效率会高很多。
