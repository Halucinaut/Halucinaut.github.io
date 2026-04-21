---
title: "SILK：面向 Speculative Decoding 的动态 Top-K 重排序方法"
date: 2026-04-21
description: "关于 speculative decoding 中 acceptance-aware reranking 的研究札记"
tags: ["Speculative Decoding", "Inference Optimization", "LLM", "Technical Report"]
cover:
    image: ""
    alt: "SILK"
---

# SILK：面向 Speculative Decoding 的动态 Top-K 重排序方法

作者：Sunrx  
日期：2026 年 4 月  
类型：技术报告 / 研究札记

定位：一篇关于 speculative decoding 中 acceptance-aware reranking 的研究札记。

## 摘要

Speculative Decoding 通过"小模型起草、大模型验证"的方式降低大语言模型推理延迟，但真实收益高度依赖 acceptance rate。本文研究 `SILK`，一种仅作用于 draft model `Top-K` 候选集合的轻量级、上下文相关重排序模块。其核心假设是：许多 draft-target mismatch 不是召回失败，而是排序失败，即 target token 已被 draft model 召回到 `Top-K` 内，但未排在第一位。基于这一假设，`SILK` 在 frozen drafter 与最终 token 决策之间插入一个小型控制器，仅对 `Top-K` logits 施加动态 bias，而不做全词表计算。

在当前可见日志中，`SILK` 展现出"方向成立但收益脆弱"的特征。保守配置下，acceptance rate 从 `0.2165` 提升到 `0.2225`，另一条重复日志达到 `0.2274`；当样本数扩展到 `2048` 时，也仍能观察到 `0.2354 -> 0.2397` 的改善。然而，激进配置会把 acceptance rate 降到 `0.1721`。这些结果表明，后处理式 `Top-K` 重排序确实识别到了真实的排序机会，但也暴露出 speculative decoding 的关键难点：局部 token 偏好对齐，并不自动转化为 block 级 accepted prefix length 的提升。本文的主要贡献因此不是提出一个已经成熟的加速模块，而是更精确地刻画了 acceptance-aware optimization 的必要性。

**证据边界**

本文所有结论都基于当前仓库中可直接读取的训练脚本与评测日志。文中不声称这些结果已经构成充分的多次复现实验，也不把当前结论外推到更大模型对或生产级 speculative decoding 场景。

## 1. 引言

大语言模型推理的主要瓶颈仍然是严格串行的自回归生成。Speculative Decoding 的重要性在于，它保留 target model 输出分布的同时，把候选 proposal 的生成工作转交给更快的 draft model，再由 target model 对整个 token block 一次性验证 [1,2]。理论上，收益来自一次前向能接受多个 token；实践上，收益最终由 acceptance dynamics 决定。

已有研究主要沿三条路线推进 speculative decoding。第一类工作改进 draft model 本身，例如更强的 drafter、self-speculative 结构或 feature-level drafting [4,5,6,7]。第二类工作改变 proposal 结构，例如 tree verification、adaptive draft tree 或 lookahead decoding [8,9,10,11]。第三类工作聚焦 adaptive draft length、硬件感知控制或系统级优化 [12,13,14]。这些路线大多有效，但它们也留下了一个较少被单独讨论的问题：如果 drafter 已经频繁将正确 token 召回进一个较小候选池，仅仅是排序不对，那么是否有必要为此重训整个 drafter？

本文的出发点正是这一更窄的问题。我们研究一个轻量 post-hoc reranker `SILK`，它不替换 drafter，不改动 transformer 主体结构，也不计算全词表分布，而是在 frozen hidden states 与最终 token 选择之间插入一个小型控制器，只对当前 `Top-K` 候选进行上下文相关重排序。这个设定试图回答三个问题：

1. 仅作用于 `Top-K` 的轻量 reranker 能否在工程上嵌入 speculative decoding，而不破坏其延迟优势？
2. 上下文相关的动态重排序能否在 static distillation 之外进一步改善 draft-target 对齐？
3. 如果收益不稳定，这种不稳定究竟揭示了 speculative decoding 的什么真实优化目标？

