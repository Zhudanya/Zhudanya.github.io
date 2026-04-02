---
title: "Danya 完全使用指南 — 游戏开发 AI 编程助手"
date: 2026-04-02 10:00:00 +0800
categories: [AI Agent, Game Development]
tags: [danya, ai-agent, game-dev, harness, cli]
pin: true
toc: true
---

## 什么是 Danya

Danya 是一个运行在终端中的 AI 编程助手，**专门为游戏开发场景设计**。它不是通用的代码补全工具，而是一个理解游戏项目架构、强制执行质量标准、能自动化整个开发工作流的 Agent。

**开箱即用** — 进入游戏项目启动 Danya，它会自动检测引擎类型（Unity / Unreal / Godot / Go 服务端），生成完整的 Harness 治理体系（规则、命令、Hook、记忆），不需要手动搭建任何环境。

**核心定位**：将经过验证的 Game Harness Engineering 内置到 AI Agent 中，让游戏开发者开箱即用。

---

## 安装与更新

### 安装

```bash
# 方式一：npm 安装（推荐）
npm install -g @danya-ai/cli

# 方式二：从源码安装
git clone https://github.com/Zhudanya/danya.git
cd danya
bun install && bun run build
npm install -g .
```

```bash
# 验证安装
danya --version
```

### 更新

```bash
npm install -g @danya-ai/cli@latest
```

---

## 首次使用

### 1. 启动 Danya

```bash
cd <你的游戏项目>
danya
```

首次启动会引导你配置 AI 模型。同时会自动检测项目引擎类型并生成 `.danya/` 目录。

### 2. 配置 AI 模型

在对话中输入 `/model`：

1. 选 **Manage Model List** → 添加新模型
2. 选择 Provider：
   - **Custom Messages API** — Claude 系列（Anthropic 官方接口）
   - **Custom OpenAI-Compatible API** — GPT / DeepSeek / 千问 / GLM 等
   - **Ollama** — 本地模型
3. 粘贴 API Key → 后续步骤一路回车（使用默认值）
4. 回到 `/model` 页面选择要用的模型

#### 常用模型配置

| 模型 | Provider | Base URL | Model ID |
|------|----------|----------|----------|
| Claude Opus 4.6 | Messages API | `https://api.anthropic.com` | `claude-opus-4-6` |
| Claude Sonnet 4.6 | Messages API | `https://api.anthropic.com` | `claude-sonnet-4-6` |
| DeepSeek V3 | Messages API | `https://api.deepseek.com/anthropic` | `deepseek-chat` |
| GPT-4o | OpenAI-Compatible | `https://api.openai.com/v1` | `gpt-4o` |
| 千问 Max | OpenAI-Compatible | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-max` |

模型配置保存在 `~/.danya/config.json`，无需手动编辑。

#### 多模型协同

Danya 支持 4 种语义指针，让不同任务用不同模型：

```yaml
pointers:
  main: "DeepSeek V3"     # 主对话用最强模型
  task: "Qwen Max"        # 子任务用便宜模型
  compact: "GLM-4"        # 上下文压缩用快速模型
  quick: "GLM-4"          # 分类判断用快速模型
