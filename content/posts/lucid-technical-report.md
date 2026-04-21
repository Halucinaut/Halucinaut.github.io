---
title: "Lucid：面向可部署 Diffusion Language Models 的任务感知解码与对齐"
date: 2026-04-21
description: "关于 deployable diffusion language models 的系统项目报告"
tags: ["Diffusion LM", "Model Deployment", "DPO", "LoRA", "Technical Report"]
cover:
    image: ""
    alt: "Lucid"
---

# Lucid：面向可部署 Diffusion Language Models 的任务感知解码、对齐与评测

作者：Sunrx  
日期：2026 年 4 月  
类型：技术报告 / 项目说明

定位：一篇关于 deployable diffusion language models 的系统项目报告。

## 摘要

本文记录 `Lucid` 这一研究项目，其目标不是让 diffusion language model 只在零散 benchmark 上"看起来可行"，而是让并行 token recovery、任务感知 decoding policy、后训练适配与系统级评测共同构成一个可部署的技术闭环。项目当前建立在 `WeDLM` 的 causal-attention diffusion serving substrate 之上，并在此基础上叠加 LoRA 适配、adapter/merge 两类模型形态，以及围绕 quality-throughput tradeoff 的 preset 搜索与 benchmark 体系。

结果表明，`Lucid` 的核心价值不在于"统一超越所有基线"，而在于更清楚地刻画不同任务族上的 Pareto 行为：知识类任务整体稳定但对模型形态敏感，推理类任务仍由 `WeDLM-8B-Inst (AR)` 占优，而代码类任务则已经出现了较有说服力的正向信号，例如 `MBPP` 上 `71.25 / 1.63`、`HumanEval` 上在 `1.40` 左右 `TPF` 下追回 AR 基线质量。本文的主要结论因此是：`Lucid` 已经证明了 diffusion decoding 的吞吐收益真实存在，但其部署价值高度依赖任务感知 policy，而非单一全局 preset。

**证据边界**

本文把 `TPF` 作为吞吐代理指标，而不是完整 wall-clock latency；因此本文更适合作为项目主页上的 system report，而不是一篇已经给出完整生产级性能结论的论文。

## 1. 引言

Autoregressive language models 在部署侧长期占优，不仅因为效果强，还因为它们天然兼容 prefix cache、拥有成熟的 serving engine，并且生成过程语义清晰。Diffusion language models 提供了另一条路线：通过每轮恢复多个 token，减少严格串行的解码步数，从而在理论上获得更好的速度-质量前沿。

真正困难的问题不在于"能否生成像样文本"，而在于"能否在真实 serving constraints 下兑现吞吐收益"。一旦把问题放到部署层面，diffusion 路线必须同时处理 causal attention 兼容性、prefix cache 复用、engine 稳定性、任务差异以及后训练对齐等问题。也正因此，单一的模型分数不再足够；需要被优化的是一个 joint system，而不是若干独立组件。

`Lucid` 正是在这个背景下形成的。项目不把 diffusion decoding、alignment 与 evaluation 分开处理，而是把它们组织为一个统一的部署问题：给定一个基础 diffusion serving substrate，如何设计任务感知 policy、轻量适配与评测体系，使模型在不同任务族上获得更合理的 quality-throughput operating point。

本文有两个目标。第一，将当前文档重写为更接近正式论文的结构。第二，在不夸大证据边界的前提下，用最终结果来回答一个更实际的问题：`Lucid` 当前到底在哪些任务上形成了可公开强调的 Pareto 改善，哪些任务仍然明显未闭环。

## 2. 相关工作

### 2.1 Diffusion language models

近年来，diffusion language models 的研究重心逐渐从"是否能生成可读文本"转向"是否能在真实系统中兑现并行恢复的收益"。LLaDA [1]、Dream 7B [2]、Mercury [3] 与 Simple and Effective Masked Diffusion Language Models [4] 从不同角度证明，离散 diffusion 路线在质量、计划能力、代码与推理等任务上已经具有竞争力。然而，这些工作也共同暴露出一个系统问题：理论上的并行 token recovery 不会自动转化为可部署的 wall-clock gain。

### 2.2 Causal-attention diffusion serving

