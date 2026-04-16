# Rebuttal 参考：统一性、安全性与“为什么是我们”

更新日期：2026-04-16

## 1. 这份文档的作用

这份文档只服务一个目标：

- 为三类高风险审稿质疑提供可复用、口径统一的回答参考

三类问题分别是：

1. 你们的贡献是否过小，本质上只是改了几个节点，没有提出统一方法？
2. 你们是否证明这些复用是真实可信且安全的？
3. 别人为什么没有做、没有想到，为什么你们做了？

这份文档不负责：

- 替代正文
- 发散新的 thesis
- 夸大当前证据边界

它的固定用途是：

1. 作为 rebuttal 起草参考
2. 作为正文 hardening 时的口径约束
3. 作为内部统一回答模板，避免回到“两个 patch”叙事

## 2. 总体回答口径

这三类问题的共同风险在于，审稿人会把工作理解为“两个局部修补 + 一点工程实现”。因此，回答必须始终收束到下面这条主线：

> 本文的贡献不是分别修通两个 feature gap，而是识别并形式化了一类此前未被显式命名的运行时失效，即 reference-compatible exact reuse 未能在当前 request 中 materialize 为 `physical hit`。  
> 围绕这一问题类，本文提出统一的 measurement surface、`reuse contract`、`SafeReuseContract`、`realization policy` 与 `bridge operator` 组织方式；`prompt_logprobs selective replay` 与 twin-pool route 只是这一统一控制面的两个实例。  
> 本文的安全主张是条件化 `soundness`，而不是无条件“所有 logical hit 都可恢复”。

如果这条主线没有守住，rebuttal 很容易退化为：

- “我们其实也做了统一”
- “我们感觉应该是安全的”
- “别人没想到，我们想到了”

这三种答法都不够强。

## 3. 问题一：你们的贡献是否过小，没有统一方法？

### 3.1 短回答

不是。本文的核心贡献不在于分别修通 `prompt_logprobs` 和 backend route 两个局部节点，而在于识别并形式化了一个新的运行时问题类，并提出了一套统一的契约感知恢复框架；两个机制只是这一框架的实例化证明。

### 3.2 展开回答

对这一问题的回答，必须首先拒绝“两个 patch”的叙事方式。正确的表述应当是：

1. 本文首先提出了一个此前缺少明确命名的问题对象，即 `false-negative exact hit`。它描述的是：在参考执行条件下已经成立的 exact 或 logical-exact reuse，并没有在当前请求中兑现为 `physical hit`。
2. 基于这一问题对象，本文建立了统一的问题表征，包括 `logical hit`、`physical hit`、`false-negative hit` 与 `bridgeable miss`。
3. 在此基础上，本文引入 `reuse contract` 与 `SafeReuseContract`，把“状态存在”“当前可消费”“当前可安全恢复”这三层语义区分开来。
4. 最后，本文把恢复组织为统一的控制链：`reuse oracle -> contract gate -> realization policy -> bridge operator -> route/admission`。

因此，统一性并不体现在：

- 两个 bridge 是否共享同一段实现代码
- 两个机制是否长得像同一个 patch

统一性真正体现在：

1. 它们共享同一套 request-level 问题表征
2. 它们共享同一套 reason taxonomy 与 trace language
3. 它们共享同一套 `SafeReuseContract` 安全边界
4. 它们共享同一条控制面决策链

### 3.3 一句最重要的话

> 本文的统一性不是“两个修复长得像”，而是“两个机制被同一套问题定义、同一套契约语义和同一条运行时控制链所组织”。

### 3.4 不应使用的弱回答

不要写成：

- “我们虽然只修了两个 case，但也算是一种统一方法”
- “统一主要体现在我们都在做 prefix reuse”
- “这两个点很重要，所以合起来就是系统工作”

这些写法会主动把工作降级为 feature aggregation。

## 4. 问题二：你们是否证明了复用真实可信且安全？

### 4.1 短回答

本文证明的不是“已有 state 一律应当复用”，而是条件化 `soundness`：只有当 `SafeReuseContract` 成立时，系统才允许恢复；否则必须 fallback 到 baseline 支持路径。

### 4.2 展开回答

