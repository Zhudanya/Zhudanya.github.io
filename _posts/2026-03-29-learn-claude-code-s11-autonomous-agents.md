---
title: "learn-claude-code 学习笔记（十一）：Autonomous Agents —— 队友自己看看板，有活就认领"
date: 2026-03-29 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, autonomous-agents]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第十一篇。s10 给团队加了结构化协议（关机握手 + 计划审批），这篇看 s11 怎么让队友从"被指派"变成"自组织"。

## 问题：领导变成瓶颈

s09-s10 中，队友只在被领导明确指派时才动。领导得给每个队友写 prompt，任务看板上 10 个未认领的任务得手动一个个分配。

而且 s09 有一个我们讨论过的痛点：bob 等 alice 写完再测试，但 bob 的循环可能已经结束了——`for _ in range(50)` 跑完变 idle，线程退出，alice 后来发的消息躺在 `bob.jsonl` 里没人读。

## 解决方案：WORK + IDLE 两阶段无限循环

s09-s10 的队友循环是单阶段的——干完就退。s11 改成了两阶段无限循环：

```
+-------+
| spawn |
+---+---+
    |
    v
+-------+  tool_use    +-------+
| WORK  | <----------- |  LLM  |    ← 正常 agent loop，最多 50 轮
+---+---+              +-------+
    |
    | 模型不再调工具（或调了 idle 工具）
    v
+--------+
| IDLE   | 每 5 秒轮询，最多 60 秒
+---+----+
    |
    +--→ 检查收件箱 → 有消息？ → 回到 WORK
    |
    +--→ 扫描 .tasks/ → 有未认领任务？ → 认领 → 回到 WORK
    |
    +--→ 60 秒超时 → SHUTDOWN
```

关键变化：**外层从 `for _ in range(50)` 变成了 `while True`。** 工作阶段结束后不是退出，而是进入空闲轮询。找到活就回去工作，60 秒没活才真正关机。

---

## 解决了 s09 的痛点

```
s09 的 bob：
  干活 → 没活了 → idle → 线程结束 → alice 的消息没人读

s11 的 bob：
  干活 → 没活了 → IDLE（每 5 秒查收件箱和任务看板）
                    ↓
              alice 发了消息 → bob 读到 → 回到 WORK
              或者 alice 完成任务 → 新任务解锁 → bob 扫描到 → 认领 → 回到 WORK
```

现在有两种方式让 bob 知道"可以开始了"：

1. **消息通知**——alice 发 `send_message`，bob 轮询时读到收件箱
2. **任务看板**——alice 完成任务后 s07 的 `_clear_dependency` 自动解锁下游任务，bob 扫描到未认领任务并认领

第二种方式不需要 alice 主动通知——任务图的依赖解除机制自动传递了"alice 做完了"这个信息。

---

## IDLE 阶段：_idle_poll

```python
POLL_INTERVAL = 5    # 每 5 秒查一次
IDLE_TIMEOUT = 60    # 最多等 60 秒

def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):   # 60/5 = 12 次
        time.sleep(POLL_INTERVAL)

        # 1. 检查收件箱
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append(...)
            return True                    # 有消息，回去工作

        # 2. 扫描任务看板
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append(...)
            return True                    # 有任务，回去工作

    return False                           # 60 秒没活，关机
```

每 5 秒查两个地方：收件箱和任务看板。找到活返回 True（回到 WORK），60 秒内都没找到返回 False（shutdown）。

60 秒是教学演示值。生产环境可以调大，或者改成无限等待。这是一个取舍——太短容易因为上游慢而提前退出，太长则空闲队友占着资源。

---

## 任务看板扫描：找什么样的任务

```python
def scan_unclaimed_tasks():
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"       # 状态是 pending
                and not task.get("owner")          # 没有人认领
                and not task.get("blockedBy")):    # 没有被阻塞
            unclaimed.append(task)
    return unclaimed
```

三个条件全满足才算"可认领"。和 s07 的任务图配合——被阻塞的任务不会被错误认领。

## 认领时加锁

```python
_claim_lock = threading.Lock()

def claim_task(task_id, owner):
    with _claim_lock:
        task = json.loads(path.read_text())
        task["owner"] = owner
        task["status"] = "in_progress"
        path.write_text(json.dumps(task))
```

新增的 `_claim_lock`——防止 alice 和 bob 同时扫描到同一个未认领任务，同时去认领。加锁保证一个任务只能被一个人认领。

---

## idle 工具：模型主动进入空闲

```python
{"name": "idle", "description": "Signal that you have no more work. Enters idle polling phase."}
```

模型调 `idle` 工具时，WORK 阶段立刻 break，进入 IDLE 轮询。和 s06 的 compact 工具、s10 的 shutdown_response 思路一样——给模型主动触发状态转换的权利。

不调 idle 也行——`stop_reason != "tool_use"` 时也会退出 WORK 阶段。idle 工具是让模型能更明确地表达"我现在没活了，去等新活"。

---

## 身份重注入：压缩后不忘记自己是谁

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": "<identity>You are 'alice', role: coder, team: default.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": "I am alice. Continuing."})
```

自治队友可能长时间运行，上下文会被压缩（s06）。压缩后 messages 可能只剩两三条摘要，队友的身份信息被压缩掉了——模型不知道自己叫什么、什么角色。

`len(messages) <= 3` 是一个简单的启发式判断："messages 这么短，大概率是被压缩过了"。此时在开头注入身份块，让模型重新知道"我是谁"。

---

## 领导的角色变了

| | s09-s10 领导 | s11 领导 |
|---|---|---|
| 分配任务 | 逐个给队友写 prompt | 往任务看板上放任务 |
| 队友认领 | 领导指定 | 队友自动扫描认领 |
| 瓶颈 | 领导是瓶颈 | 任务看板是公共资源 |

领导从"逐个分配"变成了"创建任务放看板"，队友自己去看板找活。这就是文档说的"自组织"。

---

## 和 s10 的变更对比

| 组件 | s10 | s11 |
|------|-----|-----|
| 工具数量 | 12 / 8 | 14 / 10（+idle +claim_task） |
| 队友循环 | 单阶段（干完退出） | 两阶段（WORK + IDLE，while True） |
| 任务认领 | 领导手动分配 | 队友自动扫描看板 |
| 空闲行为 | 变 idle 线程结束 | 每 5 秒轮询，有活就继续 |
| 超时 | 无 | 60 秒没活自动 shutdown |
| 身份 | system prompt | + 压缩后重注入 |
| 新增的锁 | 无 | _claim_lock 防止并发认领 |

## 小结

s11 的核心变化就一个：队友的外层循环从 `for` 变成了 `while True`，中间加了一个 IDLE 轮询阶段。这个看似简单的变化，解决了 s09 的"队友退出后消息没人读"问题，也让团队从"领导逐个分配"升级到了"队友自组织"。

下一篇是最后一课 s12：各干各的目录，互不干扰——Worktree 隔离。

---

*本文是 learn-claude-code 系列学习笔记的第十一篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s11 课程。*
