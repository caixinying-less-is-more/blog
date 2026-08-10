---
title: "AI 开发工具实践：CCR 代理与 Ollama 本地模型"
date: 2026-08-09T00:15:00+08:00
draft: false
tags: ["Claude Code", "CCR", "Ollama", "代理", "断路器"]
categories: ["AI与工具"]
description: "使用 Claude Code + CCR 代理 + Ollama 本地模型时遇到的认证问题和性能现象，涉及 API 代理架构、断路器模式、模型内存加载机制"
---

> 这篇文章记录我在使用 AI 开发工具（Claude Code、CCR、Ollama 等）时积累的知识点，会随着学习持续更新。

## Claude Code Router (CCR) 与 API 代理架构

CCR（Claude Code Router）是位于 Claude Code 客户端和 API 提供商之间的代理服务器。ccswitch 是 CCR 的一种实现，支持多 provider 路由——可以在不同 API 提供商（如 Anthropic 官方、Ollama 本地模型等）之间切换。

工作流程：

```
Claude Code → CCR 代理 (127.0.0.1:15721) → 实际 API 提供商
```

代理负责：路由请求、管理认证、负载均衡。

### 踩坑：apiKeyHelper 不兼容

Claude Code 支持 `apiKeyHelper` 机制（通过外部脚本动态获取 API Key），但 ccswitch 代理不支持这个机制。结果：认证信息无法正确传递到代理 → 401 错误。

### 断路器模式 (Circuit Breaker)

ccswitch 内置了断路器保护机制：

| 概念 | 说明 |
|---|---|
| 阈值 | 连续失败 8 次后跳闸 |
| 跳闸后行为 | 后续请求不再发送到目标服务，直接返回错误 |
| 目的 | 防止级联失败，保护系统稳定性 |

在本案例中，apiKeyHelper 不兼容导致 70 次 401 错误，远超阈值 8，断路器持续处于跳闸状态。即使后续修复了认证问题，也需要重置断路器才能恢复正常。

诊断思路：确认错误类型（401 = 认证失败）→ 检查认证机制兼容性 → 检查断路器状态 → 必要时重置断路器。

## Ollama 本地模型加载机制

使用 Ollama 运行本地模型（如 `qwen2.5:7b`）时，会遇到首次请求慢、后续变快的现象。

| 阶段 | 行为 | 耗时 |
|---|---|---|
| 冷启动 (cold-start) | 模型权重从磁盘加载到 RAM | 几秒到十几秒 |
| 热请求 | 模型已在内存，直接推理 | 毫秒级 |
| keep-alive 超时 | 5 分钟无请求后自动卸载模型 | — |

内存占用：7B 模型约 5-6GB RAM。

开发调试时的注意事项：

- 如果两次请求间隔超过 5 分钟，每次都要等冷启动
- 可以通过配置调整 keep-alive 时间避免频繁卸载
- 确保机器有足够 RAM（7B 模型至少预留 8GB）

---
*最后更新：2026-08-09*