这一问题的核心，不是“有没有看到命中恢复”，而是“恢复后的用户可见语义是否可信”。因此，回答必须把安全性拆成三个层次：

1. `identity correctness`
   状态是否来自同一 exact 或 logical-exact prefix。
2. `consumer legality`
   当前请求在自己的 API 语义、backend class 与 carrier 约束下，是否可以合法消费该状态。
3. `observational safety`
   bridge 执行后，用户可见输出是否与 baseline 支持路径等价，或至少具有明确且受控的误差边界。

因此，本文的安全主张必须被固定为：

- 不是 `completeness`
- 而是 `soundness`

也就是说：

1. 我们允许保守地错过一个本可安全恢复的 case
2. 我们不允许把一个不安全的 case 错误地恢复出来

### 4.3 对两类 bridge 的具体回答

对 `prompt_logprobs selective replay` 而言，安全性不是“允许读 prefix cache”就自动成立，而是一个严格的输出等价问题。其 baseline 是 `full-prefill + full prompt observation`。因此，这一机制的正确性必须落在：

1. prompt-logprob 条目数是否一致
2. 各位置 top-k token、rank 与 logprob 是否一致
3. 输出时机是否一致
4. 任一条件不满足时是否正确 fallback

对 twin-pool route 而言，安全主张必须更保守。本文并不证明跨不兼容 backend 直接共享 state 是安全的。第二类 bridge 的本质不是“跨池直接共用一份 KV”，而是“避免不安全消费，把 family 重新送回兼容执行面”。因此，这里的安全性来自：

1. 不跨越不兼容 backend class 直接消费 state
2. 仅在兼容执行面中兑现已有 reuse
3. 不能验证时立即 fallback

### 4.4 一句最重要的话

> 本文证明的是 `safe reuse under verified contracts`，而不是“凡是 logical hit 都恢复”；我们主张的是可验证的 `soundness`，不是无条件的 `completeness`。

### 4.5 不应使用的弱回答

不要写成：

- “我们恢复后看到了 hit，所以应该是安全的”
- “系统本来就有 cache，所以继续复用问题不大”
- “我们觉得这两个场景风险不高”

这些表述都缺乏可验证的安全边界。

## 5. 问题三：别人为什么没有做，没有想到，为什么你们做了？

### 5.1 短回答

这项工作之所以此前没有被系统化研究，不是因为问题不存在，也不是因为别人简单“没想到”，而是因为直到最近，这个问题才同时具备被严格测量、统一归因和保守恢复的条件。

### 5.2 展开回答

这一问题的回答不能写成“我们比别人更聪明”。更稳妥的回答有三层。

第一，过去缺少可测量性。现有运行时大多只记录 `physical hit`，看不到 `logical hit` 与 `physical hit` 之间的差值，因此即便问题存在，也只能表现为“某些路径没命中”，而无法被识别为一个统一的问题类。

第二，这类失效长期被 feature-local explanation 掩盖。它们通常表现为：

- capability boundary
- 保守 skip
- 某条 API 关闭 cache
- 某类 backend 路径不消费 prefix state

这些现象不会像 crash 那样显式报错，因此容易被当作“局部兼容性限制”，而不是独立的 systems question。

第三，现代 mixed-API、mixed-backend serving 才使这一问题系统化暴露。在早期单 API、单 backend 的 serving 场景中，prefix identity 与 consumer compatibility 往往近似重合，因此“前缀相同”几乎就意味着“可以直接复用”。随着 typed API、异构 backend、carrier 差异和 observation requirement 的引入，identity 与 consumability 的分离才成为系统性矛盾。

在这一背景下，本文之所以能够推进这项工作，不应表述为“我们想到了别人没想到的点子”，而应表述为：

1. 现代 serving 负载首次使该问题成为显著且稳定的系统现象
2. 我们在 request-level、family-aware 粒度上构造了可复核的 counterfactual measurement
3. 在统一测量与统一契约抽象的基础上，才第一次把分散现象上升为一个问题类

### 5.3 一句最重要的话

> 不是别人简单没有想到，而是直到 mixed-API、mixed-backend serving 成为常态，这个问题才首次同时具备了被严格测量、统一归因和保守恢复的条件。

### 5.4 不应使用的弱回答

不要写成：