本文认为，第三个问题才是该项目最重要的产出。因为只要 token-level alignment 与 accepted prefix length 之间存在系统性错位，那么单纯优化 local ranking 的方法即使偶尔有效，也难以稳定兑现为部署收益。

## 2. 相关工作

### 2.1 Speculative Decoding 与并行验证

Leviathan 等人 [1] 与 Chen 等人 [2] 奠定了现代 speculative decoding 的基本范式：由较快的 draft model 先生成多个 token，再由 target model 一次性验证，从而在不改变目标分布的前提下减少串行步数。随后，Xia 等人 [3] 对该方向进行了系统综述，将 drafter 选择、verification strategy 和工程部署问题梳理为一套相对清晰的研究框架。

### 2.2 单模型与结构化 speculative decoding

在两模型 speculative decoding 之外，Medusa [5] 通过多头并行预测实现单模型 speculative decoding，Draft & Verify [6] 则利用层跳过实现 self-speculative 机制。EAGLE [4] 将 drafting 提升到 feature level，并强调 uncertainty 建模的重要性；ReDrafter [7] 进一步用 recurrent drafter 和 tree attention 组织更高效的 candidate proposal。这些工作共同说明，speculative decoding 的收益不仅来自"更强 drafter"，也来自 proposal structure 的设计。

### 2.3 Adaptive draft structure 与 acceptance-aware 控制

SpecInfer [8]、Lookahead Decoding [9]、OPT-Tree [10]、DISCO [11]、Sequoia [12] 与 AdaEAGLE [13] 则更直接地针对 acceptance length、draft tree 或 speculation lookahead 进行自适应控制。尤其是 OPT-Tree [10] 与 AdaEAGLE [13]，已经显式把"如何最大化期望 acceptance length"作为设计目标的一部分。这些工作与本文最相关，因为 `SILK` 虽然不是 adaptive tree method，但本质上同样是在尝试对 accepted prefix length 施加更细粒度的控制。

### 2.4 与本文的差异

与上述工作相比，`SILK` 的切入点更窄。它不改变 drafter 架构，不改变 verification path，也不显式生成 tree structure；它只尝试回答一个非常具体的问题：若 target token 已经进入 draft `Top-K`，是否可以用一个便宜的上下文控制器纠正排序误差。这个问题看似局部，但其失败模式恰恰揭示了 speculative decoding 的核心困难，即 token-level preference correction 与 block-level acceptance gain 的错位。

## 3. 方法

### 3.1 问题定义

记 draft model 为 `M_d`，target model 为 `M_t`。对任意上下文 `x_<t>`，draft model 在一步 speculative proposal 中生成长度为 `B` 的 token block：

`y_{t:t+B-1}^{(d)} = (y_t^{(d)}, ..., y_{t+B-1}^{(d)})`

target model 对该 block 进行验证，并接受其中与自身一致的最长前缀。定义 acceptance length 为

`L(x_<t>) = max { l in {0,...,B} : y_{t:t+l-1}^{(d)} = y_{t:t+l-1}^{(t)} }`

则 speculative decoding 的真实目标不是提高单点 token accuracy，而是最大化期望 acceptance length：

`J = E_{x_<t>} [L(x_<t>)]`

这一定义解释了为什么单纯优化 next-token preference 可能并不足够。只要 block 中较早位置出现偏差，后续 token 即使局部更接近 target，也无法转化为有效 accepted prefix。

### 3.2 排序失败假设

令 draft model 在位置 `t+i` 产生 logits `s_{t+i} in R^{|V|}`，其 `Top-K` 候选集合记为

`C_{t+i}^{(K)} = TopK(s_{t+i})`

本文的核心假设是，许多 mismatch 满足：

`y_{t+i}^{(t)} in C_{t+i}^{(K)}`

但同时

`rank_{M_d}(y_{t+i}^{(t)}) > 1`

也就是说，错误并非 target token 未被召回，而是它在候选池中的排序位置不够高。若这一假设成立，则无需全词表对齐，一个只在 `Top-K` 内部运行的 reranker 就有机会提升 acceptance length。

### 3.3 SILK 模块

