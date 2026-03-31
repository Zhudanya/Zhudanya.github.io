---
title: "learn-claude-code 学习笔记（四）：Subagent —— 大任务拆小，上下文隔离"
date: 2026-03-22 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, subagent]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第四篇。s03 给 Agent 加了待办清单解决"忘记计划"的问题，这篇看 s04 怎么解决另一个上下文问题：噪声污染。

## 问题：上下文被噪声污染

s03 的 Agent 所有工具调用的结果都堆在同一个 messages 里。来看一个场景：你让 Agent "先查测试框架，然后修 utils.py 里的 bug"。

```
messages = [
    user: "先查测试框架，然后修 utils.py 里的 bug"

    --- 查测试框架的过程 ---
    assistant: 调 read_file("setup.py")
    user: tool_result（80 行内容）
    assistant: 调 read_file("requirements.txt")
    user: tool_result（30 行内容）
    assistant: 调 read_file("pyproject.toml")
    user: tool_result（50 行内容）
    assistant: 调 read_file("tests/conftest.py")
    user: tool_result（40 行内容）
    assistant: "用的是 pytest。"

    --- 现在修 bug ---
    assistant: 调 read_file("utils.py")    ← 上下文里有 200 行噪声
]
```

模型在修 bug 时，上下文里有 200 行 setup.py、requirements.txt、conftest.py 的内容。这些东西对修 bug **完全没用**，但模型的注意力机制还是会给它们分配权重。

**噪声 = 对当前任务没用但占着上下文的信息。污染 = 这些噪声稀释了模型对有用信息的注意力。**

s03 的 todo 解决的是"模型忘记计划"——把重要信息刷到最近的位置。s04 解决的是"上下文被无关内容塞满"——把噪声隔离到独立的空间里。

---

## 解决方案：子 Agent

加一个 `task` 工具。父 Agent 把子任务委派出去，子 Agent 在独立的上下文里干活，只把结果摘要返回。

```
父 Agent（messages=[...已有的上下文...])
    │
    │  调用 task 工具，传一个 prompt
    ▼
子 Agent（messages=[] 全新的空上下文）
    │  自己跑循环：读文件、执行命令
    │  可能跑了 20 次工具调用
    ▼
返回一段摘要给父 Agent
子 Agent 的整个 messages 被丢弃
```

同样的场景，有子 Agent 后：

```
messages = [
    user: "先查测试框架，然后修 utils.py 里的 bug"
    assistant: 调 task("查一下这个项目用什么测试框架")
    user: tool_result("这个项目用的是 pytest。")    ← 就一句话

    --- 现在修 bug ---
    assistant: 调 read_file("utils.py")              ← 上下文干干净净
]
```

200 行噪声变成了 1 句话。那 200 行去哪了？在子 Agent 的 `sub_messages` 里——函数返回后被 Python 垃圾回收了。

---

## task 不是官方工具

先澄清一件事：`task` 是这个项目**自己定义**的工具，不是 Anthropic API 内置的。和 s02 里自己定义 `read_file`、`write_file` 完全一个道理。

Anthropic API 不知道什么是"子 Agent"。从 API 的角度看，以下三次调用没有任何区别：

```python
# 模型调 bash → 你的代码跑 subprocess → 返回字符串
ToolUseBlock(name="bash", input={"command": "ls"})

# 模型调 read_file → 你的代码读文件 → 返回字符串
ToolUseBlock(name="read_file", input={"path": "hello.py"})

# 模型调 task → 你的代码启动子 Agent 循环 → 返回字符串
ToolUseBlock(name="task", input={"prompt": "find testing framework"})
```

API 看到的都是：模型调了一个工具，你返回了一段文字。中间发生了什么——跑了 subprocess 还是读了文件还是启动了一整个子 Agent 循环——API 完全不知道也不关心。

那模型怎么知道什么时候用 task 而不是 bash 或 read_file？靠 description：

```python
{"name": "bash",      "description": "Run a shell command."}
{"name": "read_file", "description": "Read file contents."}
{"name": "task",      "description": "Spawn a subagent with fresh context.
                       It shares the filesystem but not conversation history."}
```

模型根据 description 自己判断——这个任务需要探索性的多步操作，适合委派出去。这是模型训练出来的推理能力，不是 API 的规则。

**"子 Agent"完全是 Harness 层面的概念**——在工具的执行函数里再跑一次完整的 agent loop，用独立的 messages。API 对此一无所知。

