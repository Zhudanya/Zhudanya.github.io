---
title: "Harness Engineering 实践：让 AI Agent 自主验证游戏服务器"
date: 2026-03-24 20:00:00 +0800
categories: [AI, Engineering]
tags: [harness-engineering, ai-agent, game-dev, go]
---

当 AI Agent 开始写代码，工程师的工作从"写正确的代码"变成"构建让 Agent 能可靠写出正确代码的环境"。这就是 Harness Engineering。

## 问题

Agent 改完代码后说"我改完了"，但对不对？有没有改坏别的？全靠人手动验证——启动服务器、打开客户端、登录、操作一遍。每轮 5-10 分钟，来回 3-5 轮。

## 解决方案

我在一个 15 个微服务的 Go 游戏服务器项目上，搭建了完整的 Harness Engineering 环境。

### 三个阶段

**阶段一：地基** — 把项目知识文档化，让 Agent 看得到。CLAUDE.md、架构文档、执行计划。

**阶段二：约束机械化** — 9 条 Rules 约束 Agent 行为，4 个 Git Hooks 机械化执行，自演进机制让 Agent 犯过的错不再犯。

**阶段三：让应用对 Agent 可见** — Agent 能自己启动服务集群、跑 Robot 测试（真实 RPC 链路）、用 traceId 追踪请求、自主复现 Bug。

### 关键能力

- **Agent 自启集群**：一条命令启动 15 个微服务
- **Robot 真实 RPC 测试**：走完整的 客户端→gateway→proxy→logic→scene 链路
- **traceId 跨服务追踪**：一条 grep 串联一个请求在所有服务的日志
- **自主复现 Bug**：输入 bug 描述 → Agent 自动复现 → 定位到代码行号

### 效果

| 指标 | 改之前 | 改之后 |
|------|--------|--------|
| 验证一次改动 | 人工 5-10 分钟 | Agent 自动 30 秒 |
| 修 bug 来回轮数 | 3-5 轮 | 1-2 轮 |
| 定位问题 | 翻几千行日志 | traceId 一条 grep |

### 核心教训

1. **命令模板比自然语言可靠**：`/reproduce-bug` 命令强制 Agent 按"跑测试→看日志→读代码"的顺序走，自然语言没有这个约束，Agent 会猜测式调试。

2. **Harness 不是让 Agent 更聪明，是让它犯错空间更小**：Rules 减少能犯的错，Commands 减少能跳过的步骤，Hooks 减少能漏掉的检查。

3. **出错就改 Harness**：每次 Agent 犯错都是改进机会。修完 bug 后更新 Rules，让错误不再发生。

## 开源

完整配置已开源：[game-harness-engineering](https://github.com/Zhudanya/game-harness-engineering)
