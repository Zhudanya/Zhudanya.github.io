---
title: "learn-claude-code 学习笔记（一）：Agent Loop —— 一切的起点"
date: 2026-03-19 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, agent-loop]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第一篇。这个仓库用 12 个渐进式课程，拆解 Claude Code 背后的 Agent 架构。s01 是一切的起点——用不到 30 行核心代码，搭出一个能干活的 AI Agent。

## 先搞清楚几个概念

在碰代码之前，有几个概念必须理清，不然后面全是糊涂账。

### Agent 是什么

Agent 是一个经过训练的模型，学会了**感知环境、推理目标、采取行动**。注意，Agent 是上位概念，不只是 LLM：

```
Agent（能感知-推理-行动的模型）
├── LLM（Claude, GPT...）     ← 语言/代码领域
├── CNN（DQN...）             ← 游戏/视觉领域
├── AlphaStar                 ← 星际争霸
└── ...
```

LLM 是 Agent 的一种，不是 Agent 的全部。

### Harness 是什么

Harness 是 Agent 执行任务所需的一切外部支撑：

```
Harness = Tools + Knowledge + Observation + Action + Permissions
```

工具让它能动手，知识让它懂行，权限给它画边界。

### 它们的关系

这是我一开始想错的地方。总觉得"搭好 Harness，让 Agent 在里面工作"——好像是两个独立的东西。但其实 Harness 不是 Agent 的外部环境，**Harness 是 Agent 的组成部分**：

```
Agent（完整的智能体）
├── Model（大脑）—— 感知、推理、决策
└── Harness（身体 + 工作环境）—— 工具、知识、权限、执行
```

模型只能输出 token，不能真的执行命令。Harness 把模型的决策翻译成真实行动。两者合在一起，才是一个完整的 Agent。

> 更详细的概念辨析可以看我的上一篇：[搞清楚 Agent 和 Harness 到底是什么](/posts/what-is-agent-what-is-harness/)

---

## s01 的核心：一个 while 循环

整个 Agent 的骨架：

```
用户输入 → 发给模型 → 模型回复
                        ↓
                  stop_reason 是 "tool_use"？
                   /              \
                 是                否
                  ↓                ↓
            执行工具             结束
            结果喂回模型
            继续循环
```

**模型决定一切**——调什么工具、调几次、什么时候停。代码只是忠实执行模型的决定。

---

## 拆解 client.messages.create()

Agent 的每一轮"思考"都靠这一个 API 调用：

```python
response = client.messages.create(
    model=MODEL,        # 用哪个模型
    system=SYSTEM,      # 系统提示词
    messages=messages,  # 对话历史
    tools=TOOLS,        # 可用工具列表
    max_tokens=8000,    # 最大输出 token 数
)
```

逐个看。

### model — 用哪个大脑

必须是 Anthropic 支持的模型 ID，比如 `claude-opus-4-6`、`claude-sonnet-4-6`。写错直接报 404。

### system — 工作手册

```python
SYSTEM = "You are a coding agent at {cwd}. Use bash to solve tasks. Act, don't explain."
```

s01 的写法很简陋，就一句话。但 Claude Code 的真实 system prompt 是一份巨大的行为手册——身份定义、行为准则、安全边界、操作规范、输出风格、环境信息全在里面。**这就是 Harness 最核心的部分之一。** 内容你可以随便写，但质量直接决定 Agent 表现。

### messages — 对话记忆

格式是固定的，role 只有两种：`user` 和 `assistant`，必须严格交替：

```python
{"role": "user", "content": "帮我创建一个文件"}        # 用户说的话
{"role": "assistant", "content": response.content}      # 模型的回复
{"role": "user", "content": [tool_result...]}           # 工具执行结果
```

工具结果为什么放在 `role: "user"` 里？因为 API 要求严格交替。模型调了工具后（assistant），下一条必须是 user。工具结果本质上是"外部世界的反馈"，归在 user 这边。

### tools — 告诉模型它有什么工具

```python
TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

这是你自己定义的，但格式固定。三个字段：

- **name** — 你起的名字，模型靠它来调用
- **description** — 你写的描述，模型靠它来判断什么时候用这个工具
- **input_schema** — 你定义的参数格式，模型会按照这个 schema 生成参数

关键点：**模型不知道工具真正能干什么，它只看 name 和 description。** 工具背后的执行函数（`run_bash`），模型完全不知道。

### max_tokens — 模型一次最多说多少

设太小会截断回复。s01 设的 8000，够用。

### 哪些能自定义

| 参数 | 能自定义吗 | 说明 |
|------|-----------|------|
| model | 只能选 | 必须用 Anthropic 提供的 ID |
| system | 随便写 | Harness 工程师的核心发挥空间 |
| messages | 内容随便，格式固定 | 必须 user/assistant 交替 |
| tools | 随便定义 | name、description、schema 都你来 |
| max_tokens | 随便设 | 不超过模型上限就行 |

---

## 拆解 run_bash — 工具的执行函数

```python
def run_bash(command: str) -> str:
```

接收命令字符串，返回输出字符串。分三步：

**安全拦截：**

```python
dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
if any(d in command for d in dangerous):
    return "Error: Dangerous command blocked"
