---
title: "LeRobot 数据集结构详解：info.json 与 Parquet 格式"
date: 2026-08-12T12:00:00+08:00
draft: false
tags: ["LeRobot", "数据采集", "机械臂", "Python"]
categories: ["机器人"]
description: "拆解 LeRobot 数据集的内部结构——info.json 每个字段的含义、Parquet 列式存储为什么比 CSV 更适合机器人数据、Episode 与 Frame 的区别"
---

适合谁读：正在用 LeRobot 采集或训练机器人数据，但不太清楚数据集内部结构的人。以 SO-ARM101 抓红方块的数据集为例，逐字段拆解。

---

## Episode 和 Frame 的区别

这是最容易混淆的概念。

| 概念 | 含义 | 举例 |
|------|------|------|
| Episode | 一次完整的操作（从开始到结束） | 手动操控机械臂抓一次红方块 = 1 个 episode |
| Frame | 一个时间步（一帧数据） | 30fps 录制，一个 15 秒的 episode = 450 帧 |

**训练数据的实际数量 ≠ episodes 数量**。训练数据的实际数量是 frame 数。每个 frame 是一个"观察→动作"的配对（一张图片 + 关节角度 → 对应的动作），这才是模型学习的样本。

打个比方：

- Episode = 一道菜的完整菜谱
- Frame = 菜谱里的每一步操作
- 模型学的是每一步操作（frame），不是整道菜

我的抓红方块数据集：77 个 episode，29366 帧，平均每个 episode 约 381 帧（约 12.7 秒）。

---

## 数据集目录结构

一个 LeRobot 数据集在磁盘上长这样：

```text
grab_red_cube/
├── meta/
│   └── info.json          ← 数据集的"身份证"
├── data/
│   └── chunk-000/
│       ├── file-000.parquet   ← 数值数据（关节角度、动作）
│       ├── file-001.parquet
│       └── ...
└── videos/
    └── observation.images.front/
        └── chunk-000/
            ├── file-000.mp4    ← 视频数据（摄像头画面）
            └── ...
```

两个文件夹分别存不同类型的数据：

| 文件夹 | 存什么 | 格式 | 大小 |
|--------|--------|------|------|
| `data/` | 关节角度、动作等数值 | Parquet | 100MB |
| `videos/` | 摄像头画面 | MP4（AV1 编码） | 200MB |

---

## 为什么用 Parquet 而不是 CSV？

Parquet 是一种高效的**列式存储**文件格式，专门用来存表格数据。

| 对比 | CSV | Parquet |
|------|-----|---------|
| 存储方式 | 行式（一行一行存） | 列式（一列一列存） |
| 文件大小 | 大 | 小（自带压缩，通常小 3-10 倍） |
| 读取速度 | 慢 | 快 |
| 数据类型 | 全是文本 | 保留精确类型（float32, int64...） |

如果用 CSV 存 29366 帧数值数据，大约要 300-500MB；用 Parquet 只要 100MB。

更关键的是：深度学习训练时要频繁读取特定列（比如只读 `action` 列），Parquet 的列式存储让这种读取非常快——直接跳到那一列的数据块，不用扫全表。

---

## 为什么图片单独存成视频？

如果存成原始图片（每帧一张 PNG），29366 帧 × 约 200KB/张 = 约 6GB。压成 AV1 视频后只要 200MB，压缩了 30 倍。

训练时 LeRobot 会按需从视频中解压特定帧，不需要把整个视频加载到内存。

---

## info.json 逐字段详解

`info.json` 是数据集的元数据文件，LeRobot 训练框架靠它知道数据集长什么样。

### 基本信息

| 字段 | 我的值 | 含义 |
|------|--------|------|
| `codebase_version` | `v3.0` | LeRobot 数据集格式版本号，不同版本结构可能不兼容 |
| `robot_type` | `so_follower` | 录制数据用的机器人类型（SO-ARM 的从臂） |
| `total_episodes` | `77` | 总共录制了 77 次操作 |
| `total_frames` | `29366` | 所有 episode 加起来共 29366 帧 |
| `total_tasks` | `1` | 只有 1 个任务（抓红方块） |
| `fps` | `30` | 录制频率，每秒 30 帧 |

### 数据组织