`SILK` 的实现是一个轻量级模块。给定 block hidden states `H = [h_1,...,h_B]`，先构造 block-level context feature：

`g = [ mean(H) ; h_B ]`

再通过小型 MLP 得到潜变量

`z = MLP(g)`

并将其投影回 token embedding space：

`c = W_p z`

对当前步 `Top-K` token embedding `E_K = [e_1,...,e_K]`，定义交互打分

`b_k = < c, e_k >`

得到的 `Top-K` bias 向量 `b` 加回原始 draft logits，仅在候选池内部生效。记 `m_k` 为 `Top-K` mask，则修正后的 logits 为：

`tilde{s}_{t+i} = s_{t+i} + alpha * lambda^i * m(b)`

其中 `alpha` 是全局可学习缩放，`lambda in (0,1]` 是位置衰减系数，`m(b)` 表示只作用于 `Top-K` 对应坐标、其余位置为零的稀疏 bias。于是 `SILK` 的计算成本与 `K` 成正比，而不是与词表规模 `|V|` 成正比。当 `K = 50` 而 `|V| ≈ 50k` 时，计算规模相对全词表干预小约三个数量级。

### 3.4 Block 级平摊与控制机制

为了不吞掉 speculative decoding 的速度收益，`SILK` 不为每个 token 单独构造控制状态，而是在整个 block 上生成一次 context vector，再通过 `lambda^i` 对不同位置的干预强度进行衰减。于是单个 block 的额外成本可写为：

`O(d_h + d_z + K d_e)`

而不是 `O(B |V|)`。

实验早期还发现，不受约束的重排序常常伤害 acceptance rate。为此，推理侧加入了三类控制项：

- `apply_from_step`：仅从 block 后半段位置开始启用 `SILK`；
- `bias_scale`：对最终 bias 进行额外缩放；
- confidence gating：当 draft 自身置信度足够高时关闭 reranking。

这些机制意味着 `SILK` 实际上不是单一静态模块，而是一个带有 intervention policy 的 reranking system。后文将表明，这个 policy 的存在本身就是本文最重要的负证据之一。

### 3.5 训练目标与代理错位

项目围绕 speculative block 中"第一次拒绝发生的位置"构造训练样本。对上下文 `x_<t>`，若第一次分歧发生在位置 `j`，则训练样本由前缀 `x_<t> + y_{t:t+j-1}^{(d)}` 和目标 token `y_{t+j}^{(t)}` 构成。

本文尝试了四类训练路线：

1. `Top-K CrossEntropy`：若 target token 位于 `Top-K` 内，直接在候选池内部做分类。
2. AR 加权 `CrossEntropy`：对更早发生的 rejection 赋予更大权重  
   `w(j) = (B - j) / B`
3. `Top-K KL`：让重排序后的 `Top-K` 分布拟合 target 在同一候选集合上的分布。
4. attack-defense 目标：draft 错时上推 target，draft 对时尽量不破坏原排序。

这些目标都可视为 token-level proxy。其共同局限在于，它们优化的是 `p(y_{t+j}^{(t)} | x_<t>)` 或其局部近似，而不是直接优化 `E[L(x_<t>)]`。这正是后文实验中收益不稳定的理论根源。

## 4. 实验

### 4.1 实验设置

实验使用一个有意控制规模、便于分析的设置：

- Draft model：`GPT-2`（`124M`）[15]
- Target model：`GPT-2-large`（`774M`）[15]
- 训练集：`WikiText-103`
- 测试集：`WikiText-2` [16]
- `context_len = 64`
- `block_size = 8`
- `Top-K = 50`
- 最大训练样本数：`20,000`

我们按如下顺序组织实验：

1. 建立 vanilla speculative decoding 与 distilled draft 基线；
2. 验证 `SILK` 模块能否稳定插入系统；
3. 比较不同训练路线；
4. 分析保守控制、激进控制与大样本评测下的 acceptance behavior。

评价指标包括：

- Acceptance Rate：accepted tokens / drafted tokens；
- Latency：毫秒每 token；
- `match_len_hist`：不同 acceptance length 的频次；
- `diff_positions`：与 vanilla 轨迹发生差异的位置数。

