# MT-Bench W1 实验 Preflight 报告

## 总结
- 就绪状态: `blocked`
- Python: `/opt/conda/envs/vllm_longbench/bin/python`
- 当前工作目录: `/data/projects/KVresearch/vllm`
- 运行视角: `container`
- 模型路径: `/data/models_copy/llama/llama-2-7b-chat`
- 模型路径存在: `True`

## 依赖
- `datasets`: OK (`4.2.0`)
- `torch`: OK (`2.10.0+cu128`)
- `transformers`: OK (`4.57.6`)
- `vllm`: OK (`0.1.dev15498+g8848fd7bf.d20260403`)

## 数据集与样本选择
- MT-Bench 数据访问: OK (`local_snapshot_primary`)
- 样本字段: ['category', 'question_id', 'reference', 'turns']
- tokenizer + selection: FAILED
- 选中请求数: 0
- 选中 family 数: 0
- 类别分布: {}
- 首个 family: None
- tokenizer/selection 错误: selection produced zero requests

## GPU
- GPU 查询: OK
- 推荐 GPU: 2
- GPU 0: free=8033MiB / total=81920MiB / util=100% / NVIDIA A100 80GB PCIe
- GPU 1: free=8033MiB / total=81920MiB / util=100% / NVIDIA A100 80GB PCIe
- GPU 2: free=81156MiB / total=81920MiB / util=0% / NVIDIA A100 80GB PCIe
- GPU 3: free=5235MiB / total=81920MiB / util=0% / NVIDIA A100 80GB PCIe

## Blockers
- tokenizer 或样本选择失败: selection produced zero requests

## Warnings
- 无
