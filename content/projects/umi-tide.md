---
title: "UMI-TIDE"
date: 2026-05-09
weight: 3
description: "工具智能体的长期记忆系统。让 AI 记住什么值得再次使用，而不只是记住什么曾经发生过。"
tags: ["Agent Memory", "Tool Use", "Evaluation", "Cognitive Architecture"]
cover:
    image: "images/umi_tide-cover.png"
    alt: "UMI-TIDE Architecture"
demo_url: ""
repo_url: "https://github.com/Halucinaut/UMI-TIDE"
summary: "收益驱动的工具智能体记忆系统。用任务结果判断经验价值，自动淘汰低效用记录。"
likes: 0
---

## 问题

AI 智能体能记住很多，但分不清哪些经验真的帮到了任务。

结果是：重复试错、无效经验堆积、关键时刻召回的是过时的答案。

## 核心洞察

**长期记忆的风险不是"记得不够多"，而是"不知道什么值得再次使用"。**

UMI-TIDE 在任务完成后给经验打分，用实际结果判断价值。高分的留下，低分的自动废弃。

## 产品形态

一个零依赖的 Python 框架，几行代码即可接入你的工具智能体：

```python
from umi_tide import MemoryStore, ToolAgent

memory = MemoryStore()
agent = ToolAgent(memory=memory)

# 执行任务，记忆自动记录、评分、遗忘
result = agent.run(task)
```

## 当前状态

T0 验证完成：记忆写入 → 多目标检索 → 低价值废弃的完整闭环。

六组基准对照已就绪，可复现实验。

[查看技术详情 →](https://github.com/Halucinaut/UMI-TIDE)
