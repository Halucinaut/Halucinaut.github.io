---
title: "Nano Ant"
date: 2025-04-16
weight: 3
description: "不是又一个 Agent 框架，而是专注于'迭代优化'这个被忽视的核心环节。"
tags: ["Agent Framework", "Prompt Optimization", "Python", "Iteration", "LLM Tools"]
cover:
    image: "images/nano-ant-cover.png"
    alt: "Nano Ant"
demo_url: ""
repo_url: "https://github.com/Halucinaut/Nano-ant"
summary: "轻量级迭代优化框架。给定明确的优化对象，让 Agent 能连续执行、评估、修正，把一次性生成变成可迭代优化。"
layout: "single"
likes: 0
---

## 问题意识

很多任务表面上在做生成，真正难的是迭代。一个 prompt 第一次写出来通常只能跑通一部分样本；一个工作流节点在新数据上总会暴露新的边界问题；一个已经上线的脚本，稳定性往往来自反复修改、反复运行、反复审查。

现有的 Agent 框架都在扩张能力边界（多工具、多 Agent、长期记忆），却很少有人专注在"迭代优化"这个最耗人力的环节。

## 核心洞察

`Nano Ant` 只包住那条最关键的闭环：给定一个明确的优化对象，让 Agent 能**连续执行、连续评估、连续修正**，把一次性生成变成可迭代优化。

优化对象可以是 prompt、规则、局部流程。框架本身保持轻量，只要求用户提供：一个可优化对象、一个可运行回路、一个评审标准。

## 它怎么工作

你给 `Nano Ant` 一个任务目录。每一轮里，它修改优化对象，调用现有运行脚本，读取结构化结果，再让 Judge 基于评审规则给出下一轮修正方向。

外部项目继续保留自己的执行逻辑，`Nano Ant` 只负责迭代优化。

## 与其他框架的差异

| 项目 | 定位 | Nano Ant 的差异 |
| --- | --- | --- |
| Voyager | Minecraft 开放世界终身学习 | 不做具身探索，专注业务任务的局部优化闭环 |
| Reflexion | 通过语言反馈持续改进 | 受启发但重点在任务目录、结果协议、JudgeSkill 的工程运行时 |
| AgentVerse | 多 Agent 协作与 simulation | 不追求多 Agent 社会化协作，主线是单任务多轮优化 |

**一句话概括**：很多框架在扩张 Agent 的能力边界，`Nano Ant` 在压缩迭代优化的接入成本。

## 当前状态

| 版本 | 能力 | 状态 |
| --- | --- | --- |
| v0.3.x | 内部/外部任务运行、prompt 优化、JudgeSkill、checkpoint、交互式 CLI | ✅ 已支持 |
| v0.4.x | 更多任务模板、更多可优化对象 | 📝 计划中 |
| v0.5.x | 跨任务策略复用、workflow 优化 | 📝 计划中 |
