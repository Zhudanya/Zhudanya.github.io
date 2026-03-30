---
title: "learn-claude-code 学习笔记（九）：Agent Teams —— 从单兵作战到团队协作"
date: 2026-03-27 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, agent-teams]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第九篇。从这篇开始进入第四阶段——多 Agent 团队协作。s09 要解决的问题：一个 Agent 干不完的活，怎么分给多个队友。

## 问题：子 Agent 是一次性的，后台任务不会思考

s04 的子 Agent：生成、干活、返回摘要、消亡。没有身份，没有跨调用的记忆，不能和其他 Agent 通信。

s08 的后台任务：能并行跑 shell 命令，但只是执行预定义命令，不能做 LLM 引导的决策。

真正的团队协作需要三样东西：
1. **持久的身份**——队友有名字、有角色、有生命周期
2. **独立的 Agent Loop**——每个队友跑自己的循环，能独立思考和用工具
3. **通信通道**——队友之间能互相发消息

## 解决方案：TeammateManager + MessageBus

两个核心组件，通过文件系统协调：

```
.team/
  config.json           ← 团队名册（谁、什么角色、什么状态）
  inbox/
    alice.jsonl         ← alice 的收件箱
    bob.jsonl           ← bob 的收件箱
    lead.jsonl          ← 领导的收件箱
```

---

## Agent 之间怎么通信

没有任何高级的进程间通信机制。就是**往文件里写一行、从文件里读一行**：

```
alice 想给 bob 发消息
  → BUS.send("alice", "bob", "代码写完了")
  → 打开 bob.jsonl，追加一行：
    {"from":"alice", "content":"代码写完了", "timestamp":1711800000}

bob 下一轮循环开始前
  → inbox = BUS.read_inbox("bob")
  → 读取 bob.jsonl 所有行，解析成列表，清空文件
  → 注入 bob 的 messages
  → bob 的模型看到消息，知道 alice 写完了
```

**写文件 = 发消息，读文件 = 收消息。**

读完立刻清空（drain 模式），保证同一条消息不会被读两次。和 s08 的 drain_notifications 是同一个思路。

为什么用文件而不是队列、socket？文件天然持久化（崩了消息还在），简单（不需要中间件），可观测（打开 JSONL 就能看到所有消息），而且跨进程兼容（后续课程队友可能不在同一个进程里）。

---

## 每个队友都是一个线程

领导的模型决定调 `spawn_teammate` 时，创建一个守护线程，跑队友自己的 agent loop：

```python
def spawn(self, name, role, prompt):
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()                    # 先存盘
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()                         # 再启动线程
```

先存 config.json 再启动线程——如果线程跑了一半程序崩了，至少 config 里有这个队友的记录。

创建后同时跑多个线程：

```
主线程（领导）      alice 线程          bob 线程
领导的 loop        alice 的 loop      bob 的 loop
自己的 messages     自己的 messages     自己的 messages
自己的 system       自己的 system       自己的 system
9 个工具            6 个工具            6 个工具
```

每个线程里是一个完整的 agent loop，和 s01 的核心循环一模一样。共享文件系统和 API 客户端，不共享 messages 和 system prompt。

## 队友的循环

```python
def _teammate_loop(self, name, role, prompt):
    sys_prompt = f"You are '{name}', role: {role}, at {WORKDIR}. ..."
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):                         # 安全上限，不是 while True
        inbox = BUS.read_inbox(name)            # 每轮检查收件箱
        for msg in inbox:
            messages.append({"role": "user", "content": json.dumps(msg)})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # 执行工具...
    member["status"] = "idle"                   # 循环结束，标为空闲
    self._save_config()                         # 状态写盘
```

用 `for _ in range(50)` 而不是 `while True`——和 s04 子 Agent 的 `range(30)` 同样的安全考虑，防止后台线程失控。

config.json 写两次：spawn 时（status: working），循环结束时（status: idle）。

---

## 领导和队友的权限差异

队友的权限比领导少：

| 能力 | 领导 | 队友 |
|------|------|------|
| bash / read / write / edit | ✅ | ✅ |
| send_message | ✅ | ✅ |
| read_inbox | ✅ | ✅ |
| **spawn_teammate** | ✅ | ❌ |
| **list_teammates** | ✅ | ❌ |
| **broadcast** | ✅ | ❌ |

队友不能创建新队友——和 s04 子 Agent 没有 task 工具同一个设计。防止 alice 创建 alice-helper-1、alice-helper-2……无限递归。只有领导能创建队友，保证团队结构可控。

---

## 完整的协作流程

用户说"让 alice 写代码，bob 测试"：

```
1. 领导的模型决定调 spawn_teammate
   → alice 线程启动，config.json: alice status = "working"
   → bob 线程启动，config.json: bob status = "working"

2. alice 的循环里，模型写完代码后调 send_message
   → bob.jsonl 追加：{"from":"alice", "content":"登录模块写完了，请测试"}

3. bob 下一轮循环，read_inbox 读到 alice 的消息
   → bob 的模型开始测试

4. bob 测试完，调 send_message(to="lead", content="测试通过")
   → lead.jsonl 追加一行

5. 领导下一轮循环，read_inbox 读到 bob 的消息
   → 领导知道测试通过了
```

所有通信都是异步的——发消息只是往文件追加一行，不阻塞。接收方在下一轮循环时才读到。

---

## 和 s04 子 Agent 的对比

| | s04 子 Agent | s09 队友 |
|---|---|---|
| 生命周期 | 调一次就消亡 | spawn → working → idle，可再次激活 |
| 身份 | 匿名 | 有名字、有角色 |
| 通信 | 不能，只返回摘要 | JSONL 邮箱，任意人→任意人 |
| 线程跑什么 | 完整 loop（和队友一样） | 完整 loop |
| 状态持久化 | 无 | config.json |
| 循环上限 | 30 轮 | 50 轮 |

---

## s09 的局限

通信通道建起来了，但协调还比较原始。比如"bob 等 alice 写完再测试"——如果 alice 还没写完，bob 的循环可能就已经跑完变 idle 了，alice 后来发的消息躺在 bob.jsonl 里没人读。

要真正解决这种协调问题，需要后续课程：
- **s10**——结构化的请求-响应协议
- **s11**——自治 Agent，空闲时自动轮询，有活就认领

s09 先把团队的基础设施搭好：身份、线程、邮箱。后续课程在这个基础上加协调机制。

---

## 小结

s09 从单兵作战升级到团队协作。核心就两件事：

1. **TeammateManager** — 管理谁在团队里、什么状态（config.json）
2. **MessageBus** — 管理谁给谁说了什么（inbox/*.jsonl）

所有协调通过文件系统实现——这是项目反复出现的模式。s07 的任务用 JSON 文件，s08 的后台输出写文件，s09 的通信用 JSONL 文件。文件系统就是这些 Agent 的"共享内存"。

下一篇看 s10：队友之间要有统一的沟通规矩——团队协议。

---

*本文是 learn-claude-code 系列学习笔记的第九篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s09 课程。*
