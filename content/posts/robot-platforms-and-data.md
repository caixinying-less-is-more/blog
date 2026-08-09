---
title: "机器人平台与数据采集"
date: 2026-08-10T00:15:00+08:00
draft: false
tags: ["Mobile ALOHA", "UMI", "遥操作", "数据采集", "Co-training"]
categories: ["机器人"]
description: "具身智能的数据从哪来——Mobile ALOHA 与 UMI 两种遥操作采集范式，以及 Co-training 如何用少量数据激活新能力"
---

> 这篇文章记录我在学习机器人平台与数据采集方法时积累的知识点，会随着学习持续更新。

## Mobile ALOHA

Stanford 2024 年的工作，在固定双臂系统 ALOHA 基础上加了移动底盘，让机器人可以边走边干活——炒菜、开柜子、倒水、收拾桌面。

### 硬件系统

| 部件 | 配置 | 作用 |
|---|---|---|
| 移动底盘 | AgileX Tracer 差分驱动 | 全向移动，最高 1m/s |
| 双臂 | ViperX 300 × 2，各 6-DOF | 抓取和操作 |
| 相机 | 顶部 2 个 + 手腕 2 个 | 视觉观测输入 |
| 计算 | GPU 笔记本 | 实时推理策略 |

关键优势是便宜——总成本约 $32k，而商业移动操作平台通常 >$100k。

### 数据采集：Leader-Follower 遥操作

人穿戴 Leader 双臂（没有电机，只有编码器记录角度），机器人的 Follower 臂实时跟随。全程记录（相机图像, 动作）数据对。每个任务收集 50 条演示。

双臂遥操作可以采集到比 VR 手柄或键盘更自然、更精细的双手协调动作。

### 核心创新：Co-training

注意：ACT（Action Chunking with Transformers）是原始 ALOHA 论文（2023）提出的，不是 Mobile ALOHA 的原创。Mobile ALOHA 真正的贡献是 **Co-training 混合训练**。

问题很直接：移动操作任务复杂，50 条演示不够学好。但他们发现：

- 静态 ALOHA 数据：820 条（固定桌面双臂上采集，没有移动）
- 移动 ALOHA 数据：50 条/任务
- 混合在一起训练

效果惊人——平均成功率从约 15% 跳到约 85%。静态数据教会了策略"怎么抓东西"，移动数据教会了"怎么走到目标旁边"，两个能力叠加。

### ACT 的关键设计

ACT（Action Chunking with Transformers）的核心是 **Action Chunking**——一次预测未来 N 步动作，但只执行前几步，然后重新预测。这样每一步的微小预测误差不会逐步累积放大，解决了 delta 动作表示的漂移问题。

| | ACT | Diffusion Policy |
|---|---|---|
| 模型 | Transformer CVAE | 扩散模型 |
| 动作输出 | 一次输出 N 步（chunk） | 迭代去噪输出 |
| 多模态 | 不擅长（CVAE 偏单峰） | 擅长 |
| 推理速度 | 快（一次前向传播） | 慢（多步去噪） |

## UMI

UMI（Universal Manipulation Interface）也是 Stanford 2024 年初的工作，但走了完全不同的路线。

### 核心思路

不依赖机器人，用**手持夹爪**采集数据：

- 人手持带有相机的夹爪，直接去执行任务动作
- 全程记录（相机图像, 动作）数据对
- 通过 world-to-robot 推理，把手持数据转化为可部署的策略

策略使用 Diffusion Policy。

### 关键优势

任何人手持就能采集数据，不需要购买特定机器人。数据扩展性远高于 Leader-Follower 遥操作。

## UMI vs Mobile ALOHA

两者都是 2024 年初的 Stanford 工作，是**两条平行路线**解决不同的问题：

| | UMI | Mobile ALOHA |
|---|---|---|
| 解决什么问题 | 数据从哪来 | 怎么让机器人移动操作 |
| 数据采集 | 手持夹爪，无需机器人 | 必须用机器人 Leader-Follower |
| 策略 | Diffusion Policy | ACT |
| 核心创新 | 采集设备 + world-to-robot 推理 | Co-training 混合训练 |
| 数据扩展性 | 高——任何人手持就能采 | 低——必须买特定机器人 |
| 部署能力 | 固定臂操作 | 移动双臂全屋操作 |

两者后来有融合趋势——后续工作开始用 UMI 的采集方式获取大量数据，再用 co-training 的思路做场景适配。

---
*最后更新：2026-08-10*