`WeDLM` [5] 在本文中占据核心位置，因为它直接回应了 diffusion serving 的关键系统障碍。很多 diffusion language model 依赖 bidirectional attention，这会破坏标准 prefix KV cache，使得并行生成在工程上难以与成熟 AR engine 竞争。`WeDLM` 的核心贡献在于用 standard causal attention 组织 diffusion decoding，使其能够与 prefix-cache friendly serving stack 更好兼容。

### 2.3 轻量适配与偏好对齐

在后训练适配方面，LoRA [6] 已成为大模型轻量适配的标准技术之一，因为它用极低的参数代价为任务定向适配提供了工程可行性。进一步的 preference alignment 方法，如 DPO [7]，则将人类偏好或任务偏好引入到更轻量的优化框架中。对 `Lucid` 而言，LoRA 与 DPO 的意义不只在于质量提升，还在于它们是否能在 diffusion serving path 上保持稳定。

### 2.4 Benchmark 与系统评测

`Lucid` 的实验覆盖知识类、推理类与代码类 benchmark，包括 ARC [8]、HellaSwag [9]、MMLU [10]、GSM8K [11]、MATH [12]、HumanEval [13]、MBPP [14] 与 C-Eval [15]。这些 benchmark 的共同作用不只是比较绝对分数，而是揭示不同任务族对并行恢复误差的敏感程度。本文的评测立场因此更接近 Pareto frontier 分析，而非单点准确率比较。

## 3. 方法

### 3.1 系统目标

`Lucid` 把 diffusion language model 的部署问题建模为一个 operating point 选择问题。对任意任务族 `tau`，定义质量指标为 `Q_tau`，吞吐代理指标为 `T_tau`，则一个配置 `c` 的系统效用可以抽象写为：

`U(c; tau) = Q_tau(c) - lambda_tau * C(T_tau(c))`

其中 `C(.)` 表示吞吐成本函数，`lambda_tau` 表示任务对质量损失的敏感度。本文并不试图在实验中直接估计这一效用函数，而是通过多任务 benchmark 观测不同配置落在什么 Pareto 区域。这个公式只是把 `Lucid` 的方法论写得更清楚：目标不是最大化单一 accuracy，而是在不同任务族上找到更好的 `Q-T` operating point。

### 3.2 Lucid 的三层结构

`Lucid` 建立在三层相互耦合的结构上。

第一层是 diffusion serving substrate，即 `WeDLM` 的 causal-attention decoding 路径。其作用是保证并行恢复在系统上仍然兼容标准 cache 语义，而不是沦为离线实验里的并行玩具。

第二层是 decoding policy。当前 native engine 已经支持三类控制量：

- `wedlm_window_size`
- `wedlm_entropy_threshold`
- `wedlm_pos_penalty_factor`

它们共同决定"何时激进并行、何时保守回退、哪些位置的并行风险更高"。

第三层是 adaptation 与 model form factor。当前项目同时维护了：

- `Lucid-8B (AR)`
- `Lucid-8B (0.4, 0.2)`
- `Lucid-8B (adapter)`

这三种形态分别回答不同问题：适配本身是否有效；进入 diffusion preset 后，模型行为如何变化；adapter 与 merge 是否存在系统性差异。

### 3.3 任务感知 policy 设计

`Lucid` 的基本判断是：不同任务族不应共享单一并行 preset。设解码状态为 `s_t`，并行决策为 `a_t`，则 policy 层可以抽象为：

`a_t = pi_phi(s_t)`

其中 `s_t` 至少包含当前位置 entropy、已填充比例、当前位置风险、最近几轮 fill ratio 等局部状态；`a_t` 则决定 window size、是否并行填充以及 penalty 强度。当前工程实现还没有把 `pi_phi` 学成一个显式神经 policy，而是采用规则化、可解释的低层控制逻辑。本文因此将其称为"任务感知 decoding policy"，而不是"learned controller"。

### 3.4 适配目标与工程形态

在训练侧，`Lucid` 当前的稳定主线是基于 LoRA 的监督式适配。从系统设计角度，它承担两个角色：

