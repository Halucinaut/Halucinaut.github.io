---
title: "从 PRD 到 Benchmark：重新定义产品验收"
date: 2026-04-30
description: "一份面向产品与研发协作的演讲稿：把 PRD 从意图描述推进到可执行、可度量、可复核的 Benchmark。"
summary: "面向产品与研发协作的演讲：用 Benchmark 取代松散 PRD，把 AI 产品验收变成可执行、可度量、可复核的质量系统。"
tags: ["LLM", "Evaluation", "Product", "Benchmark"]
cover:
    image: "/slides/prd-benchmark/slide-01.jpg"
    alt: "从 PRD 到 Benchmark"
---

这份演讲是 [LLM as a Judge 技术报告](/posts/llm-as-a-judge/) 的产品化延伸。技术报告讨论 judge 的研究共识、失效模式与工程边界；这份材料面向产品验收，核心问题是：当 AI 产品的输出天然具有多样性时，传统 PRD 只描述意图，无法承担上线前的质量判定。

我的判断是：AI 产品验收需要从“需求是否实现”转向“行为分布是否可测”。Benchmark 的价值在于把目标、约束、失败模式、终止条件和人工抽检规则显式化，让产品、研发、测试和业务方围绕同一套可复核证据协作。

{{< rawhtml >}}
<div class="deck-actions">
  <a class="deck-download" href="/files/from-prd-to-benchmark.pptx">下载 PPT</a>
  <a class="deck-secondary" href="/posts/llm-as-a-judge/">阅读技术报告</a>
</div>

<section class="slide-reader" aria-label="从 PRD 到 Benchmark 幻灯片">
  <figure class="slide-frame" id="slide-01">
    <img loading="eager" src="/slides/prd-benchmark/slide-01.jpg" alt="Build 正经的 AI，当 Benchmark 取代 PRD，AI 产品该怎么定义">
    <figcaption>01 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-02">
    <img loading="lazy" src="/slides/prd-benchmark/slide-02.jpg" alt="传统 PRD 写的是意图，AI 产品暴露的是行为分布">
    <figcaption>02 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-03">
    <img loading="lazy" src="/slides/prd-benchmark/slide-03.jpg" alt="产品验收从功能检查转向行为测量">
    <figcaption>03 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-04">
    <img loading="lazy" src="/slides/prd-benchmark/slide-04.jpg" alt="Benchmark 是 AI 产品验收的共同语言">
    <figcaption>04 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-05">
    <img loading="lazy" src="/slides/prd-benchmark/slide-05.jpg" alt="从需求描述到可执行评测">
    <figcaption>05 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-06">
    <img loading="lazy" src="/slides/prd-benchmark/slide-06.jpg" alt="Benchmark 需要覆盖输入、约束、目标和失败模式">
    <figcaption>06 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-07">
    <img loading="lazy" src="/slides/prd-benchmark/slide-07.jpg" alt="从 PRD 到 Benchmark 的转换框架">
    <figcaption>07 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-08">
    <img loading="lazy" src="/slides/prd-benchmark/slide-08.jpg" alt="终止条件是 Agent 编排的硬门槛">
    <figcaption>08 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-09">
    <img loading="lazy" src="/slides/prd-benchmark/slide-09.jpg" alt="没有终止条件，循环只是更贵的随机游走">
    <figcaption>09 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-10">
    <img loading="lazy" src="/slides/prd-benchmark/slide-10.jpg" alt="评测协议决定产品验收的稳定性">
    <figcaption>10 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-11">
    <img loading="lazy" src="/slides/prd-benchmark/slide-11.jpg" alt="LLM as a Judge 的作用与边界">
    <figcaption>11 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-12">
    <img loading="lazy" src="/slides/prd-benchmark/slide-12.jpg" alt="评测需要拆分硬约束和软偏好">
    <figcaption>12 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-13">
    <img loading="lazy" src="/slides/prd-benchmark/slide-13.jpg" alt="评测链路需要事实、证据、质量和人工抽检">
    <figcaption>13 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-14">
    <img loading="lazy" src="/slides/prd-benchmark/slide-14.jpg" alt="位置偏差，顺序会改写 verdict">
    <figcaption>14 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-15">
    <img loading="lazy" src="/slides/prd-benchmark/slide-15.jpg" alt="生产基线需要强制顺序翻转">
    <figcaption>15 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-16">
    <img loading="lazy" src="/slides/prd-benchmark/slide-16.jpg" alt="Benchmark 要定义可接受失败和不可接受失败">
    <figcaption>16 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-17">
    <img loading="lazy" src="/slides/prd-benchmark/slide-17.jpg" alt="产品验收需要覆盖 Corner Cases">
    <figcaption>17 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-18">
    <img loading="lazy" src="/slides/prd-benchmark/slide-18.jpg" alt="从单点样例到行为分布">
    <figcaption>18 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-19">
    <img loading="lazy" src="/slides/prd-benchmark/slide-19.jpg" alt="验收指标需要服务真实产品目标">
    <figcaption>19 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-20">
    <img loading="lazy" src="/slides/prd-benchmark/slide-20.jpg" alt="从 Benchmark 到持续改进闭环">
    <figcaption>20 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-21">
    <img loading="lazy" src="/slides/prd-benchmark/slide-21.jpg" alt="把 Benchmark 接入上线流程">
    <figcaption>21 / 22</figcaption>
  </figure>
  <figure class="slide-frame" id="slide-22">
    <img loading="lazy" src="/slides/prd-benchmark/slide-22.jpg" alt="正经的 AI，需要可测量的质量系统">
    <figcaption>22 / 22</figcaption>
  </figure>
</section>
{{< /rawhtml >}}
