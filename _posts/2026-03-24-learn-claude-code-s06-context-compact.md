---
title: "learn-claude-code 学习笔记（六）：Context Compact —— 上下文总会满，要有办法腾地方"
date: 2026-03-24 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, context-compact]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第六篇。前面几课分别解决了规划（s03 todo）、噪声隔离（s04 subagent）、知识加载（s05 skill）的问题，但有一个根本问题一直没正面解决：**上下文窗口是有限的，messages 会一直增长。**

## 问题

读一个 1000 行的文件就吃掉 ~4000 token。读 30 个文件、跑 20 条命令，轻松突破 100k token。不压缩，Agent 在大项目里干着干着就把上下文撑爆了——还没做完就没空间了。

s03 的 todo 减缓了注意力稀释，s04 的子 Agent 减少了噪声堆积，但只要对话持续，messages 就会一直增长。s06 直接面对这个问题：**主动压缩上下文，换来无限会话。**

## 解决方案：三层压缩

三层策略，激进程度递增：

```
每轮循环开始时：
  [第一层 micro_compact]   ← 静默执行，每轮都跑
    把 3 轮前的旧 tool_result 替换成一句话占位符
         |
  [检查：token > 50000？]
    否 → 继续正常工作
    是 → [第二层 auto_compact]   ← 自动触发
           保存完整对话到磁盘
           LLM 做摘要
           用摘要替换全部 messages
         |
工具执行时：
  模型调了 compact 工具？
    是 → [第三层 手动 compact]   ← 模型主动触发
           和 auto_compact 一样的机制
```

---

## 第一层：micro_compact —— 日常清洁

**什么时候执行**：每轮循环开始时，在调 API 之前。静默执行，模型感知不到。

**干了什么**：把 3 轮前的旧 tool_result 替换成占位符。

```python
# 替换前（read_file 返回的几百行代码）
{"type": "tool_result", "content": "import os\nimport sys\ndef main():\n    ...(500 行)"}

# 替换后（一句话）
{"type": "tool_result", "content": "[Previous: used read_file]"}
```

**为什么是 3 轮前？**

模型的典型工作链是"读取 → 分析 → 操作"：

```
第 1 轮：read_file("utils.py")     ← 需要完整内容
第 2 轮：分析代码，决定改哪里       ← 可能还要回看
第 3 轮：edit_file("utils.py"...)   ← 可能还需要参考
第 4 轮：开始做别的事了             ← 第 1 轮的结果可以压缩了
```

3 轮之后，模型大概率已经用完了那个工具结果，不再需要完整内容。

**为什么替换成占位符而不是直接删除？**

API 要求 tool_result 必须和前面 assistant 消息里的 tool_use 一一对应。删了 tool_result，API 会报错。占位符保持了结构完整性（tool_use_id 还在），但把几千 token 缩成了几个 token。

**只压缩超过 100 字符的结果。** `"Edited utils.py"` 这种短结果本身就几个 token，压不压没区别。只有大的结果（读文件返回的几百行代码、bash 输出的大段日志）才值得压。

**效果**：假设读了 10 个文件，每个平均 2000 token。保留最近 3 个，压缩 7 个，节省约 14000 token。

---

## 第二层：auto_compact —— 紧急清理

**什么时候执行**：`estimate_tokens(messages) > 50000` 时自动触发。

**干了三步：**

### Step 1：保存完整对话到磁盘

```python
transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
with open(transcript_path, "w") as f:
    for msg in messages:
        f.write(json.dumps(msg, default=str) + "\n")
```

在 `.transcripts/` 目录下创建带时间戳的文件，完整保存对话。压缩是有损的——摘要不可能保留所有细节。这个文件是安全网，万一丢了关键信息还能恢复。

### Step 2：让 LLM 做摘要

```python
response = client.messages.create(
    model=MODEL,
    messages=[{"role": "user", "content":
        "Summarize this conversation for continuity. Include: "
        "1) What was accomplished, 2) Current state, 3) Key decisions made. "
        "Be concise but preserve critical details.\n\n" + conversation_text}],
    max_tokens=2000,
)
```