```

### 3. 初始化 Harness

Danya 首次进入项目时会自动初始化 `.danya/` 目录。也可以手动执行：

```
/init
```

这会：
- 检测项目引擎类型
- 释放完整的 Harness 体系（规则、命令、Hook、记忆模板）
- 如果存在 `.claude/` 或 `.codex/` 目录，自动整合到 `.danya/`
- 根据 AI 模型类型生成 `CLAUDE.md`（Claude 模型）或 `AGENTS.md`（其他模型）

### 4. 检查环境依赖

```bash
danya check-env
```

输出类似：

```
=== Danya Environment Check ===
  [OK] danya
  [OK] git
  [OK] make
  [OK] python3
  [OK] go
  [SKIP] dotnet (only needed for C# syntax checking)

All required dependencies found.
```

---

## .danya/ 目录结构

```
.danya/
├── rules/                 — 约束规则（每次会话自动加载）
│   ├── constitution.md    — 禁区规则（不可编辑的自动生成代码）
│   ├── golden-principles.md — 编码黄金原则（UniTask, error wrapping 等）
│   ├── known-pitfalls.md  — 已知踩坑（自演进积累）
│   ├── architecture-boundaries.md — 架构依赖方向
│   └── <engine>-style.md  — 引擎特定风格指南
│
├── commands/              — 工作流命令（/auto-work, /review 等）
│   ├── auto-work.md
│   ├── auto-bugfix.md
│   ├── review.md
│   ├── fix-harness.md
│   ├── plan.md
│   ├── verify.md
│   └── parallel-execute.md
│
├── memory/                — 持久领域知识（不随上下文压缩丢失）
│   ├── MEMORY.md          — 索引
│   └── <模块>.md          — Agent 学到的项目知识
│
├── hooks/                 — 机械执行脚本（Agent 无法绕过）
│   ├── constitution-guard.sh  — Gate 0: 禁区防护
│   ├── pre-commit.sh          — Gate 3: 提交前 lint+test
│   ├── post-commit.sh         — Gate 4: 提交后提醒 review
│   ├── push-gate.sh           — Gate 5: 推送前检查 push-approved
│   ├── harness-evolution.sh   — 自演进检测
│   └── syntax-check.sh        — 语法检查
│
├── agents/                — 角色 Agent 规格
│   ├── code-writer.md     — 编码 Agent
│   ├── code-reviewer.md   — 审查 Agent
│   ├── red-team.md        — 红队 Agent（找 bug）
│   ├── blue-team.md       — 蓝队 Agent（修 bug）
│   └── skill-extractor.md — 经验提炼 Agent
│
├── scripts/               — Shell 强制编排脚本
│   ├── auto-work-loop.sh  — 全自动流水线（Shell 强制版）
│   ├── parallel-wave.sh   — 多 Agent 波次并行
│   ├── red-blue-loop.sh   — 红蓝对抗循环
│   ├── orchestrator.sh    — 自动迭代刷分
│   ├── verify-server.sh   — 服务端量化验证（0-100 分）
│   ├── verify-client.sh   — 客户端量化验证
│   ├── check-env.sh       — 环境检查
│   └── monthly-report.sh  — 月度报告
│
├── monitor/               — 监控数据采集
│   ├── log-tool-use.py    — 工具调用记录
│   ├── log-session-end.py — 会话结束记录
│   ├── log-verify.py      — 验证耗时记录
│   ├── log-bugfix.py      — Bug 修复记录
│   ├── log-review.py      — 审查分数记录
│   ├── analyze.py         — 数据分析（8 种指标）
│   └── dashboard.py       — 实时监控仪表盘
│
├── templates/             — 任务定义模板
│   └── program-template.md
│
├── settings.json          — Hook 注册
├── gate-chain.json        — 门禁链配置
└── guard-rules.json       — 禁区规则
```

**用户可以自由增删改** `.danya/` 下的任何文件。Danya 不会覆盖用户的自定义内容。

---

## 快捷键

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+G` | 打开外部编辑器，关闭后内容自动回填 |
| `Shift+Enter` | 输入框内换行但不发送 |
| `Enter` | 提交 |
| `Ctrl+M` | 快速切换模型 |
| `Shift+Tab` | 切换输入模式（普通 / Bash / 记忆） |

---

## 对话内命令详解

### /auto-work — 全自动流水线

```
/auto-work "添加背包排序功能"
```

Agent 自动走完 7 个阶段：

1. **Stage 0: 分类** — 判断是 bug / feature / refactor
2. **Stage 1: 规划** — 分析需求，列出文件清单
3. **Stage 2: 编码** — 写代码，每个文件改完立即编译检查（fail-fast）
4. **Stage 3: 审查** — 100 分制评分，<80 或有 CRITICAL 则失败
5. **Stage 4: 提交** — git commit（pre-commit hook 会跑 lint+test）
6. **Stage 5: 沉淀** — 自动写文档到 Docs/
7. **Stage 6: 自演进** — 检查开发过程中的错误，更新 rules

**终止条件**：
- 验证 3 轮失败 → 终止
- 审查 3 轮分数 < 80 → 终止
- 提交 2 次失败 → 终止

### /auto-bugfix — Bug 自动修复

```
/auto-bugfix "角色状态切换时动画异常"
```

**必须先复现才能修**：

1. **复现** — 分析 bug，找到复现步骤，验证 bug 存在
2. **根因分析** — 不猜测，读代码、查日志
3. **修复** — 最多 5 轮：改代码 → 验证 → 不通过则重试
4. **审查** — 100 分制评分
5. **提交** — git commit
6. **沉淀** — 写入 Docs/Bugs/

### /review — 评分制代码审查

```
/review
```

**评分规则**：

| 级别 | 扣分 | 示例 |
|------|------|------|
| CRITICAL | -30 | 编译失败、编辑禁区文件、数据损坏风险 |
| HIGH | -10 | 未处理错误、竞态条件、缺少校验 |
| MEDIUM | -3 | 命名违规、缺少日志、死代码 |

- 通过线：**≥ 80 且无 CRITICAL**
- 质量棘轮：分数只能升不能降
- 通过后生成 `push-approved` 令牌（一次性使用）

**审查覆盖 4 个维度**：
1. 架构合规（禁区、跨层引用、依赖方向）
2. 编码标准（引擎规则、错误处理、命名）
3. 逻辑审查（边界、并发、错误传播）
4. Harness 完整性（错误是否已固化到规则）

### /fix-harness — 自演进

```
/fix-harness "编译时忘记检查 nil 导致 panic"
```

Agent 会：
1. 分析错误模式
2. 路由到正确的规则文件：
   - 禁区违规 → `constitution.md`
   - 编码原则违反 → `golden-principles.md`
   - 已知踩坑 → `known-pitfalls.md`
   - 架构边界违反 → `architecture-boundaries.md`
3. 添加规则：错误写法 + 正确写法
4. 保持每个规则文件 < 550 行

**自动触发**：PostToolUse Hook 检测到"出错→修复"模式时，系统自动提示运行 `/fix-harness`。

### /plan — 需求分析与规划

```
/plan "实现武器升级系统"
```

输出：
1. 需求分析（要改什么、为什么改）
2. 文件清单（每个文件 + 一句话改动意图 + 风险等级）
3. 执行顺序（依赖关系、哪些可以并行）
4. 验证策略

### /verify — 机械验证

```
/verify          # 默认 quick
/verify build    # 编译
/verify full     # 编译 + 测试
```

| 级别 | 内容 |
|------|------|
| quick | lint + 语法检查 |
| build | quick + 完整编译 |
| full | build + 运行测试 |

### /parallel-execute — 波次并行执行

```
/parallel-execute prepare "给老虎机加倍功能"
```

Agent 会：
1. 分析需求，拆解为原子任务（task-01.md, task-02.md...）
2. 声明依赖关系
3. 计算波次（拓扑排序）
4. 展示波次计划，等用户确认
5. 启动 `parallel-wave.sh`：每个任务在独立 worktree 中执行

**任务文件格式**：

```markdown
---
depends: []
---

## 任务描述
修改 handler.go，添加倍率参数支持。

## 验证方式
go build ./servers/logic_server/...
```

依赖声明：
- `depends: []` — 无依赖，第一波
- `depends: [01]` — 依赖 task-01
- `depends: [01, 02]` — 依赖 01 和 02

### /orchestrate — 自动迭代刷分

```
/orchestrate my-task.md
```

需要一个**任务定义文件**（基于 `.danya/templates/program-template.md` 模板）：

```markdown
## Goal
增加测试覆盖率从 15% 到 60%

## Modifiable Scope
- servers/logic_server/internal/slot/

## Forbidden Files
- orm/
- common/config/cfg_*.go

## Quantitative Metrics
- make build: 40 分
- make lint: 20 分
- make test: 40 分

## Context
这个模块处理老虎机游戏逻辑
```

**执行流程**：AI 写代码 → 量化验证(0-100 分) → 分数 ≥ 基线就提交、< 基线就回滚 → 循环 N 轮。5 次连续失败自动熔断。

### /red-blue — 红蓝对抗

```
/red-blue servers/logic_server/
```

1. **红队**（只读）：找所有 bug（边界、错误路径、并发、安全）
2. **蓝队**（可写）：按优先级修 bug（CRITICAL → HIGH → MEDIUM）
3. **编译检查**：修复后必须能编译
4. **循环**：直到红队找到 0 个 bug（最多 5 轮）
5. **经验提炼**：skill-extractor 分析日志，写入 rules/memory

### /monitor — 查看 Harness 效果

```
/monitor summary 7        # 最近 7 天综合报告
/monitor tools            # 工具使用分布
/monitor reviews 30       # 最近 30 天审查分数
/monitor bugfixes 30      # Bug 修复效率
/monitor sessions         # 会话数量
```

---

## 终端命令详解

### danya auto-work — Shell 强制流水线

```bash
danya auto-work "实现武器升级系统"
danya auto-work "修复登录 bug" --model opus
```

和对话内 `/auto-work` 的区别：**Shell 强制每阶段用独立的 `danya -p` 执行，Agent 不可跳步，上下文隔离**。适合大任务、无人值守场景。

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--model <m>` | 使用的模型 | sonnet |
| `--max-turns <n>` | 每阶段最大轮次 | 30 |

### danya parallel — 多 Agent 并行

```bash
danya parallel Docs/Version/v1.3/weapon/tasks
```

读取任务目录下的 task-NN.md 文件，按依赖分波次，每个任务在独立 worktree 中启动独立 danya 实例。编译通过自动合并，失败自动回滚。

### danya red-blue — 红蓝对抗（无人值守）

```bash
danya red-blue .
danya red-blue servers/logic_server/ --rounds 3 --model opus
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-n, --rounds <n>` | 最大轮数 | 5 |
| `--model <m>` | 模型 | sonnet |

### danya orchestrate — 自动迭代

```bash
danya orchestrate my-task.md
danya orchestrate my-task.md -n 30 --model opus
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-n, --iterations <n>` | 最大迭代次数 | 20 |
| `--model <m>` | 模型 | sonnet |

### danya analyze — 数据分析

```bash
danya analyze --metric summary --days 7       # 综合报告
danya analyze --metric compare --days 7       # 本周 vs 上周对比
danya analyze --metric tool-usage --days 7    # 工具使用分布
danya analyze --metric top-tools --top 10     # Top 10 工具
danya analyze --metric session-count --days 30 # 会话数
danya analyze --metric verify-time --days 7   # 验证耗时
danya analyze --metric bugfix-rounds --days 30 # Bug 修复效率
danya analyze --metric review-scores --days 30 # 审查分数趋势
```

8 种指标：

| 指标 | 看什么 |
|------|--------|
| `summary` | 综合（验证、bug 修复、审查、工具、会话） |
| `compare` | 本周期 vs 上周期对比（带趋势箭头） |
| `tool-usage` | 每个工具被调用了多少次 |
| `top-tools` | Top N 工具排行 |
| `session-count` | 每日会话数分布 |
| `verify-time` | 验证耗时（按类型，平均/最快/最慢/通过率） |
| `bugfix-rounds` | Bug 修复轮数、成功率 |
| `review-scores` | 审查均分、通过率、CRITICAL/HIGH/MEDIUM 数量趋势 |

### danya dashboard — 实时监控

```bash
danya dashboard          # 看一次快照
danya dashboard -w       # 持续监控，每 5 秒刷新
danya dashboard -w 3     # 每 3 秒刷新
danya dashboard -v       # 详细模式（显示 PID、内存、最近工具调用）
danya dashboard -w -v    # 持续 + 详细
```

显示：运行中的 Agent 进程数、活跃对话、后台任务。

### danya report — 月度报告

```bash
danya report
```

汇总 orchestrator 的所有迭代结果：总 session 数、迭代轮数、成功率、最高基线分。

---

## 门禁链

每次代码变更经过 6 道质量门禁：

```
Edit → Guard → Syntax → Verify → Commit → Review → Push
       Hook     Hook     Tool     Hook    AI+Tool   Hook
       (硬拦截) (即时检查)(编译测试)(lint+test)(评分制)(令牌制)
```

- **Gate 0 Guard**：PreToolUse Hook，编辑禁区文件时直接拦截
- **Gate 1 Syntax**：PostToolUse Hook，编辑后即时语法检查
- **Gate 2 Verify**：工具调用，编译 + 测试
- **Gate 3 Commit**：PreToolUse Hook，commit 前跑 lint + test
- **Gate 4 Review**：AI + ScoreReview 工具，100 分制评分
- **Gate 5 Push**：PreToolUse Hook，检查 push-approved 令牌

**Push 令牌制**：`/review` 通过后生成 `.danya/push-approved` 文件，push 时检查并消费（一次性使用）。

---

## 引擎检测与变体

Danya 启动时自动检测项目引擎类型：

| 引擎 | 检测方式 | 注入的领域知识 |
|------|---------|------------|
| Unity | `ProjectSettings/` + `Assets/` | MonoBehaviour 生命周期、UniTask、对象池、事件配对、架构层级 |
| Unreal | `*.uproject` | UPROPERTY、UE_LOG、FRunnable、命名规范(F/U/A/E) |
| Godot | `project.godot` | 类型提示、信号配对、_physics_process、资源加载 |
| Go Server | `go.mod` | 错误包装、safego.Go、RPC handler 规范、ECS 约束 |

### Workspace 三层隔离

当项目包含 client + server 子目录时，Danya 自动识别为 workspace 模式：

```
workspace/
├── .danya/            ← Layer 1: 跨项目规则（协议同步、版本管理）
├── client/
│   └── .danya/        ← Layer 2: 客户端专属（Unity 规则、C# Hook）
└── server/
    └── .danya/        ← Layer 2: 服务端专属（Go 规则、RPC 约束）
```

Layer 3 是 Session 级别的 git worktree 隔离（parallel-execute 时使用）。

---

## 自演进机制

当 Agent 犯错并修复后，系统自动检测"出错→修复"模式：

```
Agent 编译出错 → 修复 → 编译成功
  → PostToolUse Hook 检测到 error-then-fix
  → 提示："Error was fixed. Consider running /fix-harness"
  → Agent 运行 /fix-harness
  → 路由到 known-pitfalls.md
  → 添加：❌ 错误写法 + ✅ 正确写法
  → 下次不再犯同样的错
```

**规则文件约束**：每个文件保持 < 550 行。超过时合并精简。

---

## 上下文压缩

Danya 融合了两种压缩策略：

1. **选择性压缩**（来自 Codex）：只压缩最老的消息，保留最近 4 条原文
2. **8 段结构化摘要**（来自 Kode）：技术上下文、项目概览、代码变更、调试问题、当前状态、待办任务、用户偏好、关键决策
3. **关键文件自动恢复**：压缩后自动恢复最近访问的 5 个文件
4. **模型智能切换**：优先用 compact 模型压缩，装不下时自动切 main

**触发条件**：token 使用 ≥ 90% 上下文窗口。

**手动触发**：`/compact`

**调整阈值**：`/compact-threshold 0.85`（设为 85%）

---

## 5 个角色 Agent

这些 Agent 被 orchestrator 和 red-blue 脚本自动调用：

| 角色 | 工具权限 | 职责 |
|------|---------|------|
| **code-writer** | Edit, Write, Read, Bash, Grep, Glob | 只在指定范围内改代码，改完编译，最小改动 |
| **code-reviewer** | Read, Grep, Glob, Bash (只读) | 只读审查，100 分制打分 |
| **red-team** | Read, Grep, Glob, Bash (只读) | 假设代码有 bug，找边界/错误路径/并发/安全问题 |
| **blue-team** | Edit, Write, Read, Bash, Grep, Glob | 按优先级修 bug，防御式编程，最小修复 |
| **skill-extractor** | Read, Write, Grep, Glob | 从日志提炼规律写入 rules/memory（需 2+ 次出现） |

---

## 数据监控

### 自动采集（用户无感）

Danya 通过 Hook 自动采集数据到 `.danya/monitor/data/`：

| 数据 | Hook 类型 | 采集内容 |
|------|----------|---------|
| `tool-usage.jsonl` | PostToolUse | 工具名、时间、session_id |
| `sessions.jsonl` | Stop | 会话结束时间、原因 |
| `verify-metrics.jsonl` | 脚本调用 | 验证类型、结果、耗时 |
| `bugfix-metrics.jsonl` | 脚本调用 | Bug 描述、轮数、结果 |
| `review-metrics.jsonl` | 脚本调用 | 分数、问题数（CRITICAL/HIGH/MEDIUM） |

### 分析命令

```bash
# 综合报告
danya analyze --metric summary --days 7

# 本周 vs 上周对比（带趋势箭头）
danya analyze --metric compare --days 7

# 输出示例：
# ========================================
#   Harness Compare | last 7d vs prev 7d
# ========================================
#
#   Verify:
#     Avg duration: 12.3s (DOWN 15% OK)
#     Pass rate: 92% (UP 8% OK)
#
#   Bugfix:
#     Avg rounds: 1.5 (DOWN 25% OK)
#     Success: 90% (UP 10% OK)
#
#   Review:
#     Avg score: 91.2 (UP 5% OK)
#     CRITICAL: 0 (DOWN 100% OK)
# ========================================
```

---

## 从 Claude Code / Codex 迁移

如果你之前用 Claude Code 或 Codex，已有 `.claude/` 或 `.codex/` 配置：

| 场景 | Danya 行为 |
|------|-----------|
| 只有 `.claude/` | 自动整合到 `.danya/`，不丢失自定义内容 |
| 只有 `.codex/` | 同上 |
| 两个都有 | 都整合，`.claude/` 优先于 `.codex/` |
| 同名文件 | `.danya/` 保留，legacy 跳过 |
| 运行时 | 只读 `.danya/`，不再读 `.claude/` 或 `.codex/` |

---

## 常用场景示例

### 场景 1：小功能开发

```
danya
> /auto-work "添加背包排序功能"
```

Agent 自动走 7 阶段，不需要手动干预。

### 场景 2：大功能无人值守

```bash
danya auto-work "实现武器升级系统" --model opus
```

Shell 强制编排，每阶段独立 danya -p，不可跳步。

### 场景 3：复杂功能并行开发

```
danya
> /parallel-execute prepare "给老虎机加倍功能"
```

Agent 拆任务 → 展示波次 → 用户确认 → 多 Agent 并行。

### 场景 4：代码质量提升

```bash
danya red-blue servers/logic_server/
```

红队找 bug → 蓝队修 → 循环到零 bug → 提炼经验。

### 场景 5：自动迭代刷覆盖率

```bash
danya orchestrate test-coverage.md -n 30
```

AI 写测试 → 量化打分 → 通过提交 → 基线从 15% 一路刷到 60%。

### 场景 6：查看 Harness 效果

```bash
danya analyze --metric compare --days 7
```

本周 vs 上周：验证耗时降了多少、审查均分升了多少、bug 修复轮数减了多少。

---

## 支持的引擎

| 引擎 | 检测方式 | 构建工具 | 审查规则数 |
|------|---------|---------|-----------|
| Unity | `ProjectSettings/` + `Assets/` | UnityBuild, CSharpSyntaxCheck | 9 条 |
| Unreal Engine | `*.uproject` | UnrealBuild | 6 条 |
| Godot | `project.godot` | GodotBuild | 5 条 |
| Go 游戏服务器 | `go.mod` | GameServerBuild | 9 条 |

通用审查规则 4 条，所有项目自动应用。

---

## 13 个游戏专用工具

| 工具 | 用途 | 适用引擎 |
|------|------|---------|
| CSharpSyntaxCheck | Roslyn 即时语法检查 | Unity, Godot |
| UnityBuild | Unity 编译管线 | Unity |
| UnrealBuild | UBT 编译 | UE |
| GodotBuild | GDScript/C# 检查 | Godot |
| GameServerBuild | 分级验证 (lint/build/test) | Go Server |
| ProtoCompile | Protobuf 编译 + 桩代码生成 | 跨引擎 |
| ConfigGenerate | 配置表生成 | 跨引擎 |
| OrmGenerate | ORM 代码生成 | Go Server |
| ScoreReview | 100 分制评分审查 | 跨引擎 |
| GateChain | 门禁链编排 | 跨引擎 |
| KnowledgeSediment | 知识自动沉淀 | 跨引擎 |
| ArchitectureGuard | 依赖方向检查 | 跨引擎 |
| AssetCheck | 资源引用完整性 | Unity, Godot |

---

## 完整命令速查表

### 对话内（AI 判断模式）

| 命令 | 作用 |
|------|------|
| `/auto-work "需求"` | 全自动流水线 |
| `/auto-bugfix "bug"` | Bug 自动修复 |
| `/review` | 评分制审查 |
| `/fix-harness` | 自演进 |
| `/plan "需求"` | 规划 |
| `/verify [level]` | 验证 |
| `/parallel-execute prepare "功能"` | 并行开发 |
| `/orchestrate <task.md>` | 自动迭代 |
| `/red-blue [scope]` | 红蓝对抗 |
| `/monitor [metric] [days]` | 查看效果 |
| `/model` | 模型管理 |
| `/compact` | 手动压缩 |
| `/compact-threshold <ratio>` | 调整压缩阈值 |
| `/init` | 初始化 Harness |

### 终端（Shell 强制模式）

| 命令 | 作用 |
|------|------|
| `danya auto-work "需求"` | Shell 强制流水线 |
| `danya parallel <tasks-dir>` | 多 Agent 并行 |
| `danya red-blue [scope]` | 红蓝对抗 |
| `danya orchestrate <task.md>` | 自动迭代 |
| `danya analyze --metric <m>` | 数据分析 |
| `danya dashboard [-w] [-v]` | 实时监控 |
| `danya report` | 月度报告 |
| `danya check-env` | 环境检查 |
