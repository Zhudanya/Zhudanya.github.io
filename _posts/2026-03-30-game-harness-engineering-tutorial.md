---
title: "游戏开发 Harness Engineering 教学：为什么这套配置能让 AI Agent 高效写代码"
date: 2026-03-30 22:00:00 +0800
categories: [AI, Engineering]
tags: [harness-engineering, ai-agent, game-dev, go, unity]
---

这篇文章拆解我为一个 Unity + Go 微服务游戏项目搭建的完整 Harness。不讲理论，只讲这套配置**为什么有效、删了哪个会出什么问题、每个目录到底干什么**。

## 先说结论：为什么高效

**一句话：把人的经验变成 Agent 的约束，把重复的判断变成机械化检查。**

没有这套 Harness 时：
- Agent 改了自动生成的代码 → 下次 generate 全覆盖，白改
- Agent 用 Debug.Log 打日志 → 上线后日志混乱
- Agent 改完代码说"我改完了" → 你手动启动服务、手动测试、5-10 分钟
- Agent 修了一个 bug 引入两个新 bug → 没人发现，推上去了
- Agent 犯过的错下次还犯 → 因为它不记得

有了这套 Harness 后：
- Agent 碰自动生成的代码 → Hook 直接拦截，改不了
- Agent 写 Debug.Log → Hook 自动检测，要求换成 MLog
- Agent 改完代码 → /auto-work 自动编译、跑测试、跑 Robot、30 秒出结果
- Agent 修完 bug → 分数制审查，质量只升不降，不达标自动回滚
- Agent 犯过的错 → /fix-harness 自动更新 rules，下次不再犯

**效果数据**：验证时间从 5-10 分钟降到 30 秒，修 bug 轮数从 3-5 轮降到 1-2 轮。

---

## 第一部分：这套 Harness 引用了哪些机制

### 机制 1：三层隔离（解决"一锅粥"问题）

```
workspace/        ← 跨项目规则（配置表同步、协议约定）
  client/         ← Unity 客户端规则（四层架构、UniTask、MLog）
  server/         ← Go 服务端规则（ECS、RPC、错误处理）
```

为什么要三层？因为客户端和服务端的规则**完全不同**：
- 客户端禁止 `Debug.Log`，服务端禁止裸 `go func()`
- 客户端用 UniTask，服务端用 safego.Go
- 客户端不能 CLI 编译，服务端可以 `make build`

如果混在一起，Agent 在写 Go 代码时看到"禁止 Debug.Log"会困惑，在写 C# 时看到"用 safego.Go"也没用。三层隔离让每个上下文只看到相关的规则。

**删了会怎样**：把所有规则塞一个文件，Agent 上下文被无关规则占满，有效注意力被稀释——和 learn-claude-code s03 讲的注意力问题一样。

### 机制 2：机械执行 + AI 判断（解决"靠不住"问题）

这套 Harness 把每条规则分成两类：

```
[机械执行] — Hook 硬拦截，Agent 做不了假
[AI判断]  — 规则写进 rules，Agent 自觉遵守（可能不遵守）
```

举例：

| 规则 | 类型 | 实现 |
|------|------|------|
| 不能改 orm/ 目录 | [机械执行] | constitution-guard.sh PreToolUse Hook |
| 提交前跑 lint + test | [机械执行] | pre-commit-check.sh Hook |
| push 前必须 /review | [机械执行] | push-gate.sh Hook |
| 改完代码先列计划 | [AI判断] | golden-principles.md 规则 |
| 探索性操作用子 Agent | [AI判断] | golden-principles.md 规则 |

**为什么要区分**：AI 判断不是 100% 可靠。Agent 可能"忘了"跑 /verify 就直接 commit——但 pre-commit Hook 会拦住它。Hook 是最后一道防线，rules 是第一道引导。

**删了 Hook 会怎样**：Agent 偶尔会跳过 lint 直接提交、跳过 review 直接 push。你发现时代码已经推上去了。

