---
title: "learn-claude-code 学习笔记（八）：Background Tasks —— 慢操作丢后台，Agent 继续干活"
date: 2026-03-26 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, background-tasks]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第八篇。s07 给了 Agent 持久化的任务图，这篇看 s08 怎么解决另一个效率问题：慢命令卡住整个循环。

## 问题：所有操作都是阻塞的

从 s01 到 s07，每次调工具都是阻塞的。核心是 `subprocess.run`：

```python
def run_bash(command: str) -> str:
    r = subprocess.run(command, shell=True, ...)   # ← 卡在这里，等命令跑完
    return (r.stdout + r.stderr).strip()
```

`subprocess.run` 是同步调用——Python 停在这一行，等 shell 命令跑完拿到输出，才往下走。

`ls` 跑一瞬间感觉不到。但 `npm install` 要 2 分钟，`docker build` 要 5 分钟，`pytest` 跑完整测试套件要好几分钟。这期间 agent_loop 完全卡住——模型不能思考、不能调其他工具、什么也做不了。

用户说"装依赖，顺便建个配置文件"，模型只能先等 `npm install` 跑完，再建配置文件。其实这两件事完全不相关，可以同时进行。

## 解决方案：后台线程 + 通知队列

```
主线程                          后台线程
+------------------+            +------------------+
| agent loop       |            | npm install 在跑  |
| 模型继续建配置文件 |            | ...               |
| ...              |            | 跑完了，结果入队    |
| [下一轮 LLM 调用] | <--------- | enqueue(result)  |
|  ^先排空通知队列   |            +------------------+
+------------------+
```

模型调 `background_run("npm install")` 后立刻拿到一个 task_id，可以继续做别的。命令在后台线程里跑，跑完后结果进入通知队列。每轮 LLM 调用前，agent_loop 排空队列，把结果注入 messages——模型在下一轮就能看到。

---

## BackgroundManager 类

### 初始化

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}                  # task_id → {status, result, command}
        self._notification_queue = []    # 已完成任务的结果
        self._lock = threading.Lock()    # 线程锁
```

`self.tasks` 存所有后台任务的状态。`_notification_queue` 存已完成但还没通知模型的结果。`_lock` 是线程锁——主线程和后台线程会同时操作队列，不加锁会出并发问题。

### run — 启动后台任务

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "result": None, "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True
    )
    thread.start()
    return f"Background task {task_id} started: {command[:80]}"
```

**调用时机**：模型调 `background_run` 工具时。

生成随机 ID，创建守护线程（`daemon=True` 表示主程序退出时自动杀掉），**立刻返回**不等命令跑完。模型拿到 "Background task a1b2c3d4 started: npm install" 后可以继续做别的。

### _execute — 后台线程的执行函数

```python
def _execute(self, task_id: str, command: str):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
                           capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
        status = "completed"
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
        status = "timeout"
    self.tasks[task_id]["status"] = status
    self.tasks[task_id]["result"] = output or "(no output)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "status": status,
            "command": command[:80],
            "result": (output or "(no output)")[:500],
        })
```

**调用时机**：不是模型调的，是后台线程启动后自动执行。

超时 300 秒（比普通 bash 的 120 秒长，因为后台本来就跑慢命令）。跑完后加锁把结果放进通知队列。通知里的 result 截断到 500 字符——只是摘要通知，模型想看完整结果可以调 `check_background`。

`with self._lock` 是关键——主线程可能正在 `drain_notifications` 读队列，后台线程同时在往队列里写，不加锁会数据竞争。

### drain_notifications — 排空通知队列

```python
def drain_notifications(self) -> list:
    with self._lock:
        notifs = list(self._notification_queue)
        self._notification_queue.clear()
    return notifs
```

**调用时机**：agent_loop 每轮循环开始时。

加锁，取出所有通知，清空队列，返回。"排空"而不是"取一个"——一次性把所有已完成的后台任务结果都拿走。

### check — 查看后台任务状态

```python
def check(self, task_id: str = None) -> str:
    if task_id:
        t = self.tasks.get(task_id)
        return f"[{t['status']}] {t['command'][:60]}\n{t.get('result') or '(running)'}"
    # 不传 task_id 就列出所有
    lines = []
    for tid, t in self.tasks.items():
        lines.append(f"{tid}: [{t['status']}] {t['command'][:60]}")
    return "\n".join(lines)
```

**调用时机**：模型调 `check_background` 工具时。传 ID 看单个任务详情，不传就列出所有。

---

## agent_loop 的变化

核心变化在循环开头——每次调 LLM 之前先检查后台任务：

```python
def agent_loop(messages: list):
    while True:
        # ===== LLM 调用前：排空通知，注入结果 =====
        notifs = BG.drain_notifications()
        if notifs and messages:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
            )
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        # ===== 注入完毕，调 LLM =====
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages, ...)
```

**执行顺序**：drain → 有通知就注入 messages → 调 LLM → 模型看到后台结果。

注入的是一对 user/assistant 消息——因为 API 要求严格交替。模型在这一轮就能看到 `<background-results>` 里的内容，知道哪些后台任务跑完了。

---

## bash 和 background_run 的区别

| | bash | background_run |
|---|---|---|
| 阻塞 | 是，等命令跑完 | 否，立刻返回 task_id |
| 超时 | 120 秒 | 300 秒 |
| 结果怎么拿 | 直接作为 tool_result 返回 | 下一轮循环时通过通知注入 |
| 适合什么 | 瞬间完成的命令（ls、cat、echo） | 慢命令（npm install、pytest、docker build） |

两个都在。不是所有命令都该放后台——`ls` 放后台反而更慢（多了线程开销和一轮循环延迟）。模型根据 description 自己判断用哪个。

---

## 完整的时间线

用户说"装依赖，顺便建个配置文件"：

```
轮次 1：
  drain_notifications() → 空（没有后台任务）
  模型调 background_run("npm install")
    → 后台线程启动，立刻返回 "task a1b2c3d4 started"
  模型调 write_file("config.json", "...")
    → 直接写文件，返回 "Wrote 128 bytes"
  模型回复："依赖在后台安装中，配置文件已创建。"

（npm install 在后台跑了 2 分钟...跑完了，结果入队）

轮次 2：（用户说了新的话）
  drain_notifications()
    → [{task_id: "a1b2c3d4", status: "completed", result: "added 500 packages"}]
  注入 <background-results> 到 messages
  模型看到 npm install 完成了
  模型回复："依赖安装完成，共 500 个包。"
```

不用等 2 分钟。模型在 npm install 跑着的时候就建好了配置文件。

---

## 小结

s08 是第一次引入并发。之前所有课程都是严格单线程：调工具 → 等结果 → 喂回模型 → 调工具 → 等结果，一步一步来。s08 把"等结果"这一步从主线程拆到了后台线程。

核心设计：
- **主线程**负责思考和决策（LLM 调用）
- **后台线程**负责等待（subprocess 执行）
- **通知队列**是两者之间的通信通道——后台线程往里放，主线程每轮取

循环始终是单线程的。只有 subprocess I/O 被并行化了。Agent 可以"一边想一边等"，而不是"想完等、等完想"。

下一篇进入第四阶段——s09：任务太大一个人干不完，要能分给队友。

**Go 语言实现**：[s08/main.go](https://github.com/Zhudanya/learn-claude-code/tree/main/agents_go/s08/main.go)

---

*本文是 learn-claude-code 系列学习笔记的第八篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s08 课程。*
