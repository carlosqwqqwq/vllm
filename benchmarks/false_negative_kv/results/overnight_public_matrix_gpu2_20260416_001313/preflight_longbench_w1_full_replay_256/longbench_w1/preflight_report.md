# LongBench W1 实验 Preflight 报告

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

## 数据集与样本选择
- LongBench 数据访问: OK
- `data.zip`: `/data/infinigen/accuracy/LongBench/LongBench/longbench_data/data.zip`
- 可用 task 数: 34
- 选中 task: ['qmsum', 'narrativeqa', 'qasper', 'multifieldqa_zh', 'multifieldqa_en']
- 样本字段: ['_id', 'all_classes', 'answers', 'context', 'dataset', 'input', 'language', 'length']
- tokenizer + selection: OK
- 选中请求数: 64
- 选中 family 数: 32
- 选中 task 分布: {'qmsum': 64}
- 模型上下文上限: 32768
- prompt token budget: 32767
- 首个 family: dataset=longbench|task=qmsum|context_sha1=6caa72a68d09f2b9c7a2d59a860f519ae168e2a5|wrapper=longbench_public_v1

## GPU
- GPU 查询: OK
- 推荐 GPU: 2
- GPU 0: free=8033MiB / total=81920MiB / util=0% / NVIDIA A100 80GB PCIe
- GPU 1: free=8033MiB / total=81920MiB / util=0% / NVIDIA A100 80GB PCIe
- GPU 2: free=81156MiB / total=81920MiB / util=0% / NVIDIA A100 80GB PCIe
- GPU 3: free=5235MiB / total=81920MiB / util=0% / NVIDIA A100 80GB PCIe

## Blockers
- 无

## Warnings
- 无