- “别人没想到安全 bridge 这个方向”
- “这是一个被忽视的 brilliant idea”
- “我们比现有工作更敏锐”

这类表述在审稿中通常会适得其反。

## 6. 一段可直接复用的短版 rebuttal

下面这段话可以直接作为短版回答模板：

> 本文的贡献不是分别修通两个 feature gap，而是识别并形式化了前缀复用中的假阴性这一运行时问题类，并提出一套契约感知的统一恢复框架。统一性不体现在两个 bridge 共享同一段实现代码，而体现在它们共享同一套问题表征、同一套契约语义、同一套 reason taxonomy 以及同一条 `oracle -> contract gate -> realization policy -> bridge/route -> fallback` 控制链。与此同时，本文的安全主张是条件化 `soundness`：仅在 `SafeReuseContract` 被验证成立时恢复，否则回退到基线路径。该问题过去没有被系统性研究，并不是因为它不存在，而是因为传统运行时统计只能看到 `physical hit`，看不到 `logical hit -> physical hit` 的断裂；而 mixed-API、mixed-backend serving 直到最近才使这一断裂成为显著且可测量的系统现象。

## 7. 更强版本的长回答

如果审稿人持续追问，可以使用下面这段更完整的版本：

> 我们不同意“本文只是几个局部 patch”的判断。`prompt_logprobs selective replay` 与 twin-pool route 的确是两个机制实例，但它们并不是彼此独立的 feature fix。本文首先识别并形式化了一个此前缺少明确命名的运行时问题：在参考执行条件下已经成立的 exact 或 logical-exact reuse，没有在当前请求中兑现为 `physical hit`。围绕这一问题，我们提出了统一的 measurement surface、`logical hit / physical hit / false-negative hit / bridgeable miss` 表征、`reuse contract` 与 `SafeReuseContract` 安全边界，以及 `reuse oracle -> contract gate -> realization policy -> bridge operator -> route/admission` 这条统一控制链。两个机制之所以被放在同一篇论文中，正是因为它们共享这一抽象与控制面，而不是因为它们恰好都与 prefix reuse 有关。与此同时，我们并不声称“已有 state 一律可安全复用”。本文的安全主张是条件化 `soundness`：只有当身份、执行兼容性、载体兼容性、观测完整性与可验证回退条件同时成立时，恢复才被允许；否则系统必须回退到 baseline 支持路径。最后，这个问题并非过去不存在，而是长期被 capability boundary 和 feature-local explanation 所掩盖。直到 mixed-API、mixed-backend serving 成为常态，identity 与 consumability 的分离才成为系统性矛盾；也正因为我们在 request-level、family-aware 粒度上构造了可复核测量，才第一次把这些分散现象统一提升为一个可讨论的 systems problem。

## 8. 与主文稿的对应位置

如果需要把 rebuttal 与正文对齐，当前最相关的位置是：

1. 统一性问题
   [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)
   [05_TwinPathServe系统设计.md](../manuscript_cn/05_TwinPathServe系统设计.md)
   [08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md)
2. 安全性问题
   [04_契约感知复用抽象.md](../manuscript_cn/04_契约感知复用抽象.md)
   [05_TwinPathServe系统设计.md](../manuscript_cn/05_TwinPathServe系统设计.md)
   [06_实验评估.md](../manuscript_cn/06_实验评估.md)
   [safe_reuse_soundness_note.md](safe_reuse_soundness_note.md)
3. 为什么是现在、为什么是我们
   [01_引言.md](../manuscript_cn/01_引言.md)
   [03_FalseNegativeExactHit的测量.md](../manuscript_cn/03_FalseNegativeExactHit的测量.md)
   [07_相关工作.md](../manuscript_cn/07_相关工作.md)
   [08_讨论与结论.md](../manuscript_cn/08_讨论与结论.md)

## 9. 这份文档要守住的边界

从现在开始，围绕这三类问题，口径必须固定为：

1. 统一性来自统一抽象、统一控制链和统一安全边界，不来自“两个 patch 恰好放在一起”
2. 安全性主张是 `soundness`，不是 `completeness`
3. “为什么是我们”来自问题首次变得可测量、可统一归因，而不是“别人没想到”

如果后续文字越过这三条边界，文风就会重新滑回答辩腔或工程报告腔。