---

## 两套 system prompt——不同的角色

```python
# 父 Agent：委派者
SYSTEM = "You are a coding agent... Use the task tool to delegate exploration or subtasks."

# 子 Agent：执行者 + 总结者
SUBAGENT_SYSTEM = "You are a coding subagent... Complete the given task, then summarize your findings."
```

"summarize your findings" 这个词很关键——引导子 Agent 在最后一轮输出摘要，而不是一堆细节。因为只有最后一轮的文字会返回给父 Agent。

---

## 两套工具列表——防止无限递归

```python
CHILD_TOOLS = [bash, read_file, write_file, edit_file]        # 4 个基础工具
PARENT_TOOLS = CHILD_TOOLS + [task]                            # 多一个 task
```

子 Agent 没有 task 工具——不能再派生子 Agent。为什么？防止递归无限派生：子 Agent 派子子 Agent，子子 Agent 再派子子子 Agent……

---

## run_subagent——子 Agent 的完整实现

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

这就是一个完整的 agent loop，和 s01 的核心循环几乎一模一样。但有几个关键区别：

**`sub_messages = []` 全新上下文：** 不是父 Agent 的 messages，是独立的空列表。子 Agent 看不到父 Agent 的任何对话历史。

**`for _ in range(30)` 安全上限：** 父 Agent 的 agent_loop 是 `while True`，子 Agent 最多跑 30 轮。防止子任务失控。

**`SUBAGENT_SYSTEM` 不同的角色：** 子 Agent 被告知要"完成任务并总结"。

**`CHILD_TOOLS` 没有 task：** 子 Agent 不能再派生子 Agent。

**只返回最后的文字：** `sub_messages` 在函数返回后被丢弃。父 Agent 收到的只是一段摘要。

---

## agent_loop 的变化

和 s03 对比，循环的核心结构还是不变。区别在工具执行部分——task 工具不走 dispatch map，单独处理：

```python
for block in response.content:
    if block.type == "tool_use":
        if block.name == "task":                         # task 单独处理
            desc = block.input.get("description", "subtask")
            print(f"> task ({desc}): {block.input['prompt'][:80]}")
            output = run_subagent(block.input["prompt"])
        else:                                            # 其他工具走 dispatch map
            handler = TOOL_HANDLERS.get(block.name)
            output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

另外注意：s04 去掉了 s03 的 todo 机制和 nag reminder。每个 session 聚焦一个 harness 机制——s03 聚焦规划，s04 聚焦上下文隔离。到最终的 s_full 里会把所有机制组合在一起。

---

## 子 Agent 共享什么、不共享什么

| 维度 | 共享吗 | 原因 |
|------|--------|------|
| 文件系统 | 共享 | 子 Agent 读写的是同一个目录下的同一批文件 |
| 对话历史（messages） | 不共享 | 子 Agent 有独立的 sub_messages |
| 工具执行函数 | 共享 | 同一个 TOOL_HANDLERS，同一套 run_bash/run_read... |
| system prompt | 不共享 | 父是"委派者"，子是"执行者+总结者" |
| 可用工具列表 | 不完全共享 | 子 Agent 没有 task 工具 |

这就像公司里经理派活给员工——共享同一个办公室（文件系统），但员工有自己的笔记本（messages），做完了口头汇报结果（return 摘要），过程笔记自己留着或者扔掉（sub_messages 丢弃）。

---

## 小结

s04 解决的问题和 s03 不同但互补：

| | s03 todo | s04 subagent |
|---|---------|-------------|
| 解决什么 | 模型忘记计划 | 上下文被噪声污染 |
| 思路 | 把重要信息刷到最近位置 | 把噪声隔离到独立上下文 |
| 核心机制 | TodoManager + nag | run_subagent + 独立 messages |
| 本质 | 对抗注意力稀释 | 对抗噪声堆积 |

两者都是 Harness 层面的工程——不改模型，改环境。s03 顺着模型的注意力特性做设计（重要信息放最近），s04 从源头减少噪声（不让无关信息进入父上下文）。

下一篇我们看 s05：用到什么知识，临时加载什么知识——Skill 系统是怎么回事。

**Go 语言实现**：[s04/main.go](https://github.com/Zhudanya/learn-claude-code/tree/main/agents_go/s04/main.go)

---

*本文是 learn-claude-code 系列学习笔记的第四篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s04 课程。*
