---
title: "Nano Ant"
date: 2025-04-16
weight: 3
description: "面向迭代优化的轻量级 Agent 框架。支持 prompt 优化、工作流节点优化等任务，提供内部模式和外部模式两种使用方式。"
tags: ["Agent Framework", "Prompt Optimization", "Python", "Iteration", "LLM Tools"]
demo_url: ""
repo_url: "https://github.com/Halucinaut/Nano-ant"
summary: "轻量级迭代优化框架，让 Agent 能连续执行、连续评估、连续修正，把一次性生成变成可迭代优化。"
layout: "single"
likes: 0
---

## 📖 项目简介

很多任务表面上在做生成，真正难的是迭代。一个 prompt 第一次写出来通常只能跑通一部分样本；一个工作流节点在新数据上总会暴露新的边界问题；一个已经上线的脚本，稳定性往往来自反复修改、反复运行、反复审查。

**Nano Ant** 就是沿着这个思路做出来的。它不试图吞掉业务系统，也不试图把一切任务包装成一个庞大的自治平台。它只包住那条最关键的闭环：**给定一个明确的优化对象，让 Agent 能连续执行、连续评估、连续修正，把一次性生成变成可迭代优化**。

## 🎯 核心能力

### 模式一：内部模式 (Internal Mode)

将优化目标、测试用例、执行脚本都放在 Nano Ant 内部：
- Nano Ant 控制一切：资源加载、执行、评估、迭代
- 用户按照 Nano Ant 的规范准备材料
- 提供默认的评估脚本实现

### 模式二：外部模式 (External Mode)

外部已有成熟的项目、执行流程和评估体系，只想借用 Nano Ant 的迭代优化能力：
- 外部项目保持独立，有自己的结构和依赖
- Nano Ant 只负责迭代优化循环
- 通过适配器对接外部项目

### 统一的 JudgeSkill

无论内部模式还是外部模式，都通过 **JudgeSkill** 定义评估标准，这是用户与 Nano Ant 评估系统的核心接口。

## 🚀 快速开始

```bash
git clone https://github.com/Halucinaut/Nano-ant.git
cd "Nano ant"
pip install -e .
```

运行交互式 CLI：

```bash
ant --config config.local.yaml
```

进入后执行示例任务：

```
/run ./examples/product_prompt_sample
```

## 📦 版本路线

| 版本 | 用户可见能力 | 状态 |
| --- | --- | --- |
| v0.3.x | 内部任务目录运行、外部项目接入、基于 `prompt_use.md` 的多轮 prompt 优化、JudgeSkill、checkpoint、交互式 `ant` CLI、本地配置隔离 | ✅ 已支持 |
| v0.4.x | 更多面向产品同学的任务模板、更多可优化对象、统一任务包分发方式 | 📝 计划中 |
| v0.5.x | 更强的跨任务策略复用、更完整的自我进化沉淀、更广的 workflow 优化场景 | 📝 计划中 |

## 🔗 相关链接

- [GitHub 仓库](https://github.com/Halucinaut/Nano-ant)
- [English README](https://github.com/Halucinaut/Nano-ant/blob/main/README_EN.md)