### 机制 3：闸门链（解决"漏检"问题）

```
Agent 写代码
  → Edit/Write 触发 constitution-guard（拦截禁区编辑）
  → Edit .cs 文件触发 lint-cs-after（Roslyn 语法 + 规范检查）
Agent 提交
  → pre-commit-check（lint + test）
  → 通过后提醒"请执行 /review"
Agent 审查
  → /review 评分（100 分起扣）
  → 通过后写 push-approved 标记文件
Agent 推送
  → push-gate 检查 push-approved 是否存在
  → 存在才能 push，push 后删除标记（一次性）
```

每一关都有拦截点。代码从写到推，经过了**4 道闸门**。任何一道拦住了，后面的都走不了。

**删了 push-gate 会怎样**：Agent 可能 review 没过就 push（因为 review 是 AI 判断，不是机械执行）。push-gate 是最后的硬防线。

### 机制 4：自演进（解决"同样的错犯两次"问题）

```
Agent 犯错 → 修复后自动执行 /fix-harness → 更新对应 rule 文件
```

golden-principles.md 里写了：

> Agent 的修改导致错误并修复后，必须自动执行 fix-harness 流程

/fix-harness 有路由表——根据错误类型更新不同的 rule 文件：

| 错误类型 | 更新哪个文件 |
|---------|------------|
| 已知陷阱再犯 | known-pitfalls.md |
| 违反架构边界 | architecture-boundaries.md |
| 编码风格问题 | golden-principles.md |
| 自动生成代码被改 | constitution.md |

**删了会怎样**：Agent 今天踩的坑明天还会踩。你得自己记着"上次犯了什么错"然后手动加 rule——但你肯定会忘。

### 机制 5：分数制质量棘轮（解决"越改越差"问题）

```
审查分数 = 100 - (CRITICAL × 30) - (HIGH × 10) - (MEDIUM × 3)
PASS: ≥ 80 且无 CRITICAL
质量棘轮：分数只升不降，下降即回滚
```

auto-work 流水线里，每轮审查的分数必须 ≥ 上一轮。如果 Agent 的"修复"反而引入了新问题导致分数下降，自动 `git checkout .` 回滚。

**删了会怎样**：Agent 修一个 HIGH 问题时引入两个新 HIGH，review 还是 PASS（因为没有计分，HIGH < 3 个就过）。反复修反复引入，最后代码越来越乱。

---

## 第二部分：每个 Rule 和 Skill 的具体作用

### Rules（每次会话自动加载）

#### constitution.md — 禁区宪法

```
禁止修改：orm/、cfg_*.go、base/（submodule）、proto/（submodule）
```

这些是自动生成的代码或外部依赖。改了也没用——下次 generate 或 submodule update 就覆盖了。

**删了会怎样**：Agent 会去改 orm/ 下的文件"修 bug"，下次 `make orm` 全覆盖。或者改 base/ 里的基础库，其他项目也在用，影响面不可控。

#### golden-principles.md — 黄金原则

核心编码规范 + Agent 工作流，包含：
- 错误处理（%w 包装、禁止 _ = err）
- 并发（safego.Go、atomic）
- 数据层（所有 DB 走 db_server RPC）
- ECS 架构（Component 只存数据、System 只处理逻辑）
- **Agent 工作流**（规划先行、任务跟踪、子 Agent 策略、慢命令后台、提交链）

**删了会怎样**：Agent 不知道你的项目用 safego.Go 而不是裸 goroutine，不知道 DB 操作要走 db_server RPC。写出来的代码能编译但不符合架构——review 时才发现，浪费一整轮。

#### known-pitfalls.md — 已知陷阱

真实的 Agent 错误记录，比如：
- go.mod replace 没同步
- scene_server debug 构建的性能问题
- Windows 下 nul 文件名问题
- **禁止猜测式调试**——必须先收集运行时数据

