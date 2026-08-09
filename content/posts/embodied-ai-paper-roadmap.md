---
title: "具身智能论文学习路线"
date: 2026-08-10T00:15:00+08:00
draft: false
tags: ["论文阅读", "具身智能", "Diffusion Policy", "VLA", "学习路线"]
categories: ["学术研究"]
description: "10 篇具身智能标志性论文的分层学习策略——两条技术路线、三层投入梯度，不需要每篇都复现"
---

> 这篇文章记录我在规划具身智能论文学习路线时积累的方法和判断，会随着学习进展持续更新。

## 10 篇论文全景

一个专业朋友推荐了 10 篇标志性研究成果：Diffusion Policy、UMI、Mobile ALOHA、RT-2、Open-X Embodiment、OpenVLA、RDT、GR00T、Pi0/0.5/0.6、Generalist Gen-0。

**结论：不需要每篇都复现。** 正确策略是分层投入。

## 两条技术路线

这 10 篇论文分属两条平行发展的路线：

| | 扩散策略路线 | VLM 大模型路线 |
|---|---|---|
| 核心思想 | 用扩散/流匹配模型表示动作分布 | 用预训练大语言模型做策略 |
| 演进线 | Diffusion Policy → RDT → Pi0 | RT-2 → Open-X → OpenVLA → GR00T |
| 优势 | 动作精度高，多模态好 | 语言理解强，泛化好 |
| 劣势 | 推理慢 | 动作精度低，模型大 |

另外 UMI / Mobile ALOHA 不属于策略路线，是解决"数据从哪来"的硬件/采集方案——没有数据，什么策略都白搭。

## 三层学习策略

### 第一层：了解思想

**Diffusion Policy、UMI、Mobile ALOHA、RT-2、Open-X**

看 intro + 核心方法图，花 30 分钟搞懂"解决了什么问题、用了什么思路"就够了。不需要看源码。它们是奠基者，后来的方法都在它们的基础上改进。

### 第二层：精读 + 跑代码

**OpenVLA、RDT**

读完整论文，跑通官方 demo，理解架构设计和训练细节。OpenVLA 是开源 VLA 里复现门槛最低的，RDT 是扩散策略大规模化的代表。

### 第三层：重点跟踪

**Pi0 系列、GR00T、Generalist Gen-0**

当前 SOTA 和未来方向。Pi0 的流匹配策略目前性能最强，GR00T 是 NVIDIA 生态的核心。但这些要么不开源、要么资源需求极高，现阶段跟踪进展就行，等基础扎实后再深入。

## 学习顺序建议

1. 先打基础：LQR/MPC、DDPM/DDIM、动作空间、最优控制
2. 精读 Diffusion Policy（唯一需要现在精读的论文）
3. 第一层其余 4 篇扫一遍 intro
4. 基础阶段完成后（1-2 个月），从 OpenVLA 或 Pi0 选一个深入
5. 不要急着跳到 Pi0 / GR00T——没有 Diffusion Policy 的基础，Pi0 的流匹配读不懂；没有 RT-2 的基础，OpenVLA 的 VLM 架构读不懂

---
*最后更新：2026-08-10*
