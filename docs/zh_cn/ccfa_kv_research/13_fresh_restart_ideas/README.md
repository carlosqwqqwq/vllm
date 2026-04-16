# 第八轮在放弃旧主线后的全新候选

更新日期：2026-04-15

这一轮的前提非常明确：

- 放弃 `ExactPrefixKV` 这条组合主线；
- 不再继续围绕 `exact-prefix contract / typed prefix / residual selector` 做修补；
- 重新寻找 **能在 `vLLM` 中实现、与 2025-2026 最新工作重合更低、且论文新意更强** 的新方向。

## 为什么要重开

前一轮最大的结论不是“旧主线完全错误”，而是：

- 它有工程价值；
- 但创新性已经不够稳；
- 很容易被审稿人降格成 patch collection。

与此同时，最近一批强工作已经把很多传统空间压得很紧：

- `LMCache`
- `BanaServe`
- `HCache`
- `SIMPLE`
- `XGrammar 2`
- `Beyond Speedup`
- `Apt-Serve`

它们共同说明：

- 继续沿着 read path / exact-prefix / routing / disaggregation / decision plane 的常规套路推进，越来越容易撞车。

因此，这一轮的筛选标准只有三个：

1. **问题必须和旧主线明显不同**
2. **与最新强工作不能有太多正面重合**
3. **第一版必须能在 `vLLM` 中做出轻量原型**

## 这一轮保留的三个方向

最终保留三个方向：

1. [WritePathKV](./writepathkv/README.md)
   - 把 KV 的“发布/导出/存储”写路径本身变成一等优化对象。
2. [ObservationSidecarKV](./observationsidecarkv/README.md)
   - 为 mixed API 请求缓存极小观测 sidecar，而不只缓存 KV。
3. [DecisionSidecarKV](./decisionsidecarkv/README.md)
   - 在 prefix 共享场景下复用结构化输出 / 采样侧的 decision state。

## 当前排序

当前排序是：

1. `WritePathKV`
2. `ObservationSidecarKV`
3. `DecisionSidecarKV`

排序理由如下：

- `WritePathKV`
  - 与最新系统工作的边界最清楚；
  - 最像新的系统问题；
  - 在 `vLLM` 里也最自然。
- `ObservationSidecarKV`
  - capability gap 非常真实；
  - 落地性很强；
  - 但要小心被看成 API feature patch。
- `DecisionSidecarKV`
  - 创新性很高；
  - 但和 `SIMPLE / XGrammar 2` 一类 decision-plane 工作更近；
  - 风险也最高。

## 与前几轮的关系

这三个方向不是凭空冒出来的，而是对前几轮“已不够新”的反向修正。

更具体地说：

- `WritePathKV`
  - 来自我们对 local cache / offload / connector / disaggregation 路径的重新观察；
  - 它主动避开 read-path reuse 这个拥挤问题。
- `ObservationSidecarKV`
  - 延续了 `ObservationKV` 的真实缺口意识；
  - 但不再只讲“支持更多 API”，而是把复用对象扩展成 `KV + minimal observation sidecar`。
- `DecisionSidecarKV`
  - 延续了“复用不应只发生在 data plane”的判断；
  - 但不再走通用 exact-prefix contract，而是转向更具体的 decision-state reuse。

## 建议阅读顺序

1. [comparison.md](./comparison.md)
2. [WritePathKV](./writepathkv/README.md)
3. [ObservationSidecarKV](./observationsidecarkv/README.md)
4. [DecisionSidecarKV](./decisionsidecarkv/README.md)

## 当前总判断

截至 `2026-04-15`，我当前最推荐的新主线是：

- **`WritePathKV`**

因为它最符合下面这个目标：

> 不再和最近论文在“怎么读 cache / 怎么命中 cache / 怎么路由 cache”上正面重合，
> 而是切向一个更少人系统化研究、但在 `vLLM` 新栈里越来越重要的问题：
> **新生成状态是否值得发布、何时发布、发布到哪一层，以及写路径本身的放大效应。**
