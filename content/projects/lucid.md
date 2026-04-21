---
title: "Lucid"
date: 2026-04-21
weight: 5
description: "面向可部署 Diffusion Language Models 的任务感知解码、对齐与评测"
tags: ["Diffusion LM", "Model Deployment", "DPO", "LoRA"]
repo_url: "https://github.com/sunrx/pace"
report_url: "/posts/lucid-technical-report/"
summary: "让并行 token recovery、任务感知 decoding policy、后训练适配与系统级评测共同构成可部署的技术闭环。"
layout: "single"
likes: 0
---

## 问题意识

Diffusion language models 理论上可通过并行 token recovery 获得更好的速度-质量前沿，但**真实 serving constraints 下的吞吐收益兑现**才是核心难点。需要同时处理 causal attention 兼容性、prefix cache 复用、任务差异与后训练对齐。

## 核心洞察

`Lucid` 将部署问题建模为 **operating point 选择问题**：在不同任务族上找到更合理的 quality-throughput (Q-T) 权衡点。建立在三层结构上：

1. **Diffusion serving substrate**：基于 `WeDLM` 的 causal-attention decoding
2. **Decoding policy**：任务感知的并行控制（window size、entropy threshold、position penalty）
3. **Adaptation**：LoRA 适配 + adapter/merge 模型形态

## 主要结论

| 任务族 | 状态 | 关键结果 |
|---|---|---|
| **知识类** | 稳定但形态敏感 | ARC/MMLU 基本稳定，HellaSwag 对模型形态极其敏感 |
| **推理类** | 关键短板 | GSM8K/MATH 仍未追上 AR 基线，TPF↑但质量损失难接受 |
| **代码类** | **最强正向证据** | MBPP 71.25/1.63、HumanEval 在 TPF~1.40 追回 AR 质量 |

**核心判断**：Diffusion decoding 的吞吐收益真实存在，但部署价值高度依赖**任务感知 policy**，而非单一全局 preset。

## 关键发现

Adapter vs Merge 不是实现细节——在 MBPP 上 `Lucid-8B (adapter)` 明显落后于 merged 形态，说明 **model form factor 本身就是系统设计的一等问题**。

当前最有价值的产出不是"diffusion 全面胜过 AR"，而是一个更可操作的研究框架：
- 哪些任务值得激进并行
- 哪些任务必须保守回退
- 哪些对齐收益能在真实 serving path 上保留

[阅读完整技术报告 →](/posts/lucid-technical-report/)
