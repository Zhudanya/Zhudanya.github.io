---
title: "learn-claude-code 学习笔记（七）：Task System —— 大目标拆成小任务，记在磁盘上"
date: 2026-03-25 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, task-system]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第七篇。从这篇开始进入第三阶段——持久化与多 Agent 协作。s07 要解决的问题：s03 的 todo 只是内存里的扁平清单，压缩后就没了，而且不知道任务之间谁先谁后。

## 问题：s03 的 Todo 三个致命限制

1. **只在内存里**——s06 的上下文压缩一跑，todo 就没了。程序重启也没了
2. **扁平清单，没有依赖关系**——不知道"任务 B 要等任务 A 做完才能开始"
3. **分不清能做/被卡住**——只有做完/没做完，没有"被阻塞"的概念

真实目标是有结构的：任务 B 依赖任务 A，任务 C 和 D 可以并行，任务 E 要等 C 和 D 都完成。没有显式的依赖关系，模型分不清什么能做、什么被卡住、什么能同时跑。

## 解决方案：持久化的任务图

把扁平清单升级为持久化到磁盘的**任务图（DAG）**。每个任务是一个 JSON 文件，有状态、有依赖关系。

```
.tasks/
  task_1.json  {"id":1, "status":"completed", "blockedBy":[], "blocks":[2,3]}
  task_2.json  {"id":2, "status":"pending",   "blockedBy":[1], "blocks":[4]}
  task_3.json  {"id":3, "status":"pending",   "blockedBy":[1], "blocks":[4]}
  task_4.json  {"id":4, "status":"pending",   "blockedBy":[2,3]}

           +----------+
      +--> | task 2   | --+
      |    | pending  |   |
+-----+--+  +----------+   +-> +----------+
| task 1  |                     | task 4   |
|completed| -> +----------+ +-> | blocked  |
+---------+    | task 3   | -+  +----------+
               | pending  |
               +----------+

任务 1 完成后 → 任务 2 和 3 解锁，可以并行
任务 2 和 3 都完成后 → 任务 4 解锁
```

任务图随时回答三个问题：
- **什么能做？** — `blockedBy` 为空的 pending 任务
- **什么被卡住？** — `blockedBy` 不为空的任务
- **什么做完了？** — status 为 completed 的任务

### 这个方案怎么解决那三个问题

**问题 1：没有依赖关系 → blockedBy + blocks 建立依赖**

每个任务有两个字段表达关系：
- `blockedBy: [1]` — "任务 1 没做完，我不能开始"
- `blocks: [2, 3]` — "我没做完，任务 2 和 3 不能开始"

模型调 `task_list` 看到的是：

```
[ ] #1: Setup project
[ ] #2: Write code (blocked by: [1])
[ ] #3: Write tests (blocked by: [1])
[ ] #4: Deploy (blocked by: [2, 3])
```

一眼就能判断该做什么、什么被卡着。

**问题 2：分不清状态 → 完成时自动解锁后续任务**

任务 1 标为 completed 时，系统自动遍历所有任务，把 `blockedBy` 里包含 1 的全部移除：

```
完成前：task_2.blockedBy = [1]    → 被卡住
完成后：task_2.blockedBy = []     → 解锁了，可以做了
```

模型不需要自己记"任务 1 做完了该做 2 和 3"——调 task_list 就能看到最新状态。

**问题 3：压缩后丢失 → 存在磁盘上**

每个任务是 `.tasks/task_N.json` 文件。s06 的 auto_compact 压缩 messages，任务文件不受影响。程序重启，TaskManager 从文件名恢复 ID 计数，调 task_list 就能看到完整任务图。

**状态从"绑定在对话里"变成了"独立于对话之外"。**

---

## TaskManager 类详解

### 内部方法（和磁盘打交道）

**`__init__`** — 程序启动时执行一次。创建 `.tasks/` 目录（已存在就跳过），从现有文件里算出最大 ID，下一个任务从 max+1 开始编号。这就是为什么重启后不会 ID 冲突。

**`_load`** — 按 ID 读磁盘上的 JSON 文件，返回 Python 字典。文件不存在就抛异常。

**`_save`** — 把任务字典写入磁盘。`json.dumps(task, indent=2)` 格式化为可读的 JSON。

**`_max_id`** — 从文件名里提取 ID（`task_3.json` → 3），取最大值。目录为空返回 0。

**`_clear_dependency`** — 依赖解除的核心。遍历所有任务文件，把 `blockedBy` 里包含指定 ID 的全部移除。

### 公开方法（面向模型的工具）

**`create(subject, description)`** — 创建任务

```python
def create(self, subject: str, description: str = "") -> str:
    task = {
        "id": self._next_id, "subject": subject, "description": description,
        "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
    }
    self._save(task)
    self._next_id += 1
    return json.dumps(task, indent=2)
```