单独发一次 API 调用（不走 agent_loop），让模型总结之前的对话。为什么用 LLM 而不是代码做？因为代码不知道什么信息重要——50000 token 的对话里，哪些是关键决策、哪些是临时过程，只有理解语义的 LLM 能判断。

### Step 3：用摘要替换全部 messages

```python
return [
    {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
    {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
]
```

50000 token → ~2000 token。整个对话历史变成两条消息。压缩后的消息里还保留了 transcript 文件路径——模型以后可以 read_file 读这个文件来回忆细节。

**一个重要的语法细节**：循环里用的是 `messages[:] = auto_compact(messages)` 而不是 `messages = auto_compact(messages)`。messages 是外部传进来的引用，`messages[:]` 原地替换列表内容，外部变量也能看到变化；`messages =` 只改了局部变量的指向，外部不受影响。

---

## 第三层：compact 工具 —— 主动断舍离

**什么时候执行**：模型主动调用 compact 工具时。

**和 auto_compact 有什么区别？** 压缩机制完全一样，区别在于触发方式：

- auto_compact：token 计数超过 50000 自动触发
- compact 工具：模型自己觉得该清理了，主动触发

**为什么需要手动触发？** 两种场景 token 计数帮不上：

1. **上下文没满但已经很乱**——做了很多失败的尝试，上下文里充满了无关信息，虽然 token 数没到 50000，但噪声已经严重干扰了判断
2. **想主动翻篇**——完成了一个大阶段，想从干净的状态开始下一阶段

compact 工具给了模型这个自主权。

---

## 三层的关系

| | 第一层 micro | 第二层 auto | 第三层 manual |
|---|---|---|---|
| 触发方式 | 每轮自动 | token > 50000 | 模型主动调用 |
| 激进程度 | 温和 | 激进 | 激进 |
| 压缩什么 | 旧 tool_result 内容 | 整个 messages | 整个 messages |
| 压缩成什么 | 一句话占位符 | LLM 摘要 | LLM 摘要 |
| 信息损失 | 小 | 大（靠摘要保留关键点） | 大 |
| 备份 | 不需要 | 保存到 .transcripts/ | 保存到 .transcripts/ |

第一层是日常清洁，持续发挥作用，延缓上下文增长。第二层是紧急措施，快满了才用。第三层是模型的自主选择——觉得乱了就主动清理。

---

## agent_loop 怎么变的

核心结构不变，但前面插入了压缩逻辑，工具执行部分增加了 compact 的特殊处理：

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # 第一层：每轮清理
        if estimate_tokens(messages) > THRESHOLD:      # 第二层：超阈值压缩
            messages[:] = auto_compact(messages)
        response = client.messages.create(...)         # 正常调模型
        messages.append(...)
        if response.stop_reason != "tool_use":
            return
        # 工具执行...
        manual_compact = False
        for block in response.content:
            if block.name == "compact":
                manual_compact = True                  # 先标记，不立刻压缩
                output = "Compressing..."
            else:
                ...正常执行工具...
        messages.append({"role": "user", "content": results})
        if manual_compact:                             # 第三层：本轮结束后再压缩
            messages[:] = auto_compact(messages)
```

compact 工具为什么不在遍历 block 时立刻压缩？因为那会导致 messages 在遍历途中被替换——先标记，等所有工具结果追加完毕后再执行。

---

## 小结

s06 解决的是上下文的**物理限制**。前面几课的机制（todo、subagent、skill）都在优化上下文的**质量**，但 s06 解决的是**容量**问题——不管优化得多好，对话足够长就会撑满。

三层压缩的类比：
- 第一层像**整理桌面**——把不用的旧文件收起来，只留最近在看的
- 第二层像**年终归档**——把全年的文件打包存库房，桌上只留一份摘要
- 第三层像**主动断舍离**——自己觉得该清理了，手动触发

> Agent 可以策略性地遗忘，然后永远工作下去。

下一篇我们进入第三阶段——s07：大目标要拆成小任务，排好序，记在磁盘上。

---

*本文是 learn-claude-code 系列学习笔记的第六篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s06 课程。*
