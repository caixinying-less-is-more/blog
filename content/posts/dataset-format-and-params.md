---
title: "数据集格式和参数：读懂 info.json 与 action 归一化"
date: 2026-08-12T16:10:00+08:00
draft: false
tags: ["LeRobot", "数据采集", "动作空间", "So-101", "ACT"]
categories: ["机器人"]
description: "以 SO-ARM101 抓红方块数据集为例，拆解 info.json 的每一项参数，讲清 Episode/Frame 的区别、数据组织方式与 action 归一化格式"
---

第一次用 LeRobot 训练机械臂策略时，很多人会困惑：训练数据到底是什么格式？一个 episode 算一个样本吗？action 存的是角度变化量还是绝对位置？这篇文章以 SO-ARM101 机械臂"抓红方块"数据集为例，从 `info.json` 出发把这些基础问题一次讲清楚。

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

## info.json：数据集的"身份证"

在服务器上跑：

```bash
cat /gpfs1/scratch/xinyincai3/data_collection_1/grab_red_cube/meta/info.json
```

`info.json` 是数据集自动生成的描述文件，告诉 LeRobot 这个数据集长什么样、有哪些字段、怎么找到文件。下面逐项拆解。

### 基本信息

| 字段 | 值 | 含义 |
|------|-----|------|
| `codebase_version` | v3.0 | LeRobot 数据集格式版本号。v3.0 是当前版本，不同版本之间结构可能不兼容 |
| `robot_type` | so_follower | 录制数据用的机器人类型。so_follower = SO-ARM 的从臂（被操控的那只） |
| `total_episodes` | 77 | 总共录制了 77 次完整的操作 |
| `total_frames` | 29366 | 所有 episode 加起来共 29366 帧（每帧是一个训练样本） |
| `total_tasks` | 1 | 只有 1 个任务（抓红方块） |
| `fps` | 30 | 录制频率，每秒 30 帧 |

注意看：77 个 episode 对应 29366 帧，平均每个 episode 约 380 帧（≈12.7 秒），和上面"15 秒 ≈ 450 帧"的估算对得上。

### 数据组织

| 字段 | 值 | 含义 |
|------|-----|------|
| `chunks_size` | 1000 | 每 1000 帧切分成一个 chunk（数据块）。29366 帧会被分成 30 个 chunk |
| `data_path` | `data/chunk-{chunk_index:03d}/file-{file_index:03d}.parquet` | parquet 文件的存放路径模板。`{chunk_index:03d}` 表示补零三位数，如 chunk-000、chunk-001 |
| `video_path` | `videos/{video_key}/chunk-.../file-...mp4` | 视频文件的存放路径模板。`{video_key}` 会被替换成摄像头名，如 observation.images.front |

### 数据大小

| 字段 | 值 | 含义 |
|------|-----|------|
| `data_files_size_in_mb` | 100 | 所有 parquet 文件加起来 100MB（关节角度 + 动作等数值数据） |
| `video_files_size_in_mb` | 200 | 所有 mp4 视频文件加起来 200MB（摄像头画面） |

数值数据（parquet）只占三分之一，视频才是大头——这也解释了为什么采集 10 万帧数据动辄几十 GB：视频按原始画面存，压缩率低。

### 数据划分（splits）

| 字段 | 值 | 含义 |
|------|-----|------|
| `splits` | `{"train": "0:77"}` | 训练集 = 第 0 到第 76 个 episode（全部 77 个都用于训练，没有留验证集） |

### 特征定义（features）——核心部分

定义了每帧数据里有哪些字段：

| 字段名 | 类型 | 维度 | 含义 |
|--------|------|------|------|
| `action` | float32 | [6] | 模型要预测/执行的动作。6 个关节的目标位置 |
| `observation.state` | float32 | [6] | 当前机械臂的实际关节角度。模型输入（"我现在在哪"） |
| `observation.images.front` | video | [480, 640, 3] | 摄像头画面。480 行 × 640 列 × 3 色通道（RGB） |

`action` 和 `observation.state` 的 6 个关节（两个完全一样）：

| 序号 | 名称 | 含义 |
|------|------|------|
| 1 | `shoulder_pan.pos` | 肩部左右旋转 |
| 2 | `shoulder_lift.pos` | 肩部上下抬放 |
| 3 | `elbow_flex.pos` | 肘部弯曲 |
| 4 | `wrist_flex.pos` | 腕部弯曲 |
| 5 | `wrist_roll.pos` | 腕部旋转 |
| 6 | `gripper.pos` | 夹爪开合 |

### 索引字段（LeRobot 内部用，你不需要管）

| 字段名 | 类型 | 含义 |
|--------|------|------|
| `timestamp` | float32 | 这帧在 episode 内的时间戳（秒），从 0.000 开始，每帧加 0.033（1/30fps） |
| `frame_index` | int64 | 这帧在整个数据集里的全局序号（0~29365） |
| `episode_index` | int64 | 这帧属于第几个 episode（0~76） |
| `index` | int64 | 这帧的数据库内部索引（通常跟 frame_index 一样） |
| `task_index` | int64 | 这帧对应的任务编号（你只有 1 个任务，所以全是 0） |

---

## action 不是变化量，而是归一化后的绝对目标位置

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
- `info.json` 是数据集的"身份证"：基本信息、数据组织、划分、特征定义都在这一个文件里
- action 是**归一化后的绝对目标位置**：身体关节 [-100, +100]，夹爪 [0, 100]
- 推理时 LeRobot 会把归一化值反归一化回原始编码器值，写入舵机 `Goal_Position`
- 数据里视频占大头（200MB），数值数据很小（100MB）

*最后更新：2026-08-12*
