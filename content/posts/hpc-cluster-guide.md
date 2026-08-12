---
title: "HPC 集群使用指南：从登录到提交 GPU 训练任务"
date: 2026-08-10T23:26:57+08:00
draft: false
tags: ["HPC", "Slurm", "LeRobot", "GPU", "conda"]
categories: ["AI与工具"]
description: "0 基础在城市大学 HPC 集群上从零配置环境、上传数据集、提交 GPU 训练任务的完整指南"
---

适合谁读：需要用学校 HPC 集群跑深度学习训练，但之前没有 Slurm / Linux 集群经验的同学。以香港城市大学 burgundy 集群 + LeRobot 训练为例，从头到尾走一遍完整流程。

---

## 什么是 HPC 集群？

HPC（高性能计算）集群 = 很多台服务器通过网络连在一起。你通过 SSH 登录到的 `hpclogin02` 只是**登录节点**（前台），它负责让你提交任务，真正干活的是后面的几百台**计算节点**。

打个比方：

- 登录节点 = 餐厅前台，你在这里点菜（提交任务）
- 计算节点 = 厨房，真正做菜（跑训练）的地方
- 登录节点不能跑训练（就像你不能在前台炒菜）

---

## 第 1 步：检查系统环境

登录集群后，先摸清楚这台机器上有什么：

```bash
# 1. 这是什么系统？有没有 GPU 调度？
which srun sbatch          # HPC 集群一般用 Slurm
which module               # 或者 Environment Modules
nvidia-smi                 # 登录节点可能没有 GPU，报错也没关系

# 2. Python / conda 环境
which python python3 conda
python3 --version
conda --version 2>/dev/null

# 3. 磁盘空间（HPC 一般 home 有 quota，还有 scratch 大盘）
df -h ~
ls /scratch /workspace /data 2>/dev/null   # 看看有没有共享大盘
```

关键点：HPC 集群通常不能直接在登录节点跑训练，要用 Slurm 提交作业。

---

## 第 2 步：读懂 Slurm 分区信息

运行 `sinfo` 会输出分区表格，每一列的含义如下：

### PARTITION（分区名）

一组同类计算节点的"池子"名字：

| 值 | 含义 |
|------|------|
| `batch*` | 通用 CPU 节点。末尾的 `*` 表示默认分区 |
| `tiny` | 小型节点（CPU 核数少） |
| `gpu_a100` | 装 NVIDIA A100 GPU 的节点 |
| `gpu_v100s` | 装 NVIDIA V100 GPU 的节点 |
| `highmem_4t` | 4TB 超大内存节点 |
| `stingy` | 低优先级共享节点 |
| `coursework` | 课程作业节点 |

### AVAIL（是否开放）

| 值 | 含义 |
|------|------|
| `up` | 开放，可以提交任务 |
| `down` | 关闭，不接受任务 |

### TIMELIMIT（单任务最长运行时间）

格式是 `天-小时:分:秒`，超过这个时间 Slurm 会自动杀掉你的任务：

| 值 | 含义 |
|------|------|
| `3-00:00:00` | 3 天 |
| `5-00:00:00` | 5 天 |
| `10-00:00:00` | 10 天 |
| `infinite` | 无限制 |

### NODES + STATE（节点数量与状态）

同一个分区会出现多行，因为一个分区里的节点状态不同，Slurm 按状态分组统计。以 `batch*` 为例，可能出现 3 行：

```text
batch*  up  3-00:00:00  1    drain*   ← 有 1 台节点在维护中
batch*  up  3-00:00:00  1    down*    ← 有 1 台节点宕机了
batch*  up  3-00:00:00  110  alloc    ← 有 110 台节点正在被人使用
```

说明 batch 分区总共有 1+1+110 = 112 台节点。

STATE 各状态含义：

| 值 | 含义 | 能不能提交任务 |
|------|------|------|
| `idle` | 完全空闲 | 最容易抢到资源 |
| `mix` | 部分资源被占用，还有剩余 | 可以提交 |
| `alloc` | 资源全部分配出去了 | 可能需要排队 |
| `drain*` | 维护中，正在清退已有任务 | 不能提交 |
| `down*` | 宕机了 | 不能提交 |

`*` 号表示节点被标记了某种异常状态（不能接新任务）。

比如看到 GPU 分区的状态：

```text
gpu_a100   up  5-00:00:00  6  mix  ← 6 台 A100 节点，都是 mix 状态，有空闲 GPU
gpu_v100s  up  5-00:00:00  6  mix  ← 6 台 V100 节点，也是 mix 状态
```

这说明现在就能提交 GPU 训练任务。

### GROUPS（谁有权使用）

| 值 | 含义 |
|------|------|
| `all` | 所有用户都能用 |