| 字段 | 含义 |
|------|------|
| `chunks_size` | 每 1000 帧切分成一个 chunk（数据块） |
| `data_path` | Parquet 文件的路径模板，`{chunk_index:03d}` 表示补零三位数（如 `chunk-000`） |
| `video_path` | 视频文件的路径模板，`{video_key}` 替换为摄像头名 |

 chunks_size
  数据里每帧都有 episode_index 字段（0~76）。LeRobot 加载数据时按 episode_index 分组，不按chunk
  Chunk 只是文件存储的分块方式，不影响数据的逻辑完整性。

  LeRobot 怎么保证 episode 完整性

  数据里每帧都有 episode_index 字段（0~76）。LeRobot 加载数据时按 episode_index 分组，不按 chunk
  分组：

  Chunk 0 (frame 0-999):
    ├─ frame 0-380:   episode 0 (完整) ✓
    ├─ frame 381-761: episode 1 (完整) ✓
    └─ frame 762-999: episode 2 的前半部分

  Chunk 1 (frame 1000-1999):
    ├─ frame 1000-1142: episode 2 的后半部分
    ├─ ...

  episode 2 的帧虽然跨了两个 chunk 文件，但 LeRobot
  加载时会自动把它们拼回来。训练时模型看到的是完整的 episode，感知不到 chunk 边界。

  为什么要分 chunk

  纯粹是文件管理的原因：

  - 不分 chunk：29366 帧存成 1 个 100MB 的 parquet 文件，加载慢
  - 分 chunk：每个文件约 3-4MB，可以并行加载，内存占用小

### 数据大小

| 字段 | 值 | 含义 |
|------|-----|------|
| `data_files_size_in_mb` | `100` | 所有 Parquet 文件总和 |
| `video_files_size_in_mb` | `200` |所有 MP4 视频文件总和 |

### 数据划分

| 字段 | 值 | 含义 |
|------|-----|------|
| `splits` | `{"train": "0:77"}` | 训练集 = 第 0 到第 76 个 episode（全部用于训练，没有验证集） |

---

## 核心特征定义

`features` 是 info.json 里最重要的部分，定义了每帧数据包含哪些字段。

### 训练相关（模型输入和输出）

| 字段名 | 类型 | 维度 | 角色 |
|--------|------|------|------|
| `action` | float32 | [6] | 模型要预测的动作（**训练目标**） |
| `observation.state` | float32 | [6] | 当前关节角度（**模型输入**） |
| `observation.images.front` | video | [480, 640, 3] | 摄像头画面（**模型输入**） |

`action` 和 `observation.state` 的 6 个关节完全一样：

| 序号 | 名称 | 含义 |
|------|------|------|
| 1 | `shoulder_pan.pos` | 肩部左右旋转 |
| 2 | `shoulder_lift.pos` | 肩部上下抬放 |
| 3 | `elbow_flex.pos` | 肘部弯曲 |
| 4 | `wrist_flex.pos` | 腕部弯曲 |
| 5 | `wrist_roll.pos` | 腕部旋转 |
| 6 | `gripper.pos` | 夹爪开合 |

模型做的事情就是：给定 `observation`（当前状态 + 摄像头画面），预测 `action`（下一步该怎么动）。

### 索引字段（LeRobot 内部管理用）

| 字段名 | 类型 | 含义 |
|--------|------|------|
| `timestamp` | float32 | 帧在 episode 内的时间戳（秒），从 0.000 开始，每帧加 0.033（1/30fps） |
| `frame_index` | int64 | 帧在整个数据集里的全局序号（0 到 29365） |
| `episode_index` | int64 | 帧属于第几个 episode（0 到 76） |
| `index` | int64 | 数据库内部索引（通常跟 frame_index 一样） |
| `task_index` | int64 | 任务编号（只有 1 个任务时全是 0） |

---


---



## 总结

LeRobot 数据集的设计思路：

- **数值数据**用 Parquet（列式存储，小而快）
- **图像数据**用 AV1 视频压缩（比原始图片小 30 倍）
- **info.json** 是入口文件，告诉训练框架数据集的结构
- **Episode** 是人类视角的"操作次数"，**Frame** 才是模型学习的样本量

*最后更新：2026-08-12*
