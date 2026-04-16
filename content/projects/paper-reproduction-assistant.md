---
title: "Paper Reproduction Assistant"
date: 2025-01-28
weight: 2
description: "A standardized paper reproduction expert knowledge base. Injects 'top-tier conference reviewer' level code audit and replication capabilities into AI coding assistants."
tags: ["Paper Reproduction", "Code Audit", "SOTA", "Deep Learning"]
cover:
    image: "images/paper-assistant-cover.jpg"
    alt: "Paper Reproduction Assistant"
demo_url: ""
repo_url: "https://github.com/Halucinaut/halucinaut-skills/tree/main/skills/research%26study/paper-reproduction-assistant"
summary: "为 AI 编程助手注入顶会审稿人级别的代码审计与复现能力。"
parent_project: "Halucinaut Skills Registry"
likes: 256
---
## 📖 项目简介

这是一个标准化的论文复现专家知识库。旨在为 AI 编程助手注入"顶会审稿人"级别的代码审计与复现能力。它不仅仅是一堆 Prompt，而是一套完整的 SOP。它强制 AI 遵循以下逻辑：

1. **拒绝盲从**：不盲目生成代码，先进行环境与逻辑审计。
2. **双轨机制**：针对"已有代码"和"无代码"提供两套完全不同的处理流。
3. **范式强制**：强制使用 2026 年主流的 SOTA 工程范式（如 FlashAttention-3, Flow Matching, MoE），拒绝过时的代码风格。

## 🎯 核心能力

### Branch A: 审计已有代码库

- 环境兼容性检查（CUDA、PyTorch、依赖版本）
- 代码逻辑正确性验证
- 潜在 Bug 和风险识别
- 性能瓶颈分析

### Branch B: 从论文重构代码

- 论文核心算法提取
- SOTA 工程范式应用
- 维度推导和形状检查
- 可复现性保证

## 🚀 使用方式

访问完整仓库：

```bash
git clone https://github.com/Halucinaut/halucinaut-skills.git
cd halucinaut-skills/skills/research\&study/paper-reproduction-assistant
```