1. 将 base diffusion model 的行为向更目标化的任务分布拉近；
2. 在尽量不破坏 serving path 的前提下，缓解并行恢复带来的输出漂移。

项目同时保留了更激进的对齐路线，例如 DPO 和 trajectory alignment 方案。但在本文中，这些路线都被视为后续方向，而不是最终结果归因的主体。

### 3.5 评测定义

本文统一使用最终实验结果。对 diffusion 配置，表中同时记录质量指标和 `TPF`，记作：

`R_tau(c) = (Q_tau(c), T_tau(c))`

其中 `T_tau(c)` 即 `tokens per forward`，用来近似刻画 diffusion decoding 的并行恢复效率。需要强调的是，`TPF` 不是完整 wall-clock latency，但它足以揭示不同 preset 是否真的在"每次前向恢复更多 token"这一点上取得系统收益。

## 4. 实验

### 4.1 实验口径

本文所有最终结果都来自实验总表。该文件同时给出了：

- 外部对照模型：`Qwen2.5-7B-Inst`、`Qwen3-8B-Inst`、`Dream-7B`、`TiDAR`
- 内部 AR 形态：`WeDLM-8B-Inst (AR)`、`Lucid-8B (AR)`
- diffusion 形态：`WeDLM-8B-Inst (0.4,0.2)`、`Lucid-8B (0.4,0.2)`、`Lucid-8B (adapter)`

从任务族划分看，实验覆盖：

- 知识类：`ARC-C`、`ARC-E`、`HellaSwag`、`MMLU`
- 推理类：`GSM8K`、`MATH`
- 代码类：`MBPP`、`HumanEval`、`HumanEval+`

本文不再回到历史日志中重新挑选单次 runs，而统一用这张结果总表来分析 `Lucid` 当前最稳定、最适合公开叙述的结论。

### 4.2 知识类任务

知识类任务结果如表 1 所示。

**表 1：知识类任务结果**

| Benchmark | WeDLM-8B-Inst (AR) | Lucid-8B (AR) | WeDLM-8B-Inst `(0.4,0.2)` | Lucid-8B `(0.4,0.2)` | Lucid-8B `(adapter)` |
|---|---:|---:|---:|---:|---:|
| ARC-C | 92.92 | 93.24 | 93.40 / 1.14 | 93.40 / 1.14 | 93.40 / 1.14 |
| ARC-E | 97.43 | 97.42 | 97.52 / 1.17 | 97.44 / 1.17 | 97.44 / 1.17 |
| HellaSwag | 82.94 | 89.26 | 93.61 / 1.11 | 89.26 / 1.12 | 89.26 / 1.12 |
| MMLU | 75.14 | 75.36 | 75.30 / 1.10 | 75.33 / 1.10 | 75.33 / 1.10 |

这组结果呈现出两个信号。第一，`Lucid-8B (AR)` 相对 `WeDLM-8B-Inst (AR)` 并不差，说明 LoRA 适配本身不是纯粹破坏性的。第二，进入 diffusion preset 之后，任务差异被迅速放大：`ARC-C` 与 `ARC-E` 基本稳定，`MMLU` 波动很小，但 `HellaSwag` 对模型形态极其敏感，`WeDLM` preset 达到 `93.61 / 1.11`，而 `Lucid` diffusion 形态退回 `89.26 / 1.12`。

这说明知识类任务并不存在一个统一最优的 `Lucid` operating point。某些任务更偏好保留底座模型配并行 preset，另一些任务则要求更保守的适配形态。

### 4.3 推理类任务

推理类任务结果如表 2 所示。

**表 2：推理类任务结果**

| Benchmark | WeDLM-8B-Inst (AR) | Lucid-8B (AR) | WeDLM-8B-Inst `(0.4,0.2)` | Lucid-8B `(0.4,0.2)` | Lucid-8B `(adapter)` |
|---|---:|---:|---:|---:|---:|
| GSM8K | 92.27 | 90.75 | 91.51 / 1.45 | 90.67 / 1.44 | 90.67 / 1.44 |
| MATH | 64.80 | 60.20 | 63.20 / 1.49 | 60.20 / 1.48 | 60.20 / 1.48 |

