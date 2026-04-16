# FalseNegativeKV 实现目录

更新日期：2026-04-15

这个目录统一收纳 `FalseNegativeKV / TwinPathServe` 当前主线的实现规格、原型设计和推进计划。

## 当前内容

- [false_negative_instrumentation_spec.md](false_negative_instrumentation_spec.md)
  - P0 measurement 的最小观测接口、字段、JSONL schema 与代码入口
- [reuse_contract_spec.md](reuse_contract_spec.md)
  - 当前主线的正式方法层规格，定义 `reuse contract`、`bridgeability`、`bridge operator` 与 `TwinPathServe` 的统一抽象边界
- [realization_policy_spec.md](realization_policy_spec.md)
  - 当前主线的 realization-aware 决策规格，固定 `bridge cost`、`realized gain`、P1 选择规则和 fallback 语义
- [twin_path_serve_minimal_architecture.md](twin_path_serve_minimal_architecture.md)
  - `TwinPathServe` 的双池系统边界、route key、bridge operator 和 fallback
- [prompt_logprobs_selective_replay_design.md](prompt_logprobs_selective_replay_design.md)
  - 第一个 bridge 的 prototype 设计，固定 `sidecar + boundary replay + full-prefill fallback`
- [prompt_logprobs_selective_replay_status.md](prompt_logprobs_selective_replay_status.md)
  - 当前主方法的实现落地状态、冒烟对照结果和剩余工程边界
- [false_negative_kv_hardening_plan.md](false_negative_kv_hardening_plan.md)
  - 从“好问题雏形”推进到 OSDI/CCF-A thesis 的提标差距分析
- [code_implementation_plan.md](code_implementation_plan.md)
  - 代码施工顺序、ablation 开关组和统一实验设置代码的固定要求
- [full_method_readiness_audit_20260415.md](full_method_readiness_audit_20260415.md)
  - 复核当前 thesis、历史 idea 吸收关系、方法完整度与“先补方法再跑公开 benchmark”的硬性 gate
- [twin_path_backend_route_status.md](twin_path_backend_route_status.md)
  - 第二条主机制的当前实现状态、双池 smoke validation 结果与剩余缺口

## 维护约定

- 这里只放可执行的规格、设计和推进计划
- 不在这里堆 paper narrative 文案
- 不在这里堆新 idea 比较文档