新任务默认 pending，依赖为空。`owner` 字段空着——后续 s09 多 Agent 才用到。返回完整 JSON 让模型确认。

**`get(task_id)`** — 查看任务详情，就是 `_load` 的包装。`list_all` 只显示摘要，`get` 返回完整内容。

**`update(task_id, status, add_blocked_by, add_blocks)`** — 最复杂的方法，干三件事：

第一件：**更新状态**。如果标为 completed，自动调 `_clear_dependency` 解锁后续任务。

第二件：**添加 blockedBy**。把新依赖合并进去，`set()` 去重。

第三件：**添加 blocks + 双向维护**。这是一个重要的设计——给任务 1 加 `blocks: [2, 3]`，代码会自动去任务 2 和 3 的文件里把 1 加到它们的 `blockedBy` 里：

```python
if add_blocks:
    task["blocks"] = list(set(task["blocks"] + add_blocks))
    for blocked_id in add_blocks:
        blocked = self._load(blocked_id)
        if task_id not in blocked["blockedBy"]:
            blocked["blockedBy"].append(task_id)
            self._save(blocked)
```

模型只需要说"任务 1 挡着 2 和 3"，反向关系自动建立。

**`list_all()`** — 列出所有任务

```python
[x] #1: Setup project
[>] #2: Write code
[ ] #3: Write tests (blocked by: [2])
[ ] #4: Deploy (blocked by: [2, 3])
```

和 s03 的 render 类似，但多了依赖信息。模型看到 `(blocked by: [2])` 就知道这个任务还不能开始。

### 方法调用关系

```
模型调工具
  ├── task_create  →  create()  →  _save()
  ├── task_get     →  get()     →  _load()
  ├── task_update  →  update()  →  _load()
  │                               →  _clear_dependency() → 遍历所有文件
  │                               →  双向维护 blocks/blockedBy
  │                               →  _save()
  └── task_list    →  list_all() →  读取所有文件
```

注意 update 不是轮询的——只在模型主动调 `task_update` 时才执行。没有定时器或后台线程。模型决定什么时候更新状态，Harness 负责自动维护依赖关系。

---

## 四个新工具

全部走标准 dispatch map，agent_loop 不需要特殊处理：

```python
TOOL_HANDLERS = {
    # ...基础工具...
    "task_create": lambda **kw: TASKS.create(kw["subject"], kw.get("description", "")),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status"), ...),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

agent_loop 回归了标准模式——和 s05 一样简洁，没有任何工具需要特殊分支处理。

---

## Claude Code 的任务系统

学到这里我好奇验证了一下 Claude Code 的实际存储。

Claude Code 的任务文件不在项目目录里，而是在用户的临时目录下：

```
C:\Users\{用户名}\AppData\Local\Temp\claude\{项目路径编码}\{会话ID}\tasks\
```

每个项目按路径编码成目录名，每个会话有独立的子目录。

另外有个容易混淆的点：**Plan 和 Task 不是一回事**。Claude Code 的 `/plan` 命令产生的是对话层面的规划（更接近 s03 的 todo，在上下文里），不会产生磁盘文件。s07 的 Task 系统才会写磁盘。Plan 随上下文压缩可能丢失，Task 持久存在。

---

## 为什么这课是后续的基础

文档里说这个任务图是"s07 之后所有机制的协调骨架"。后面的课程都在这个结构上叠加：

- **s08 后台执行** — 任务可以在后台线程里跑，完成后更新 task 状态
- **s09 多 Agent 团队** — 不同 Agent 认领不同的 task，通过任务文件协调
- **s12 Worktree 隔离** — 每个 task 绑定一个独立的 git worktree

任务图是这些机制的公共语言——所有 Agent 读写同一套 `.tasks/*.json` 文件，通过文件系统实现协调。

---

## 和 s03 的对比

| | s03 TodoManager | s07 TaskManager |
|---|---|---|
| 存储 | 内存（self.items） | 磁盘（.tasks/*.json） |
| 结构 | 扁平清单 | 带依赖的 DAG |
| 更新方式 | 全量替换 | CRUD |
| 依赖关系 | 无 | blockedBy + blocks |
| 压缩后 | 丢失 | 存活 |
| 重启后 | 丢失 | 存活 |
| 适用场景 | 单次会话的快速清单 | 跨会话的持久目标 |

下一篇我们看 s08：慢操作丢后台，Agent 继续想下一步——后台执行是怎么回事。

**Go 语言实现**：[s07/main.go](https://github.com/Zhudanya/learn-claude-code/tree/main/agents_go/s07/main.go)

---

*本文是 learn-claude-code 系列学习笔记的第七篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s07 课程。*