**删了会怎样**：Agent 会重复踩这些坑。特别是"猜测式调试"——Agent 天然倾向于看到报错就猜一个修法，改了跑不过再猜，三五轮过去还没修好。"先拿数据再修"这条 rule 能把修复轮数从 5 轮降到 1-2 轮。

#### architecture-boundaries.md — 架构边界

依赖方向、模块通信方式、跨服务调用规则。

**删了会怎样**：Agent 会在 common/ 里 import servers/ 下的包（反向依赖），或者在非 db_server 里直接连 MongoDB（绕过数据层）。编译过了但架构被破坏。

### Skills（按需加载）

Skills 和 rules 的区别：rules 每次自动加载（因为是约束），skills 按需加载（因为是知识）。

客户端的 skills：core-systems（Manager 系统）、ui-framework（UI 系统）、3c-system、kcc-system、coding-standards 等。

**删了会怎样**：Agent 做 UI 功能时不知道 UIManager 的层级结构，可能直接 new Panel() 而不是走 UIManager.OpenPanel。不会报错，但不符合架构。skills 在的话，Agent 需要时加载完整指南，按规范做。

### Memory（持久化知识）

和 rules/skills 的区别：memory 是**事实性知识**（这个系统怎么工作的），rules 是**行为约束**（应该怎么做），skills 是**操作指南**（怎么做的详细步骤）。

**删了会怎样**：上下文压缩后 Agent 忘了"15 个服务的通信关系"或"RPC 链路怎么走"。memory 在磁盘上，压缩不影响——Agent 随时可以读。

---

## 第三部分：每个目录的作用和组合关系

### 完整目录结构

```
game-harness-engineering/
├── workspace/                    ← 跨项目层
│   ├── CLAUDE.md                 ← 入口：子项目列表、跨项目约束、索引
│   └── .claude/
│       ├── commands/fix-harness.md
│       ├── docs/                 ← 搭建指南、框架文档
│       ├── memory/               ← 跨项目知识（协议同步、联调陷阱）
│       └── settings.local.json   ← 权限配置
│
├── client/                       ← Unity 客户端层
│   ├── CLAUDE.md                 ← 入口：技术栈、架构、命令索引
│   └── .claude/
│       ├── rules/   (8 个)       ← 行为约束（每次自动加载）
│       ├── commands/ (11 个)     ← 工作流命令（用户/流水线调用）
│       ├── hooks/                ← 机械化检查（自动触发）
│       ├── memory/  (5 个)       ← 持久化知识（磁盘，不丢）
│       ├── docs/exec-plans/      ← 执行计划存档
│       └── settings.json         ← Hook 配置
│
├── server/                       ← Go 服务端层
│   ├── CLAUDE.md                 ← 入口：技术栈、架构、命令索引
│   ├── scripts/                  ← 集群启停脚本
│   └── .claude/
│       ├── rules/   (9 个)       ← 行为约束
│       ├── commands/ (12 个)     ← 工作流命令（含 auto-work/auto-bugfix/parallel-execute）
│       ├── hooks/   (4 个)       ← 闸门链（禁区→提交→审查→推送）
│       ├── memory/  (5 个)       ← 持久化知识
│       ├── scripts/              ← 并行执行脚本
│       ├── docs/exec-plans/      ← 执行计划存档
│       └── settings.json         ← Hook 配置
│
├── Docs/                         ← 知识沉淀（Agent 的输入和输出）
│   ├── Design/                   ← 策划需求（人工同步）
│   ├── Engine/Business/          ← 模块技术文档（做完自动沉淀）
│   ├── Engine/Research/          ← 技术调研（调研完自动沉淀）
│   ├── Version/                  ← 迭代记录（auto-work 自动沉淀）
│   └── Bugs/                     ← Bug 记录（auto-bugfix 自动沉淀）
│
└── Tools/                        ← 辅助工具（Hook 和脚本的"肌肉"）
    ├── CSharpSyntaxChecker/      ← Roslyn C# 语法检查器
    ├── auto-research/            ← 可量化任务的自动迭代框架
    │   ├── orchestrator.sh       ← 主循环
    │   └── agents/               ← 角色化 Agent（writer/reviewer/red/blue/extractor）
    └── claude-monitor/           ← 执行监控 + 数据收集 + 分析
```

