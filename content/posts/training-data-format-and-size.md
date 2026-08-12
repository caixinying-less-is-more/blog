---
title: "训练数据格式和大小：模型到底在学什么"
date: 2026-08-12T16:10:00+08:00
draft: false
tags: ["LeRobot", "数据采集", "动作空间", "So-101", "ACT"]
categories: ["机器人"]
description: "用 SO-ARM101 机械臂抓红方块为例，讲清模仿学习训练数据里 Episode/Frame 的区别、action 的归一化格式，以及数据占多大空间"
---

第一次用 LeRobot 训练机械臂策略时，很多人会困惑：训练数据到底是什么格式？一个 episode 算一个样本吗？action 存的是角度变化量还是绝对位置？这篇文章以 SO-ARM101 机械臂"抓红方块"为例，把这些基础问题一次讲清楚。

---

## Episode 和 Frame 的区别

| 概念 | 含义 | 举例 |
|------|------|------|
| **Episode** | 一次完整的操作（从开始到结束） | 你手动操控机械臂抓一次红方块 = 1 个 episode |
| **Frame** | 一个时间步（一帧数据） | 30fps 录制，一个 15 秒的 episode = 450 帧 |

训练数据的数量 **不等于 episodes 数量**。模型真正学习的样本是 **frame 数**——每个 frame 是一个"观察 → 动作"的配对（一张图片 + 关节角度 → 对应的动作），这才是模型学习的样本。

打个比方：

- Episode = 一道菜的完整菜谱
- Frame = 菜谱里的每一步操作
- 模型学的是每一步操作（frame），不是整道菜

---

## 来查你的实际数据

在服务器上跑：

```bash
cat /gpfs1/scratch/xinyincai3/data_collection_1/grab_red_cube/meta/info.json
```

- `cat` = 显示文件内容
- `info.json` = 数据集自动生成的描述文件，里面有 episodes 数量、帧数、fps 等信息

---

## Action 不是变化量，而是归一化后的绝对目标位置

### Action 到底存的是什么

从代码 `so_follower.py:51` 可以看到：

```python
norm_mode_body = MotorNormMode.DEGREES if config.use_degrees else MotorNormMode.RANGE_M100_100
```

SO-ARM101 默认用 `RANGE_M100_100`（夹爪除外），意思是 **5 个身体关节**（shoulder_pan 到 wrist_roll）归一化到 **[-100, +100]** 的范围：

```
归一化值 = ((原始编码器值 - 最小值) / (最大值 - 最小值)) × 200 - 100
```

| 归一化值 | 含义 |
|----------|------|
| -100 | 关节在运动范围的最负端 |
| 0 | 关节在正中间 |
| +100 | 关节在运动范围的最正端 |

### 夹爪（gripper）

从代码 `so_follower.py:60` 可以看到，夹爪用不同的模式：

```python
"gripper": Motor(6, "sts3215", MotorNormMode.RANGE_0_100)
```

归一化到 **[0, 100]**：

| 归一化值 | 含义 |
|----------|------|
| 0 | 完全打开 |
| 100 | 完全闭合 |

### 推理时怎么用

从代码 `so_follower.py:221`：

```python
self.bus.sync_write("Goal_Position", goal_pos)
```

模型预测出 action（如 `[-23.5, 45.2, -10.0, 5.3, 0.0, 85.0]`），LeRobot 会把归一化值**反归一化**回原始编码器值（0-4095），然后直接写入舵机的 `Goal_Position` 寄存器——告诉舵机"去到这个位置"。

关键点：action 是**绝对目标位置**，不是"这次该转多少度"的变化量。模型要学的是"当前状态 → 下一步该把关节转到哪"。

---

## 数据大小

| 字段 | 值 | 含义 |
|------|-----|------|
| `data_files_size_in_mb` | 100 | 所有 parquet 文件加起来 100MB（关节角度等数值数据） |
| `video_files_size_in_mb` | 200 | 所有 mp4 视频文件加起来 200MB（摄像头画面） |

数值数据（parquet）只占三分之一，视频才是大头——这也解释了为什么采集 10 万帧数据动辄几十 GB：视频按原始画面存，压缩率低。

---

## observation.state 和 action 的关系

`observation.state`（当前关节角度）和 `action`（目标关节角度）用的是**完全一样**的归一化方式：

| 数据 | 归一化范围 | 含义 |
|------|------------|------|
| `observation.state[0:5]` | -100 ~ +100 | 5 个身体关节当前所在位置 |
| `observation.state[5]` | 0 ~ 100 | 夹爪当前开合程度 |
| `action[0:5]` | -100 ~ +100 | 5 个身体关节下一步要去的位置 |
| `action[5]` | 0 ~ 100 | 夹爪下一步要去的开合程度 |

状态和动作**格式完全一样**，区别只是语义：一个是"我现在在哪"，一个是"我下一步要去哪"。

---

## 总结

- 模型学习的样本单位是 **frame**（"观察 → 动作"配对），不是 episode
- action 是**归一化后的绝对目标位置**：身体关节 [-100, +100]，夹爪 [0, 100]
- 推理时 LeRobot 会把归一化值反归一化回原始编码器值，写入舵机 `Goal_Position`
- 数据里视频占大头，数值数据很小

*最后更新：2026-08-12*
