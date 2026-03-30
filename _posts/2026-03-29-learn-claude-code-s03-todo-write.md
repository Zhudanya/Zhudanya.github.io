---
title: "learn-claude-code 学习笔记（三）：TodoWrite —— 给 Agent 一个待办清单"
date: 2026-03-29 22:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, todo-write, planning]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第三篇。s01 搭了循环，s02 加了工具箱，这篇看 s03 怎么解决一个更根本的问题：Agent 干着干着就忘了自己在干什么。

## 问题：对话越长，模型越容易跑偏

s02 的 Agent 有了四个工具，给它简单任务没问题。但复杂任务就出事了。

比如你说"重构这个文件：加类型提示、加文档字符串、加 main guard、加错误处理、加单元测试"——五个步骤。模型一开始记得很清楚，做完第一步，做第二步。但到第三步、第四步时，messages 里已经累积了大量工具调用和返回结果（几百上千行代码），原始 prompt 里的五个步骤被淹没在最前面。

messages 不清空，信息确实还在。但 LLM 不是数据库——它不会平等对待上下文里的每一个 token。**Transformer 的注意力机制天然对最近的内容给更高权重。** 原始 prompt 沉在上下文底部，几千行工具结果压在上面，模型的有效注意力已经照顾不到那五个步骤了。

结果就是：
- 做完 1-2 步就开始即兴发挥
- 重复做已经做过的事
- 跳步、漏步
- 做到一半跑偏去干别的

注意：这不是代码的问题，是**模型架构的特性**。注意力权重分布是训练决定的，你改不了。但你能改的是——让重要信息出现在注意力最强的位置。

---

## 解决方案：一个 todo 工具

加一个 `todo` 工具，让模型自己管理一个带状态的任务列表。核心设计：

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [x] #1: type hints   |
              | [>] #2: docstrings   |
              | [ ] #3: main guard   |
              +-----------------------+
                          |
              if 连续 3 轮没更新 todo:
                inject <reminder>
```

不是你替它规划，是**它自己规划、自己更新、自己追踪进度**。

---

## TodoManager——带状态的任务管理器

### class 是干什么的

先说 Python 基础。`class` 把数据和操作数据的函数打包在一起：

```python
class TodoManager:
    def __init__(self):       # 初始化：创建时执行一次
        self.items = []       # 数据：任务列表

    def update(self, ...):    # 操作：更新任务
    def render(self):         # 操作：渲染展示

TODO = TodoManager()          # 创建实例，整个程序生命周期内存在
```

为什么需要 class？因为 `self.items` 要在多次工具调用之间**保持状态**。模型第一次调 todo 创建了 3 个任务，第二次调 todo 更新状态——两次操作的是同一份数据。普通函数做不到这一点。

### update——更新任务列表

模型每次调用 todo 工具时，完整的调用链是：

```
模型输出 ToolUseBlock(name="todo", input={"items": [...]})
  → dispatch map 查找 handler
  → lambda **kw: TODO.update(kw["items"])
  → TODO.update([...])
```

update 做了四层校验，然后全量替换旧列表：

```python
def update(self, items: list) -> str:
    if len(items) > 20:                    # 1. 不超过 20 条
        raise ValueError(...)
    for item in items:
        if not text:                       # 2. 不能是空任务
            raise ValueError(...)
        if status not in ("pending",       # 3. 状态只能是这三种
                   "in_progress", "completed"):
            raise ValueError(...)
        if status == "in_progress":
            in_progress_count += 1
    if in_progress_count > 1:              # 4. 同时只允许一个 in_progress
        raise ValueError(...)
    self.items = validated
    return self.render()
```

**为什么同时只允许一个 in_progress？** 强制模型顺序聚焦——做完一件事再做下一件。不然模型可能标三个任务都是 in_progress，实际上哪个都没认真做。

**为什么是全量替换不是增量更新？** 模型每次传完整列表，不是传"把第 3 条改成 completed"。这样更简单，也避免了增量更新时的状态不一致。

注意 update 会 `raise ValueError`——这就是 s03 给 agent_loop 加 try/except 的原因。异常被捕获后变成错误信息返回给模型，模型看到 "Error: Only one task can be in_progress at a time" 就知道自己传错了，会修正后重试。

### render——渲染成可读文本

```python
def render(self) -> str:
    for item in self.items:
        marker = {"pending": "[ ]", "in_progress": "[>]",
                  "completed": "[x]"}[item["status"]]
        lines.append(f"{marker} #{item['id']}: {item['text']}")
    done = sum(1 for t in self.items if t["status"] == "completed")
    lines.append(f"\n({done}/{len(self.items)} completed)")
    return "\n".join(lines)