### 4.2 基线与中性插入

实验给出了本文最重要的基线之一。结果如表 1 所示。

**表 1：基线与中性配置**

| 方法 | Acceptance Rate | Latency (ms/token) |
|---|---:|---:|
| Vanilla speculative decoding | 0.2165 | 18.94 |
| Distilled draft | 0.1755 | 17.50 |
| 中性 `SILK` 配置 | 0.2165 | 17.71 |

这个结果说明两件事。第一，蒸馏 draft 虽然更快，但 acceptance rate 明显下降。第二，`SILK` 即使接入系统，也不会自动带来收益。换句话说，`SILK` 的正面结果不能被解释为"模块存在即可提升"，而必须来自更精细的训练与控制。

### 4.3 保守控制下的正向结果

当前可见日志中，最稳定的正向结果来自 AR 加权训练加保守推理控制，即：

- `apply_from_step = 2`
- `bias_scale = 0.8`
- 开启 confidence gating

**表 2：保守配置下的正向结果**

| 方法 | Acceptance Rate | Latency (ms/token) |
|---|---:|---:|
| Vanilla speculative decoding | 0.2165 | 19.13 |
| `SILK best` | 0.2225 | 18.21 |

同日另一条重复日志进一步给出：

| 方法 | Acceptance Rate | Latency (ms/token) |
|---|---:|---:|
| Vanilla speculative decoding | 0.2165 | 18.57 |
| `SILK` | 0.2274 | 18.06 |

这两条日志共同支持一个结论：在当前可见实验中，`SILK` 的正向信号并非单次偶然，而是可以在接近设置下重复出现。更重要的是，这些收益并未以明显的 latency 增长为代价。

### 4.4 敏感性分析与失败模式

如果说正向结果说明了 `SILK` 的机会窗口，那么负向结果才真正说明了它的边界。日志显示，激进配置会将 acceptance rate 显著拉低，如表 3 所示。

**表 3：激进配置的失败模式**

| 方法 | Acceptance Rate | Latency (ms/token) |
|---|---:|---:|
| Vanilla speculative decoding | 0.2165 | 18.60 |
| `SILK` 激进配置 | 0.1721 | 18.29 |

同一条日志中，`diff_positions = 305`，显著高于正向配置下的 `142` 和 `165`。这意味着激进 `SILK` 并非"没有工作"，恰恰相反，它在强力改变生成轨迹，只是改变后的轨迹更早偏离了 target model 可接受的前缀。

**图 1：不同配置下的 acceptance rate（ASCII 示意）**

```text
Vanilla (512)      0.2165 |██████████████████████
SILK neutral       0.2165 |██████████████████████
SILK conservative  0.2225 |███████████████████████
SILK repeat        0.2274 |████████████████████████
SILK aggressive    0.1721 |█████████████████
Vanilla (2048)     0.2354 |█████████████████████████
SILK (2048)        0.2397 |██████████████████████████
```

从这些结果可以提炼出三个稳定信号：

1. 对 block 前部位置的干预最危险；
2. `bias_scale` 的细微变化足以决定方法到底帮助还是伤害；
3. token-level reranking 的改善并不保证 accepted prefix length 的提升。

### 4.5 更大样本验证

实验将评测样本数从 `512` 扩大到 `2048`。结果如表 4 所示。

**表 4：更大样本下的重复验证**

| 方法 | Acceptance Rate | Latency (ms/token) |
|---|---:|---:|
| Vanilla speculative decoding | 0.2354 | 17.61 |
| Distilled draft | 0.1852 | 17.31 |
| `SILK` | 0.2397 | 17.88 |

这条日志的意义不在于绝对数值，而在于 `SILK` 的正向信号没有随着样本量增加而立刻消失。因此，可以较有把握地判断：`SILK` 捕捉到的不是纯粹噪声，而是一个真实但幅度有限、鲁棒性不足的重排序机会。

### 4.6 实验结论

综合所有实验，可得到如下判断：