这里的结论最直接：在当前最终结果中，`WeDLM-8B-Inst (AR)` 仍是更稳的推理基线。`Lucid` 无论在 AR 形态还是 `(0.4, 0.2)` diffusion preset 下，都没有追回 `GSM8K` 与 `MATH` 的差距。与此同时，diffusion preset 的 `TPF` 依然明显高于 `1.0`，达到 `1.44` 到 `1.49`。

这意味着 `Lucid` 在推理任务上的问题不是"没有吞吐收益"，而是"吞吐收益尚不足以覆盖质量损失"。从部署角度看，这是一种更重要也更诚实的结论，因为它明确指出了当前系统最需要继续打磨的任务区域。

### 4.4 代码类任务

代码类任务结果如表 3 所示。

**表 3：代码类任务结果**

| Benchmark | WeDLM-8B-Inst (AR) | Lucid-8B (AR) | WeDLM-8B-Inst `(0.4,0.2)` | Lucid-8B `(0.4,0.2)` | Lucid-8B `(adapter)` |
|---|---:|---:|---:|---:|---:|
| MBPP | 70.53 | 70.23 | 69.51 / 1.66 | 71.25 / 1.63 | 69.51 / 1.63 |
| HumanEval | 80.49 | 79.88 | 77.44 / 1.42 | 80.49 / 1.40 | 80.49 / 1.40 |
| HumanEval+ | 73.78 | 75.00 | 70.73 / 1.42 | 73.17 / 1.40 | 73.17 / 1.40 |

这是 `Lucid` 当前最有说服力的一组结果。`MBPP` 上，`Lucid-8B (0.4,0.2)` 达到 `71.25 / 1.63`，不仅高于 `WeDLM` diffusion 形态，也高于 `WeDLM-8B-Inst (AR)`。`HumanEval` 上，`Lucid` diffusion 形态把 `WeDLM` 的 `77.44 / 1.42` 拉回到 `80.49 / 1.40`，等于追回了 AR base 的质量。`HumanEval+` 上虽然未超过 `Lucid-8B (AR)` 的 `75.00`，但相对 `WeDLM` diffusion 形态依然明显更强。

这表明代码任务是 `Lucid` 当前最值得公开强调的正向证据：适配并非只是"减缓 diffusion 损失"，而是在某些任务上已经将系统推到了比 AR baseline 更好的 Pareto 点附近。

### 4.5 Adapter 与 merge 的差异

结果还揭示了一个工程上很重要的现象：adapter 与 merge 不是可互换形态。在 `HumanEval`、`HumanEval+` 上，两者结果一致；但在 `MBPP` 上，`Lucid-8B (adapter)` 明显落后于 merged `Lucid-8B`。这说明 model form factor 本身就是系统设计的一部分，而不是实现细节。未来如果要把 `Lucid` 推向更稳定的 serving path，就必须把 adapter/merge 差异当作一等问题来分析。

### 4.6 总体结果图景

图 1 用 ASCII 形式总结 `Lucid` 在三类任务上的总体 Pareto 图景。

**图 1：Lucid 当前的任务族 Pareto 图景（示意）**

```text
知识类   : 质量总体稳定，个别任务对模型形态敏感
推理类   : TPF 明显上升，但质量下降更难接受
代码类   : 在 TPF > 1.4 条件下，已出现可公开强调的质量恢复/提升
```

综合表 1 到表 3，可得到四个当前阶段最稳的结论：

1. diffusion decoding 的吞吐收益是真实存在的；
2. 这种收益不会自动转化为统一的质量增益；
3. 代码任务是当前最强的正向证据；
4. 推理任务仍然是当前最关键的短板。

## 5. 讨论

`Lucid` 当前最重要的技术启示，是 diffusion language model 的部署问题本质上是 policy problem，而不是单一 model problem。只要系统目标写成 `Q-T` operating point 的选择，那么不同任务族对质量下降的容忍度必然不同。代码生成任务在当前结果中已证明可以接受更高的 `TPF` 并维持或追回质量；而 reasoning-heavy 任务则更容易因并行恢复误差而失去任务成功率。

