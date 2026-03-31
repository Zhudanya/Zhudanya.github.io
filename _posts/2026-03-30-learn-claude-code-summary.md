---
title: "learn-claude-code 学习笔记（完结）：12 课全景总结"
date: 2026-03-30 21:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, summary]
---

[learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列 12 课学完了。这篇做一个全景总结——每课解决了什么问题、怎么解决的、加了什么机制。

## 核心论点

```
Agent = Model（大脑）+ Harness（身体 + 环境）
```

模型提供智能，Harness 提供表达智能的空间。12 课教的都是 Harness——循环从未改变。

## 12 课总览

| 课程 | 问题 | 解决方案 | 核心机制 | 工具数 |
|------|------|---------|---------|--------|
| **s01** Agent Loop | 模型碰不到真实世界 | while 循环 + bash 工具 | `stop_reason` 控制退出 | 1 |
| **s02** Tool Use | bash 干什么都不精准（截断、转义、安全） | 专用工具 + dispatch map | `TOOL_HANDLERS` 字典路由 + `safe_path` 沙箱 | 4 |
| **s03** TodoWrite | 复杂任务做着做着就跑偏 | TodoManager + nag 提醒 | 只允许一个 in_progress + 3 轮不更新就催 | 5 |
| **s04** Subagent | 子任务的过程污染父上下文 | 子 Agent + 独立 messages | `sub_messages=[]` 全新上下文，只返回摘要 | 5 |
| **s05** Skill Loading | 领域知识全塞 system prompt 太浪费 | 两层注入：目录 + 按需加载 | system prompt 放摘要，tool_result 放全文 | 5 |
| **s06** Context Compact | 上下文窗口有物理上限 | 三层压缩 | micro（占位符）+ auto（LLM 摘要）+ manual（模型主动） | 5 |
| **s07** Task System | todo 在内存里，压缩后丢失，无依赖关系 | 磁盘持久化的任务图（DAG） | `blockedBy/blocks` 依赖 + 完成时自动解锁 | 8 |
| **s08** Background Tasks | 慢命令卡住整个循环 | 后台线程 + 通知队列 | goroutine/thread 跑命令，drain 通知注入 messages | 6 |
| **s09** Agent Teams | 子 Agent 是一次性的，不能通信 | 持久队友 + JSONL 邮箱 | config.json 团队名册 + inbox/*.jsonl 通信 | 9 |
| **s10** Team Protocols | 队友之间缺少结构化协调 | 关机握手 + 计划审批 | request_id 关联 + `pending→approved/rejected` FSM | 12 |
| **s11** Autonomous Agents | 领导逐个分配任务，成为瓶颈 | WORK + IDLE 两阶段循环 | 空闲时每 5 秒轮询收件箱和任务看板，自动认领 | 14 |
| **s12** Worktree Isolation | 共享目录导致文件冲突 | git worktree 目录隔离 | 任务管"做什么"，worktree 管"在哪做"，task_id 绑定 | 16 |

## 四个阶段

### 第一阶段：基础能力（s01-s02）

| | 解决什么 | 加了什么 |
|---|---|---|
| s01 | 模型只能输出文字，不能执行 | while 循环 + bash |
| s02 | bash 什么都能干但干不好 | 专用工具 + 路径沙箱 + dispatch map |

从零到一：一个能读写文件、跑命令的 Agent。

### 第二阶段：智能管理（s03-s06）

| | 解决什么 | 加了什么 |
|---|---|---|
| s03 | 注意力稀释，忘记计划 | TodoManager（刷新到最近位置）+ nag 提醒 |
| s04 | 上下文被噪声污染 | 子 Agent（独立 messages，只返回摘要）|
| s05 | 知识前置塞入太浪费 | 两层注入（目录 + 按需加载）|
| s06 | 上下文物理上限 | 三层压缩（占位符 / LLM 摘要 / 手动触发）|

四课围绕**上下文质量**做文章：s03 对抗注意力稀释，s04 对抗噪声污染，s05 对抗无效占用，s06 对抗物理上限。

### 第三阶段：持久化（s07-s08）

| | 解决什么 | 加了什么 |
|---|---|---|
| s07 | 任务状态在压缩后丢失，无依赖关系 | 磁盘 JSON 文件 + blockedBy/blocks DAG |
| s08 | 慢命令阻塞整个循环 | 后台线程/goroutine + drain 通知队列 |

状态从内存走向磁盘，执行从单线程走向并发。

### 第四阶段：团队协作（s09-s12）

| | 解决什么 | 加了什么 |
|---|---|---|
| s09 | 子 Agent 是一次性的，不能通信 | 持久队友（config.json）+ JSONL 邮箱 |
| s10 | 缺少结构化协调 | 关机握手 + 计划审批（request_id + FSM）|
| s11 | 领导逐个分配成为瓶颈 | WORK+IDLE 循环 + 自动扫描看板认领 |
| s12 | 共享目录文件冲突 | git worktree 目录隔离 + 事件流 |

从单兵到团队：身份、通信、协议、自组织、隔离执行。

## 贯穿始终的设计模式

| 模式 | 出现在哪 | 做什么 |
|------|---------|--------|
| **dispatch map** | s02 起所有课 | 工具名 → 执行函数，循环永不改 |
| **drain 队列** | s08 通知、s09 邮箱、s11 轮询 | 读取并清空，保证不重复消费 |
| **文件持久化** | s07 任务、s09 团队、s12 worktree | 状态独立于对话，压缩和重启后存活 |
| **注入 messages** | s03 nag、s05 skill、s08 通知、s09 收件箱 | 把外部信息塞到模型能看到的位置 |
| **安全上限** | s04 子 Agent 30 轮、s09 队友 50 轮、s11 IDLE 60 秒 | 防止失控的后台循环 |
| **request_id 关联** | s10 关机、s10 计划审批 | 把请求和响应对上号 |
| **状态机** | s03 todo、s07 任务、s10 协议、s12 worktree | pending → in_progress → completed |

## 循环从未改变

从 s01 到 s12，核心循环始终是：

```python
while True:
    response = client.messages.create(model, system, messages, tools)
    messages.append({"role": "assistant", "content": response.content})
    if response.stop_reason != "tool_use":
        return
    results = execute_tools(response.content)
    messages.append({"role": "user", "content": results})
```

12 课加的所有机制——规划、子 Agent、技能、压缩、任务、后台、团队、协议、自治、隔离——都是在这个循环的**外围**。工具从 1 个增长到 16 个，线程从 1 个增长到 N 个，状态从内存扩展到磁盘——但循环本身，一行没改。

**这就是 Harness Engineering 的核心：不改模型，改环境。**

**全部 Go 实现**：[agents_go/](https://github.com/Zhudanya/learn-claude-code/tree/main/agents_go)

---

*本文是 learn-claude-code 系列学习笔记的完结篇。全部 12 篇文章基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库。Go 语言实现版本见同仓库 `agents_go/` 目录。*