- `SILK` 作为后处理式 `Top-K` reranker 在工程上是可插拔的；
- 它的增益依赖保守控制，而不是自然出现；
- 其主要失败模式不是"无效"，而是"过度干预导致 accepted prefix 提前崩塌"；
- 实验结果与第三章中的代理错位分析一致，说明未来必须直接面向 acceptance-aware objective 设计训练目标。

## 5. 讨论

本文最重要的发现不是"动态 reranking 有效"，而是"局部 token alignment 与真实 speculative objective 错位"。如果把训练目标写成

`min L_token = - log p(y_{t+j}^{(t)} | x_<t>)`

那么优化的是 target token 在某个位置上的局部概率；而系统真正关心的是

`max J = E[L(x_<t>)]`

二者显然相关，但并不等价。尤其在 block speculative decoding 中，较早位置的小偏差会以路径依赖的方式放大成整个 block 的 acceptance 崩塌。`SILK` 的激进配置之所以失败，正是因为它提升了局部排序变化，却没有学习何时不该改动。

从这个意义上说，`SILK` 更像一个诊断性模块。它证明了"排序失败"确实存在，也证明了 post-hoc reranking 作为干预点在计算上足够便宜；但它同时暴露出，仅靠 token-level proxy 学习出来的 bias 信号并不天然适合 speculative decoding。推理侧必须补上 `apply_from_step`、`bias_scale` 与 confidence gating 这些"刹车系统"，本身就说明未来工作需要把 intervention policy 纳入学习目标。

本文也有明显局限。首先，模型对仍然过小，`GPT-2` / `GPT-2-large` 更适合受控实验，而非生产级结论。其次，`WikiText` 的域过于狭窄，无法覆盖 instruction following、代码、结构化输出与推理任务。再次，`block_size = 8` 有利于分析，但不足以代表更大 speculative block 下的行为。最后，`SILK` 无法召回 `Top-K` 外的 token，因此它天生受限于 draft recall 质量。

## 6. 结论

本文研究了 `SILK`，一种仅作用于 draft `Top-K` 候选集合的动态重排序模块。其出发点很简单：如果 target token 已经被 drafter 召回到候选池中，那么是否可以通过一个廉价 reranker 修正排序误差，并把这种局部修正转化为 speculative decoding 的 acceptance gain。

当前答案是"部分成立，但远未闭环"。`SILK` 在保守配置下能够带来小幅且可重复的 acceptance-rate 提升，并且在更大样本设置中仍保留正向信号；但它也对干预强度极度敏感，激进配置会明显伤害 acceptance behavior。实验结果与理论分析共同说明，speculative decoding 不应继续被视为纯粹的 next-token optimization 问题，而应直接面向 accepted prefix length 或 block-level reward 设计目标函数。

因此，`SILK` 当前最重要的价值不只是一个轻量模块，而是一种更清晰的问题刻画：speculative decoding 中确实存在值得利用的排序机会，但如果训练目标不直接面向 acceptance dynamics，这种机会就很难稳定兑现为部署收益。

## 参考文献