这也是为什么本文不把 `Lucid` 描述成一个"统一优于 AR" 的系统。当前证据支持的是更窄但更扎实的判断：并行恢复的系统收益已经存在，但其部署价值高度依赖任务感知 policy。换言之，真正应该被学习或被工程化的对象，不只是更好的 diffusion base model，而是"哪类任务值得激进并行、哪类任务必须保守回退"的 decision layer。

本文同时也保留几个清楚的局限。第一，最终结果来自结果总表，它适合给出跨任务的 Pareto 轮廓，但还不足以替代更完整的 wall-clock latency、重复运行方差与服务稳定性统计。第二，`TPF` 不是完整部署指标，它只能近似刻画每次前向恢复多少 token，而不能单独代表端到端成本。第三，当前最佳结果高度任务依赖；代码任务已经出现有说服力的正向信号，但推理任务仍未闭环。第四，DPO 与 trajectory alignment 仍处于规划阶段，因此本文没有把更激进的对齐方法纳入最终结论。

## 6. 结论

本文从部署视角总结了 `Lucid` 当前的系统结构、方法设计与最终结果。相较于把 diffusion language model 视为单点模型改进，`Lucid` 更适合被理解为一个 joint system：它把 causal-attention diffusion decoding、任务感知 policy、轻量适配、adapter/merge 形态选择和 Pareto 式评测放进同一个框架中。

基于最终结果，可以给出一个克制而清楚的判断：`Lucid` 已经证明了 diffusion decoding 的吞吐收益是真实存在的；代码任务上已经出现值得公开强调的 Pareto 改善；知识任务总体稳定但形态敏感；推理任务仍然是最需要继续攻克的区域。因此，`Lucid` 当前最重要的价值不在于宣称"diffusion 已经全面胜过 AR"，而在于把部署问题压缩成一个更可操作的研究框架：哪些任务值得并行，哪些任务不值得，哪些对齐收益能在真实 serving path 上保留下来。

## 参考文献

