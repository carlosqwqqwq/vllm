# BFCL W2 实验 Preflight 报告

## 总结
- 就绪状态: `ready`
- Python: `/opt/conda/envs/vllm_longbench/bin/python`
- 当前工作目录: `/data/projects/KVresearch/vllm`
- 运行视角: `container`
- 模型路径: `/data/models_copy/llama/llama-2-7b-32k`
- 模型路径存在: `True`

## 依赖
- `torch`: OK (`2.10.0+cu128`)
- `transformers`: OK (`4.57.6`)
- `vllm`: OK (`0.1.dev15498+g8848fd7bf.d20260403`)

## 本地数据快照
- BFCL 本地快照: OK
- 选中子集: `['multi_turn_base', 'multi_turn_composite', 'multi_turn_long_context', 'multi_turn_miss_func', 'multi_turn_miss_param']`
- 子集文件: `{'multi_turn_base': '/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/public_bench/data/bfcl/BFCL_v3_multi_turn_base.json', 'multi_turn_composite': '/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/public_bench/data/bfcl/BFCL_v3_multi_turn_composite.json', 'multi_turn_long_context': '/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/public_bench/data/bfcl/BFCL_v3_multi_turn_long_context.json', 'multi_turn_miss_func': '/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/public_bench/data/bfcl/BFCL_v3_multi_turn_miss_func.json', 'multi_turn_miss_param': '/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/public_bench/data/bfcl/BFCL_v3_multi_turn_miss_param.json'}`

## 选样结果
- 选样状态: OK
- 原始行数: 1000
- 原始请求数: 4528
- 超出上下文预算请求数: 0
- 过阈值请求数: 4528
- 选中请求数: 64
- 选中 family 数: 32
- 模型上下文上限: 32768
- prompt token 统计: min=5331, mean=8681.84, max=12243
- 选中 subset 分布: {'multi_turn_base': 61, 'multi_turn_composite': 3}
- 选中 turn 分布: {'0': 16, '1': 14, '2': 14, '3': 12, '4': 8}
- 选中 tool class 分布: {'GorillaFileSystem': 48, 'MathAPI': 10, 'MessageAPI': 18, 'TicketAPI': 10, 'TwitterAPI': 14, 'VehicleControlAPI': 16}
- 首个 family: `dataset=bfcl|tool_schema_hash=b47e11121a53b883f9d540ee55c6202db90c1d04|wrapper=bfcl_public_w2_v1|tool_mode=multi_turn_v3|turn_index=0`
- 首个 sample: `multi_turn_base_0:turn0`

## GPU
- GPU 状态查询: OK
- 推荐 GPU: `2`
- GPU0: NVIDIA A100 80GB PCIe, free=8033 MiB / total=81920 MiB, util=0%
- GPU1: NVIDIA A100 80GB PCIe, free=8033 MiB / total=81920 MiB, util=0%
- GPU2: NVIDIA A100 80GB PCIe, free=81156 MiB / total=81920 MiB, util=0%
- GPU3: NVIDIA A100 80GB PCIe, free=69633 MiB / total=81920 MiB, util=0%
