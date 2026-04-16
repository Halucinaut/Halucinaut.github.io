---
title: "Doppel"
date: 2026-04-12
weight: 4
description: "不是功能测试工具，而是体验验证系统。让 AI 以'第一次接触产品的真实用户'身份，在上线前暴露认知摩擦和流失风险。"
tags: ["Synthetic User", "Experiential Testing", "Sandbox Runtime", "UX Evaluation"]
cover:
    image: "images/doppel-cover.png"
    alt: "Doppel"
demo_url: ""
repo_url: "https://github.com/Halucinaut/Doppel"
summary: "Synthetic User Runtime。在可控 Sandbox 中运行 AI 模拟用户，完成指定任务并输出基于行为证据的体验反馈。"
layout: "single"
likes: 0
---

## 问题意识

AI 把"做出一个产品"的成本降得很低，却没有同步降低"判断这个产品是不是能被陌生人真正用起来"的成本。

今天的测试工具解决了不同层的问题：单元测试验证代码逻辑，集成测试验证系统协作，E2E 测试验证预定义路径是否可走通。但仍然缺失一个问题的答案：

> 第一次来的用户，会不会看不懂、找不到、点不下去、或者直接流失？

## 核心洞察

Doppel 是一个 **Synthetic User Runtime**。它让 AI 代理以"第一次接触产品的真实用户"身份，进入一个目标产品，在尽量少的背景信息下完成指定任务，并输出基于行为证据的体验反馈。

目标不是回答"这个功能是否正常工作"，而是回答：
- 一个陌生人第一次来到这里，能不能理解这个产品？
- 他会不会在关键路径上迷路、犹豫、点错、放弃？
- 哪些页面、文案、交互会制造认知摩擦？

## 四大原则

**冷启动优先** —— Agent 只拿到最少产品语义信息，验证产品本身是否自解释。

**视觉优先于 DOM** —— 对页面的理解以视觉结果为主，测的是"用户会如何理解页面"，而不是"页面结构允许什么"。

**Sandbox 优先于 Workflow** —— 必须在可控环境中运行，使测试可以隔离、重置、重复和比较。

**证据优先于观点** —— 报告中的结论必须绑定具体行为证据：第几步停留多久、在同一页面来回跳转几次、点击错误按钮几次后才找到主路径。

## 系统架构

Doppel 由 4 个核心子系统组成：

- **SandboxManager** —— 准备隔离环境
- **AgentRuntime** —— 执行体验过程
- **JudgeEngine** —— 把证据变成结论
- **ReportBuilder** —— 输出体验报告

CLI 只是入口，Sandbox 才是测试基础设施。