```

粗暴的字符串匹配，教学级别的实现。生产环境（Claude Code）有完整的权限治理。

**执行命令：**

```python
r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                   capture_output=True, text=True, timeout=120)
out = (r.stdout + r.stderr).strip()
return out[:50000] if out else "(no output)"
```

- `shell=True` — 支持管道、通配符
- `stdout + stderr` — 正常输出和错误输出合在一起返回
- `[:50000]` — 截断到 5 万字符，防止撑爆上下文

**超时兜底：**

```python
except subprocess.TimeoutExpired:
    return "Error: Timeout (120s)"
```

返回值永远是字符串——成功、失败、超时都是。模型看到错误信息后会自己决定下一步怎么办。

---

## 拆解 response.content — block 到底是什么

`response.content` 不是字符串，是一个 **block 列表**。模型的每次回复可能包含多个 block。

### 情况一：纯文字回复

```python
response.content = [
    TextBlock(type="text", text="当前目录有两个文件")
]
response.stop_reason = "end_turn"    # 不调工具，循环结束
```

### 情况二：想调工具

```python
response.content = [
    TextBlock(type="text", text="让我看看当前目录"),
    ToolUseBlock(
        type="tool_use",
        id="toolu_01ABC123",
        name="bash",
        input={"command": "ls -la"}
    )
]
response.stop_reason = "tool_use"    # 有工具调用，循环继续
```

### 情况三：一次调多个工具

```python
response.content = [
    ToolUseBlock(type="tool_use", id="toolu_01AAA", name="bash",
                 input={"command": "cat file1.py"}),
    ToolUseBlock(type="tool_use", id="toolu_01BBB", name="bash",
                 input={"command": "cat file2.py"}),
]
```

每个工具调用有不同的 id。

### block.id — 调用的身份证

API 自动生成的唯一标识，格式类似 `"toolu_01ABC123"`。用途只有一个：**把工具结果和工具调用对应起来。**

```python
# 模型调了两个工具
ToolUseBlock(id="toolu_01AAA", input={"command": "ls"})
ToolUseBlock(id="toolu_01BBB", input={"command": "pwd"})

# 返回结果时必须用对应的 id
{"type": "tool_result", "tool_use_id": "toolu_01AAA", "content": "file1.py"}
{"type": "tool_result", "tool_use_id": "toolu_01BBB", "content": "/home/user"}
```

id 对不上，API 直接报错。

### block.input — 模型生成的参数

模型按照你在 TOOLS 里定义的 `input_schema` 生成的参数字典。你定义了 `{"command": {"type": "string"}}`，模型就返回 `{"command": "ls -la"}`。

如果你的工具有多个参数（比如后续 s02 的 `read_file` 有 `path` 和 `limit`），`block.input` 就会是 `{"path": "hello.py", "limit": 100}`。

---

## 完整的一轮循环

用户说"帮我创建一个 hello.py"，messages 的变化过程：

```python
# 1. 用户输入
[user: "帮我创建一个 hello.py"]

# 2. 模型回复：想调 bash
[user: "帮我创建一个 hello.py"]
[assistant: TextBlock("好的") + ToolUseBlock(id="AAA", command="echo ...> hello.py")]

# 3. 执行工具，结果喂回去
[user: "帮我创建一个 hello.py"]
[assistant: ...]
[user: tool_result(id="AAA", content="(no output)")]

# 4. 模型看到执行成功，纯文字回复，循环结束
[user: "帮我创建一个 hello.py"]
[assistant: ...]
[user: ...]
[assistant: TextBlock("hello.py 已创建完成")]
```

四条消息，user/assistant 严格交替。`stop_reason` 从 `"tool_use"` 变成 `"end_turn"`，循环退出。

---

## 小结

s01 的全部内容就这些：一个 while 循环 + 一个 bash 工具。代码不到 30 行，但这是后面 11 个 Session 的地基——**循环本身从 s01 到 s12 始终不变**，所有后续课程都是在这个循环之上叠加 Harness 机制。

下一篇我们看 s02：怎么在不改循环的前提下，给 Agent 加更多工具。

---

*本文是 learn-claude-code 系列学习笔记的第一篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s01 课程。*
