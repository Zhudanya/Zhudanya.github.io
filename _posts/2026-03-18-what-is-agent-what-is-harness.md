---
title: "搞清楚 Agent 和 Harness 到底是什么"
date: 2026-03-18 20:00:00 +0800
categories: [AI, Engineering]
tags: [ai-agent, harness-engineering, llm]
---

最近在啃 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 这个仓库，里面有一句话反复出现：

> 模型就是 Agent。代码是 Harness。

第一眼觉得挺对的。但越想越不对。

## Agent 到底是什么

项目给的定义：Agent 是一个经过训练的神经网络，学会了**感知环境、推理目标、采取行动**。

这个定义没毛病。回看历史上每一个被叫做 Agent 的东西：

- DQN 玩 Atari —— 一个 CNN，接收像素，输出操作
- AlphaStar 打星际 —— 一个 Transformer，接收游戏状态，输出动作序列
- 绝悟打王者 —— 训练强度一天等于人类 440 年，自己学会了全英雄池
- Claude Code 写代码 —— 一个 LLM，接收上下文，输出工具调用

共性很明显：**一个模型，在某个环境里，通过训练学会了自主行动。**

所以 Agent 是上位概念。LLM 只是其中一种——在语言和代码领域表现出 agency 的那种模型。DQN 是另一种，AlphaStar 又是另一种。

## Harness 到底是什么

Harness 是 Agent 执行任务所需的一切外部支撑：

```
Harness = Tools + Knowledge + Observation + Action + Permissions
```

| 组成 | 干嘛的 | 举例 |
|------|--------|------|
| Tools | 让 Agent 能动手 | bash、文件读写、浏览器 |
| Knowledge | 让 Agent 有领域知识 | 文档、API 规范、代码规范 |
| Observation | 让 Agent 能感知 | git diff、错误日志、测试结果 |
| Action | 让 Agent 能影响世界 | CLI 命令、API 调用 |
| Permissions | 给 Agent 画边界 | 沙箱、审批流、信任边界 |

## 它们什么关系

这是我一开始想错的地方。

最初的理解是："搭建 Harness 环境，让 Agent 在里面工作"——好像 Agent 和 Harness 是两个独立的东西，一个住进另一个。

但仔细想就发现不对。

一个 LLM 没有工具就不能行动。一个 DQN 没有游戏模拟器也不能行动。**任何模型脱离了执行环境，都只能输出信号，不能真正"做事"。** LLM 输出 token，DQN 输出 Q-values，AlphaStar 输出动作向量——都得靠外部环境翻译成真实行动。

所以 Harness 不是 Agent 的"外部环境"，**Harness 是 Agent 的组成部分**：

```
Agent（完整的智能体）
├── Model（大脑）—— 感知、推理、决策
└── Harness（身体 + 工作环境）—— 工具、知识、权限、执行
```

用人来类比：你不会说"大脑在身体里工作"——大脑和身体一起构成了一个人。同理，Model 不是"在 Harness 里工作"，它们一起构成了 Agent。

**智能在模型里，但 agency 是模型和 harness 共同实现的。**

## 那"模型就是 Agent"这句话错了吗

没完全错，但不精确。

这句话的本意是纠正一个行业偏差：太多人以为用框架、工作流、提示词链把 LLM 串起来就是在"构建 Agent"。不是的。智能不在那些代码里，智能在模型的权重里。这个纠偏是对的。

但反过来说"模型就是 Agent"也走过头了。一个外科医生被关在空房间里，没有手术刀、没有病人——他还是有外科知识，但他做不了手术。他的能力在大脑里，但他作为外科医生的完整 agency，需要大脑、双手、手术室共同实现。

更准确的说法：**模型是 Agent 的核心，但不是 Agent 的全部。**

## 落到实际工作上

我做游戏开发，日常用 Claude Code。拆开来看：

```
我的游戏开发 Agent = Claude（模型）
                   + Claude Code 的 Harness（循环、工具、任务系统、上下文压缩...）
                   + 我配的 Harness（CLAUDE.md、Skills、Commands、Rules）
```

Claude Code 提供的是通用编程能力的 Harness。我配的是游戏开发领域的专业 Harness。两层叠在一起，加上 Claude 模型，才组成了一个能在我的项目里干活的 Agent。

而且每家产品都是这个结构：

```
Claude Code = Claude  + Anthropic 造的 Harness
Codex CLI   = GPT     + OpenAI 造的 Harness
Gemini CLI  = Gemini  + Google 造的 Harness
Cursor      = 多模型  + Cursor 团队造的 Harness
```

模型不同，但 Harness 要解决的问题完全一样——怎么执行代码、怎么读写文件、上下文太长怎么办、大任务怎么拆、怎么控制权限。

## Harness 工程师到底在干嘛

搞清楚这些概念之后，"Harness 工程师"这个身份就很清晰了：

**你不是在给 Agent 造个家，你是在造 Agent 本身的一半。**

具体来说就是两件事：给 Agent 造适合它的身体，给它该去干什么活的工作手册。身体造得好不好、手册写得清不清楚，直接决定了这个 Agent 能发挥出多少智能。

同一个 Claude 模型，放在一个只有聊天框的网页上，和放在配好了完整 Harness 的 Claude Code 里，表现天差地别。不是模型变聪明了，是 Harness 让它的智能有了表达空间。

**造好 Harness，Agent 会完成剩下的。**

## 结论

三句话总结：

1. **Agent = Model + Harness。** 不是二选一，缺哪个都不完整。模型提供智能，Harness 提供表达智能的空间，两者共同构成一个能干活的 Agent。

2. **"模型就是 Agent"是一句好的矫枉，但不是终点。** 它打醒了那些以为堆框架就能造出智能的人，但如果因此忽视 Harness 的作用，就从一个极端走到了另一个极端。智能在权重里没错，但光有智能，没有手脚和工作台，什么也干不成。

3. **Harness 工程师造的不是 Agent 的"家"，是 Agent 的"半条命"。** 你写的每一行 CLAUDE.md、每一条 Rule、每一个 Skill，都在决定这个 Agent 能把多少智能转化成实际产出。同一个大脑，配不同的身体和工具，就是完全不同的 Agent。