[1] Yaniv Leviathan, Matan Kalman, and Yossi Matias, "Fast Inference from Transformers via Speculative Decoding," 2023. [https://arxiv.org/abs/2211.17192](https://arxiv.org/abs/2211.17192)

[2] Charlie Chen, Sebastian Borgeaud, Geoffrey Irving, Jean-Baptiste Lespiau, Laurent Sifre, and John Jumper, "Accelerating Large Language Model Decoding with Speculative Sampling," 2023. [https://arxiv.org/abs/2302.01318](https://arxiv.org/abs/2302.01318)

[3] Heming Xia, Zhe Yang, Qingxiu Dong, Peiyi Wang, Yongqi Li, Tao Ge, Tianyu Liu, Wenjie Li, and Zhifang Sui, "Unlocking Efficiency in Large Language Model Inference: A Comprehensive Survey of Speculative Decoding," ACL 2024. [https://arxiv.org/abs/2401.07851](https://arxiv.org/abs/2401.07851)

[4] Yuhui Li, Fangyun Wei, Chao Zhang, and Hongyang Zhang, "EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty," ICML 2024. [https://arxiv.org/abs/2401.15077](https://arxiv.org/abs/2401.15077)

[5] Tianle Cai, Yuhong Li, Zhengyang Geng, Hongwu Peng, Jason D. Lee, Deming Chen, and Tri Dao, "Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads," ICML 2024. [https://proceedings.mlr.press/v235/cai24b.html](https://proceedings.mlr.press/v235/cai24b.html)

[6] Jun Zhang, Jue Wang, Huan Li, Lidan Shou, Ke Chen, Gang Chen, and Sharad Mehrotra, "Draft & Verify: Lossless Large Language Model Acceleration via Self-Speculative Decoding," 2023. [https://arxiv.org/abs/2309.08168](https://arxiv.org/abs/2309.08168)

[7] Aonan Zhang, Ray Zhang, Yunfei Cheng, Chong Wang, and Yi Wang, "Recurrent Drafter for Fast Speculative Decoding in Large Language Models," 2024. [https://arxiv.org/abs/2403.09919](https://arxiv.org/abs/2403.09919)

[8] Xupeng Miao, Gabriele Oliaro, Zhihao Zhang, Xinhao Cheng, Zeyu Wang, Rae Ying Yee Wong, Zhuoming Chen, Daiyaan Arfeen, Reyna Abhyankar, and Zhihao Jia, "SpecInfer: Accelerating Generative LLM Serving with Speculative Inference and Token Tree Verification," 2023. [https://arxiv.org/abs/2305.09781](https://arxiv.org/abs/2305.09781)

[9] Yichao Fu, Peter Bailis, Ion Stoica, and Hao Zhang, "Break the Sequential Dependency of LLM Inference Using Lookahead Decoding," ICML 2024. [https://arxiv.org/abs/2402.02057](https://arxiv.org/abs/2402.02057)

[10] Jikai Wang, Yi Su, Juntao Li, Qingrong Xia, Zi Ye, Xinyu Duan, Zhefeng Wang, and Min Zhang, "OPT-Tree: Speculative Decoding with Adaptive Draft Tree Structure," 2024. [https://arxiv.org/abs/2406.17276](https://arxiv.org/abs/2406.17276)

[11] Jonathan Mamou, Oren Pereg, Daniel Korat, Moshe Berchansky, Nadav Timor, Moshe Wasserblat, and Roy Schwartz, "Dynamic Speculation Lookahead Accelerates Speculative Decoding of Large Language Models," 2024. [https://arxiv.org/abs/2405.04304](https://arxiv.org/abs/2405.04304)

[12] Zhuoming Chen, Avner May, Ruslan Svirschevski, Yuhsun Huang, Max Ryabinin, Zhihao Jia, and Beidi Chen, "Sequoia: Scalable, Robust, and Hardware-aware Speculative Decoding," 2024. [https://arxiv.org/abs/2402.12374](https://arxiv.org/abs/2402.12374)

[13] Situo Zhang, Hankun Wang, Da Ma, Zichen Zhu, Lu Chen, Kunyao Lan, and Kai Yu, "AdaEAGLE: Optimizing Speculative Decoding via Explicit Modeling of Adaptive Draft Structures," 2024. [https://arxiv.org/abs/2412.18910](https://arxiv.org/abs/2412.18910)

[14] Sam Stern, Noam Shazeer, and Jakob Uszkoreit, "Blockwise Parallel Decoding for Deep Autoregressive Models," NeurIPS 2018. [https://arxiv.org/abs/1811.03115](https://arxiv.org/abs/1811.03115)

[15] Alec Radford, Jeffrey Wu, Rewon Child, David Luan, Dario Amodei, and Ilya Sutskever, "Language Models are Unsupervised Multitask Learners," 2019. [https://github.com/openai/gpt-2](https://github.com/openai/gpt-2)

[16] Stephen Merity, Nitish Shirish Keskar, and Richard Socher, "Regularizing and Optimizing LSTM Language Models," ICLR 2018. [https://arxiv.org/abs/1708.02182](https://arxiv.org/abs/1708.02182)

[17] Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N. Gomez, Lukasz Kaiser, and Illia Polosukhin, "Attention Is All You Need," NeurIPS 2017. [https://arxiv.org/abs/1706.03762](https://arxiv.org/abs/1706.03762)