---

## 第 3 步：查看 GPU 分区详情

想看每台 GPU 机器具体有多少卡、多少核、多少内存：

```bash
sinfo -p gpu_a100,gpu_v100s -N -o "%N %G %c %m %l %t"
```

参数拆解：

- `-p gpu_a100,gpu_v100s` = 只看这两个分区（逗号分隔）
- `-N` = 按节点（机器）显示，每台一行
- `-o "..."` = 自定义格式：`%N` 节点名 | `%G` 每节点 GPU 数 | `%c` CPU 核数 | `%m` 内存大小 | `%l` 最长运行时间 | `%t` 状态

运行后你会看到类似这样的输出：

```text
NODELIST      GRES         CPUS  MEMORY     STATE
gpu-a100-07   gpu:a100:4   32    1024000    alloc
gpu-a100-08   gpu:a100:4   32    1024000    mix
gpu-a100-09   gpu:a100:4   32    1024000    mix
gpu-a100-10   gpu:a100:4   32    1024000    mix
gpu-a100-11   gpu:a100:4   32    1024000    mix
gpu-a100-12   gpu:a100:4   32    1024000    mix
```

怎么读这张表：

- `NODELIST` = 节点名，`gpu-a100-07` 就是第 7 号 A100 节点
- `GRES` = Generic Resource，`gpu:a100:4` 表示每台节点装了 4 张 A100
- `CPUS` = 每节点 32 个 CPU 核
- `MEMORY` = 每节点 1TB 内存（单位是 MB，1024000 MB ≈ 1TB）
- `STATE` = 节点当前状态

这个例子里有 **6 台 A100 节点，每台 4 张卡，总共 24 张 A100**。其中：

- 1 台 `alloc`（4 张卡全被占满，没空位）
- 5 台 `mix`（部分卡在用，还有空闲 GPU 可以抢）

所以现在就能提交 GPU 训练任务——`mix` 状态的节点上还有卡。这也是第 2 步里 STATE 那张表的实际用法：看到 `mix` 就知道"有空闲资源"。

同时搜索 module 里的 Python 和 conda：

```bash
module avail 2>&1 | grep -i python
module avail 2>&1 | grep -i conda
module avail 2>&1 | grep -i miniforge
```

`module avail` 输出在错误通道上，所以要用 `2>&1` 合并后才能 grep。

---

## 第 4 步：加载 conda

找到模块名后（比如 `Miniconda3/24.7`），加载它：

```bash
module load Miniconda3/24.7
```

`module load` 相当于"激活这个软件"，加载后系统会把 Miniconda3 的路径加到 PATH 里，`conda` 命令就能用了。

验证：

```bash
conda --version
```

让 conda 每次登录自动加载（HPC 每次重新登录 module 都会重置）：

```bash
echo 'module load Miniconda3/24.7' >> ~/.bashrc
```

- `~/.bashrc` = 每次登录终端时自动执行的文件（类似 macOS 的开机启动项）
- `>>` = 追加到文件末尾（注意：两个 `>` 是"追加"，一个 `>` 是"覆盖"，千万别打成一个）
- 单引号 `'` 包起来防止内容被提前解析

---

## 第 5 步：创建 conda 虚拟环境

```bash
conda create -n lerobot python=3.10 -y
conda activate lerobot
python --version
```

- `conda create -n lerobot` = 创建名为 lerobot 的环境
- `python=3.10` = 指定 Python 版本（LeRobot 要求 3.10）
- `-y` = 自动确认
- `conda activate lerobot` = 激活环境，激活后提示符会出现 `(lerobot)`

---

## 第 6 步：克隆 LeRobot 并安装依赖

```bash
# 下载源码
cd ~ && git clone https://github.com/Seeed-Projects/lerobot.git

# 安装 PyTorch
pip install torch torchvision

# 安装 lerobot（可编辑模式 + feetech 电机驱动）
cd ~/lerobot && pip install -e ".[feetech]"
```

参数解释：

- `pip install -e .` = editable 模式，以后改代码不用重新安装
- `.[feetech]` = 安装当前目录的项目，额外带上 `feetech` 可选依赖组（方括号在 shell 里有特殊含义，加引号防止误解）

---

## 第 7 步：上传数据集到服务器

先在 Mac 本地确认数据集还在：

```bash
ls -lh /private/tmp/grab_red_cube.tar.gz
```

然后从 Mac 终端上传到服务器：

```bash
scp /private/tmp/grab_red_cube.tar.gz \
  xinyincai3@burgundy.hpc.cityu.edu.hk:/gpfs1/scratch/xinyincai3/
```

- `scp` = secure copy，通过 SSH 加密通道复制文件
- 上传速度取决于网络，可能需要几分钟，等它跑到 100%

