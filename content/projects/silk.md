---
title: "SILK"
date: 2026-04-21
weight: 4
description: "面向 Speculative Decoding 的动态 Top-K 重排序方法"
tags: ["Speculative Decoding", "Inference Optimization", "LLM"]
repo_url: "https://github.com/sunrx/silk"
report_url: "/posts/silk-technical-report/"
summary: "一种仅作用于 draft model Top-K 候选集合的轻量级、上下文相关重排序模块。"
layout: "single"
likes: 0
---

## 问题意识

Speculative Decoding 通过"小模型起草、大模型验证"降低推理延迟，但真实收益高度依赖 acceptance rate。许多 draft-target mismatch 不是召回失败，而是**排序失败**——target token 已在 Top-K 内，但未排在第一位。

## 核心洞察

`SILK` 在 frozen drafter 与最终 token 决策之间插入小型控制器，**仅对 Top-K logits 施加动态 bias**，而不做全词表计算。计算成本与 `K` 成正比（如 K=50, |V|=50k 时，规模缩小约三个数量级）。

## 主要结论

| 配置 | Acceptance Rate | 结论 |
|---|---|---|
| 保守控制 | 0.2165 → 0.2225/0.2274 | 方向成立，收益可重复 |
| 激进配置 | 降至 0.1721 | 过度干预导致提前崩塌 |
| 大样本(2048) | 0.2354 → 0.2397 | 信号稳定，非纯噪声 |

后处理式 Top-K 重排序确实识别到了真实的排序机会，但**局部 token 偏好对齐并不自动转化为 block 级 accepted prefix length 的提升**。

## 关键发现

实验结果揭示的深层问题：训练目标优化的 token-level proxy 与系统真正关心的 `E[L(x_<t>)]`（期望 acceptance length）存在**代理错位**。这正是收益不稳定的理论根源。

**SILK 最重要的价值**不是提出一个成熟的加速模块，而是更精确地刻画了 acceptance-aware optimization 的必要性。

[阅读完整技术报告 →](/posts/silk-technical-report/)