```

输出长这样：

```
[x] #1: Add type hints
[>] #2: Add docstrings        ← 正在做
[ ] #3: Add main guard

(1/3 completed)
```

这个字符串作为 tool_result 返回，有两个读者：
- **模型**——出现在上下文最近的位置，模型"看到"当前进度
- **你**——终端里 print 出来，你实时知道 Agent 在做什么

这就是 todo 解决注意力问题的核心机制：**每次更新，完整的进度表就出现在上下文最新的位置——注意力最强的地方。**

---

## Nag Reminder——追着模型问"你更新计划了吗"

光有 todo 工具还不够。模型天然倾向于"埋头干活"，经常忘记更新。所以 s03 加了一个催促机制：

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.insert(0, {"type": "text",
                        "text": "<reminder>Update your todos.</reminder>"})
```

逻辑很简单：
- 模型调了 todo → 计数器归零
- 没调 todo → 计数器 +1
- 连续 3 轮不更新 → 在 tool_result 最前面插一条提醒

`insert(0, ...)` 插在最前面——模型最先看到这条提醒。用 `<reminder>` 标签包裹，让模型识别出这是系统催促而不是工具输出。

**本质上是顺着模型的注意力特性做设计**：既然最近的内容权重最高，那就把提醒塞到最近的位置。

---

## agent_loop 的变化

和 s02 对比，循环的核心结构完全不变（while True → create → append → stop_reason → execute → append results）。新增了六处：

```python
def agent_loop(messages: list):
    rounds_since_todo = 0              # 【新增】计数器
    while True:
        response = client.messages.create(...)
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        used_todo = False              # 【新增】标记
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                try:                   # 【新增】异常捕获
                    output = handler(**block.input) if handler else ...
                except Exception as e:
                    output = f"Error: {e}"
                results.append(...)
                if block.name == "todo":    # 【新增】检测
                    used_todo = True
        rounds_since_todo = 0 if used_todo else rounds_since_todo + 1  # 【新增】更新
        if rounds_since_todo >= 3:     # 【新增】nag 注入
            results.insert(0, {"type": "text",
                                "text": "<reminder>Update your todos.</reminder>"})
        messages.append({"role": "user", "content": results})
```

每一处都是为 todo 机制服务的：计数器追踪模型行为，标记检测是否用了 todo，try/except 处理 TodoManager 的校验异常，nag 在模型"遗忘"时拉回注意力。

---

## system prompt 为什么变了

```python
# s02
"Use tools to solve tasks. Act, don't explain."

# s03
"Use the todo tool to plan multi-step tasks.
 Mark in_progress before starting, completed when done.
 Prefer tools over prose."
```

模型从 TOOLS schema 能知道 todo 工具**是什么**（更新任务列表的工具）和**怎么调用**（传 items 数组）。但不知道**什么时候该用、怎么用才对**。

system prompt 里新加的那句话就是使用规范：
- "plan multi-step tasks" → 遇到复杂任务要先列计划
- "Mark in_progress before starting" → 做之前先标正在做
- "completed when done" → 做完要标完成

工具是工具，使用规范是规范。就像给新员工一个项目管理系统，你还得告诉他"接了任务要标进行中，做完要标已完成"。

---

## s03 的本质

回到 Harness 工程的核心哲学：**让模型不偏航，但不替它画航线。**

Harness 提供了三样东西：
1. **TodoManager**——规划的工具（模型自己列计划、更新进度）
2. **只允许一个 in_progress**——聚焦的约束（做完一件再做下一件）
3. **Nag reminder**——行为的纠偏（忘了更新就催它）

但具体怎么拆分任务、先做什么后做什么、每步怎么执行——全是模型自己决定的。

你改不了模型的注意力分布（那是训练决定的），但你能顺着它的特性做设计——把重要信息反复刷新到注意力最强的位置。这就是 s03 做的事。

---

## 和 s02 的变更对比

| 组件 | s02 | s03 |
|------|-----|-----|
| 工具数量 | 4 | 5（+todo） |
| 规划能力 | 无 | TodoManager（带状态、带约束） |
| 行为提醒 | 无 | nag reminder（3 轮不更新就催） |
| 错误处理 | 无 | try/except 捕获工具异常 |
| agent_loop | 简单分发 | + 计数器 + nag 注入 |

下一篇我们看 s04：大任务拆小，每个小任务干净的上下文——子 Agent 是怎么回事。

---

*本文是 learn-claude-code 系列学习笔记的第三篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s03 课程。*