上传完成后切回服务器终端，确认文件到了：

```bash
ls -lh /gpfs1/scratch/xinyincai3/grab_red_cube.tar.gz
```

---

## 第 8 步：解压数据集并清理

在服务器上执行：

```bash
# 进入 scratch 目录
cd /gpfs1/scratch/xinyincai3

# 解压 tar 包
tar -xzf grab_red_cube.tar.gz

# 查看解压出来的目录结构
ls -la

# 删除 macOS 影子文件（重要！）
find . -name "._*" -delete
find . -name ".DS_Store" -delete
```

Mac 打的 tar 包里会带 `._` 开头的隐藏文件（AppleDouble 影子文件），这些文件会让 LeRobot 报错，必须删掉。

`tar` 参数：`-x` 解压 | `-z` 用 gzip 解压 | `-f` 指定文件名。

---

## 第 9 步：提交训练任务

HPC 集群不能直接在登录节点跑训练，要写一个 Slurm 脚本提交到计算节点。

### 创建训练脚本

```bash
cat > /gpfs1/scratch/xinyincai3/train_act.slurm << 'EOF'
#!/bin/bash
#SBATCH --job-name=act_train
#SBATCH --partition=gpu_a100
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=64G
#SBATCH --time=05:00:00
#SBATCH --output=/gpfs1/scratch/xinyincai3/logs/act_train_%j.log

module load Miniconda3/24.7
conda activate lerobot

export LEROBOT_HOME=/gpfs1/scratch/xinyincai3
export WANDB_MODE=online

lerobot-train \
    dataset.repo_id=data_collection_1/grab_red_cube \
    policy=act \
    output_dir=/gpfs1/scratch/xinyincai3/outputs/train/act_so101 \
    policy.chunk_size=100 \
    policy.n_action_steps=100 \
    batch_size=8 \
    steps=200000 \
    save_freq=20000 \
    eval_freq=20000 \
    eval.n_episodes=5 \
    eval.batch_size=5 \
    wandb.enable=true \
    wandb.project=act_so101_grab_cube \
    wandb.entity=xinying_cai_123
EOF
```

`cat > ... << 'EOF'` 的意思是一直读到遇到 `EOF` 这行为止，中间所有内容写入文件。

### Slurm 指令逐行解释

| 指令 | 含义 |
|------|------|
| `--job-name=act_train` | 任务名 |
| `--partition=gpu_a100` | 用 A100 GPU 分区 |
| `--gres=gpu:1` | 申请 1 张 GPU |
| `--cpus-per-task=8` | 申请 8 个 CPU 核 |
| `--mem=64G` | 申请 64GB 内存 |
| `--time=05:00:00` | 最多跑 5 小时（训练约 3 小时，留余量） |
| `--output=...%j.log` | 日志输出路径，`%j` 会被替换成任务 ID |

计算节点是全新环境，不会继承登录节点的设置，所以脚本里要重新 `module load` 和 `conda activate`。

### 提交任务

```bash
# 创建日志目录
mkdir -p /gpfs1/scratch/xinyincai3/logs

# 提交
sbatch /gpfs1/scratch/xinyincai3/train_act.slurm
```

提交成功会显示 `Submitted batch job 12345`（数字是任务 ID）。

### 常用监控命令

```bash
# 查看自己的任务状态
squeue -u $USER

# 查看任务详情
scontrol show job <任务ID>

# 取消任务
scancel <任务ID>

# 实时查看训练日志
tail -f /gpfs1/scratch/xinyincai3/logs/act_train_<任务ID>.log
```

### 查看谁在占着 GPU

想知道 A100 分区上所有人的排队和运行情况：

```bash
squeue -p gpu_a100 -o "%.10i %.9P %.8j %.8u %.2t %.10M %.6D %R" | head -30
```

- `-p gpu_a100` = 只看 A100 分区
- `-o "..."` = 自定义输出列：任务 ID、分区、任务名、用户名、状态、运行时间、节点数、原因
- `| head -30` = 只看前 30 条

这样你能看到是谁在用卡、排了多少人在你前面。

---

## 总结

整个流程其实就 9 步：检查环境 → 读懂分区 → 加载 conda → 建虚拟环境 → 装依赖 → 传数据 → 解压数据 → 写 Slurm 脚本 → 提交训练。

核心原则只有一条：**登录节点只用来提交任务，真正干活一定要提交到计算节点。**

踩坑提醒：
- Mac 打的 tar 包一定要删 `._` 文件
- `.bashrc` 里追加用 `>>` 不是 `>`
- `--time` 要留余量，超时会被自动杀掉
- 计算节点不继承登录节点的环境，脚本里要重新 load module

*最后更新：2026-08-12*