[1] Shen Nie, Fengqi Zhu, Zebin You, Xiaolu Zhang, Jingyang Ou, Jun Hu, Jun Zhou, Yankai Lin, Ji-Rong Wen, and Chongxuan Li, "Large Language Diffusion Models," 2025. [https://arxiv.org/abs/2502.09992](https://arxiv.org/abs/2502.09992)

[2] Jiacheng Ye, Zhihui Xie, Lin Zheng, Jiahui Gao, Zirui Wu, Xin Jiang, Zhenguo Li, and Lingpeng Kong, "Dream 7B: Diffusion Large Language Models," 2025. [https://arxiv.org/abs/2508.15487](https://arxiv.org/abs/2508.15487)

[3] Inception Labs, Samar Khanna, Siddhant Kharbanda, Shufan Li, Harshit Varma, Eric Wang, Sawyer Birnbaum, Ziyang Luo, Yanis Miraoui, Akash Palrecha, Stefano Ermon, Aditya Grover, and Volodymyr Kuleshov, "Mercury: Ultra-Fast Language Models Based on Diffusion," 2025. [https://arxiv.org/abs/2506.17298](https://arxiv.org/abs/2506.17298)

[4] Subham Sekhar Sahoo, Marianne Arriola, Yair Schiff, Aaron Gokaslan, Edgar Marroquin, Justin T. Chiu, Alexander Rush, and Volodymyr Kuleshov, "Simple and Effective Masked Diffusion Language Models," 2024. [https://arxiv.org/abs/2406.07524](https://arxiv.org/abs/2406.07524)

[5] Aiwei Liu, Minghua He, Shaoxun Zeng, Sijun Zhang, Linhao Zhang, Chuhan Wu, Wei Jia, Yuan Liu, Xiao Zhou, and Jie Zhou, "WeDLM: Reconciling Diffusion Language Models with Standard Causal Attention for Fast Inference," 2025. [https://arxiv.org/abs/2512.22737](https://arxiv.org/abs/2512.22737)

[6] Edward J. Hu, Yelong Shen, Phillip Wallis, Zeyuan Allen-Zhu, Yuanzhi Li, Shean Wang, Lu Wang, and Weizhu Chen, "LoRA: Low-Rank Adaptation of Large Language Models," 2021. [https://arxiv.org/abs/2106.09685](https://arxiv.org/abs/2106.09685)

[7] Rafael Rafailov, Archit Sharma, Eric Mitchell, Christopher D. Manning, Stefano Ermon, and Chelsea Finn, "Direct Preference Optimization: Your Language Model is Secretly a Reward Model," NeurIPS 2023. [https://arxiv.org/abs/2305.18290](https://arxiv.org/abs/2305.18290)

[8] Peter Clark, Isaac Cowhey, Oren Etzioni, Tushar Khot, Ashish Sabharwal, Carissa Schoenick, and Oyvind Tafjord, "Think You Have Solved Question Answering? Try ARC, the AI2 Reasoning Challenge," 2018. [https://arxiv.org/abs/1803.05457](https://arxiv.org/abs/1803.05457)

[9] Rowan Zellers, Ari Holtzman, Yonatan Bisk, Ali Farhadi, and Yejin Choi, "HellaSwag: Can a Machine Really Finish Your Sentence?" ACL 2019. [https://arxiv.org/abs/1905.07830](https://arxiv.org/abs/1905.07830)

[10] Dan Hendrycks, Collin Burns, Steven Basart, Andy Zou, Mantas Mazeika, Dawn Song, and Jacob Steinhardt, "Measuring Massive Multitask Language Understanding," ICLR 2021. [https://arxiv.org/abs/2009.03300](https://arxiv.org/abs/2009.03300)

[11] Karl Cobbe, Vineet Kosaraju, Mohammad Bavarian, Mark Chen, Heewoo Jun, Lukasz Kaiser, Matthias Plappert, Jerry Tworek, Jacob Hilton, Reiichiro Nakano, Christopher Hesse, and John Schulman, "Training Verifiers to Solve Math Word Problems," 2021. [https://arxiv.org/abs/2110.14168](https://arxiv.org/abs/2110.14168)

[12] Dan Hendrycks, Collin Burns, Saurav Kadavath, Akul Arora, Steven Basart, Eric Tang, Dawn Song, and Jacob Steinhardt, "Measuring Mathematical Problem Solving With the MATH Dataset," 2021. [https://arxiv.org/abs/2103.03874](https://arxiv.org/abs/2103.03874)

[13] Mark Chen, Jerry Tworek, Heewoo Jun, Qiming Yuan, Henrique Ponde de Oliveira Pinto, Jared Kaplan, Harri Edwards, Yuri Burda, Nicholas Joseph, Greg Brockman, Alex Ray, Raul Puri, Gretchen Krueger, Michael Petrov, Heidy Khlaaf, Girish Sastry, Pamela Mishkin, Brooke Chan, Scott Gray, Nick Ryder, Mikhail Pavlov, Alethea Power, Lukasz Kaiser, Mohammad Bavarian, Clemens Winter, Philippe Tillet, Felipe Petroski Such, Dave Cummings, Matthias Plappert, Fotios Chantzis, Elizabeth Barnes, Ariel Herbert-Voss, William Hebgen Guss, Alex Nichol, Alex Paino, Nikolas Tezak, Jie Tang, Igor Babuschkin, Suchir Balaji, Shantanu Jain, William Saunders, Christopher Hesse, Andrew N. Carr, and others, "Evaluating Large Language Models Trained on Code," 2021. [https://arxiv.org/abs/2107.03374](https://arxiv.org/abs/2107.03374)

[14] Jacob Austin, Augustus Odena, Maxwell Nye, Maarten Bosma, Henryk Michalewski, David Dohan, Ellen Jiang, Carrie Cai, Michael Terry, Quoc Le, and Charles Sutton, "Program Synthesis with Large Language Models," 2021. [https://arxiv.org/abs/2108.07732](https://arxiv.org/abs/2108.07732)

[15] Yuzhen Huang, Yuzhuo Bai, Zhuoyue Zhao, Yichen Zhang, and Weinan Zhang, "C-Eval: A Multi-Level Multi-Discipline Chinese Evaluation Suite for Foundation Models," NeurIPS 2023 Datasets and Benchmarks. [https://arxiv.org/abs/2305.08322](https://arxiv.org/abs/2305.08322)