### 它们怎么组合到一起

```
用户说"给老虎机加倍功能"
  │
  ▼
CLAUDE.md（入口）→ Agent 知道这是什么项目、有什么技术栈
  │
  ▼
rules/（约束）→ Agent 知道什么能做什么不能做
  │
  ▼
memory/（知识）→ Agent 知道 slot 模块怎么工作、RPC 链路怎么走
  │
  ▼
/auto-work 命令（流水线）→ 自动走完 分类→规划→编码→验证→审查→提交
  │
  ├── 编码时：golden-principles 约束写法
  ├── 编辑 .cs 时：lint-cs-after Hook 自动检查语法
  ├── 编辑禁区时：constitution-guard Hook 自动拦截
  ├── 验证时：/verify-server 跑编译+lint+test+Robot
  ├── 审查时：/review 分数制评分，不达标回滚
  ├── 提交时：pre-commit Hook 再跑一遍 lint+test
  ├── 推送时：push-gate 检查 push-approved
  │
  ▼
Docs/（沉淀）→ 规划和结果自动写入 Version/
  │
  ▼
/fix-harness（自演进）→ 如果过程中犯了错，自动更新 rules
```

每个目录都有明确的角色：

| 目录 | 角色 | 什么时候生效 |
|------|------|-----------|
| CLAUDE.md | 身份和入口 | 每次会话开始 |
| rules/ | 行为约束 | 每次会话自动加载 |
| memory/ | 领域知识 | Agent 需要时读取 |
| commands/ | 工作流定义 | 用户或流水线调用时 |
| hooks/ | 机械化检查 | 工具执行前后自动触发 |
| Docs/ | 知识沉淀 | 流水线完成后自动写入 |
| Tools/ | 辅助工具 | Hook 和脚本调用 |
| scripts/ | 集群和并行 | 命令内部调用 |

**缺任何一个都能跑，但效果会打折**：
- 没有 rules → Agent 能写代码但不符合架构
- 没有 hooks → 规则靠 Agent 自觉，偶尔漏检
- 没有 memory → 上下文压缩后忘了项目知识
- 没有 commands → 每步都要手动触发
- 没有 Docs → 工作产出没沉淀，下次从零开始
- 没有 Tools → Hook 只能做文本级检查，不能做 AST 级

**全部组合在一起**：约束解决"做对"，流水线解决"做完"，闸门解决"不漏检"，自演进解决"不重犯"，知识库解决"不遗忘"，沉淀解决"不浪费"。

---

## 怎么从零搭建类似的 Harness

如果你想为自己的项目搭一套，优先级：

```
第 1 天：CLAUDE.md + constitution.md + golden-principles.md
  → Agent 知道项目是什么，知道什么不能做

第 2 天：/verify + /review + known-pitfalls.md
  → Agent 能自己验证，犯过的错被记录

第 3 天：Hooks（constitution-guard + pre-commit + push-gate）
  → 关键规则机械化执行，不靠 Agent 自觉

第 4 天：/auto-work + 分数制 review
  → 全自动流水线，一条命令走完开发周期

之后持续：/fix-harness 自演进 + memory 积累 + Docs 沉淀
  → 越用越好
```

完整的搭建指南见 `workspace/.claude/docs/harness-bootstrap-guide.md`（615 行），从预分析到交付，6 个阶段，预计 3.5 小时完成最小可用版本。

---

*本文基于 [game-harness-engineering](https://github.com/Zhudanya/game-harness-engineering) 项目。Harness Engineering 的理论基础来自 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 的 12 课课程。*
