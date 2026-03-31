---
title: "learn-claude-code 学习笔记（十）：Team Protocols —— 队友之间要有统一的沟通规矩"
date: 2026-03-28 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, team-protocols]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第十篇。s09 搭好了团队的基础设施（身份、线程、邮箱），这篇看 s10 怎么在通信通道上加结构化的协调协议。

## 问题：s09 缺少结构化协调

s09 的队友能干活能通信，但有两个场景处理不了：

**关机问题**：s09 的队友一旦 spawn 出去就是"放养"状态——要么自己干完停下来，要么跑满 50 轮。领导没有任何方式主动让一个正在干活的队友停下来。

什么时候需要叫停？方向错了（应该改 B 模块不是 A）、任务取消了（用户改主意了）、资源冲突了（两个人改同一个文件）、已经被别人做完了。这些场景的共同点：领导获得了新信息，需要改变队友的行为。

直接杀线程不行——队友可能写了一半文件。需要一个握手：领导说"请关机"，队友说"好的我收尾一下"或者"不行我正在写文件"。

**计划审批问题**：领导说"重构认证模块"，队友立刻开干。但高风险变更应该先过审——队友先提交计划，领导审批通过了再做。

## 解决方案：request-response 模式

两种协议，结构完全一样——一方发带唯一 ID 的请求，另一方引用同一 ID 响应：

```
关机协议：
  领导 --shutdown_request(req_id="abc")--> 队友
  队友 --shutdown_response(req_id="abc", approve=true)--> 领导

计划审批协议：
  队友 --plan_approval(plan="重构方案...")--> 领导（生成 req_id="xyz"）
  领导 --plan_approval(req_id="xyz", approve=true)--> 队友
```

---

## 状态机：一个模式，两种用途

两种协议共享同一个状态变化流程：

```
[pending] --approve--> [approved]
[pending] --reject---> [rejected]
```

关机和计划审批的流转完全一样：

```
关机：领导发请求 → pending → 队友批准 → approved / 拒绝 → rejected
审批：队友提交计划 → pending → 领导批准 → approved / 拒绝 → rejected
```

都是"一方发起，另一方回应同意或拒绝"。不需要两套不同的流程，一套 `pending → approved/rejected` 就够了。

---

## Tracker 字典：追踪请求状态

```python
shutdown_requests = {}   # {req_id: {"target": "alice", "status": "pending"}}
plan_requests = {}       # {req_id: {"from": "alice", "plan": "...", "status": "pending"}}
_tracker_lock = threading.Lock()
```

**为什么需要 tracker？** 邮箱是 drain 模式——读完就清空。领导发了关机请求，队友的回应通过邮箱回来，但领导读完就没了。领导怎么查"我之前发的那个关机请求，alice 同意了没有"？

tracker 字典就是一个**工单系统**——提了工单（pending），对方处理完了更新状态（approved/rejected），随时能查进度。邮箱负责传递消息，tracker 负责记录状态。

**为什么需要锁？** 主线程（领导）和队友线程同时读写同一个字典。不加锁的话，一个线程写到一半另一个来读，可能拿到不一致的数据。`_tracker_lock` 保证同一时间只有一个线程能操作字典。

---

## 关机协议的完整流程

### 1. 领导发起关机请求

```python
def handle_shutdown_request(teammate):
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
```

生成唯一 `req_id`，记入 tracker（pending），通过邮箱发给队友。

### 2. 队友收到请求，决定是否同意

队友的循环读到收件箱里的关机请求，模型判断后调 `shutdown_response`：

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response", {"request_id": req_id, "approve": approve})
```

更新 tracker 状态，通过邮箱通知领导。

### 3. 队友同意后优雅退出

```python
if block.name == "shutdown_response" and block.input.get("approve"):
    should_exit = True    # 标记退出，不是立刻杀线程

# 循环结束时
member["status"] = "shutdown" if should_exit else "idle"
```

不是直接 break——先标记 `should_exit = True`，等本轮工具执行完、结果追加到 messages 后，下一轮循环开头才 break。保证当前操作完整完成。

退出后 config.json 里状态是 `"shutdown"`，不是 `"idle"`——区分"正常完成"和"被请求关机"。

---

## 计划审批的完整流程

### 1. 队友提交计划

```python
if tool_name == "plan_approval":
    req_id = str(uuid.uuid4())[:8]
    plan_requests[req_id] = {"from": sender, "plan": plan_text, "status": "pending"}
    BUS.send(sender, "lead", plan_text, "plan_approval_response",
             {"request_id": req_id, "plan": plan_text})
```

队友自己生成 `req_id`（不是领导生成），记入 plan_requests，通过邮箱发给领导。

### 2. 领导审批

```python
def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback, "plan_approval_response",
             {"request_id": request_id, "approve": approve, "feedback": feedback})
```

引用同一个 `req_id`，更新 tracker，通过邮箱把结果和反馈发给队友。

### 3. 队友收到结果

队友下一轮循环读到审批结果，继续执行（approved）或调整方案（rejected + feedback）。

---

## request_id 是关键

领导可能同时对多个队友发关机请求，多个队友同时提交计划。怎么把请求和响应对上号？靠 `request_id`。

```
shutdown_requests = {
    "abc": {"target": "alice", "status": "approved"},
    "def": {"target": "bob", "status": "pending"},
}

plan_requests = {
    "xyz": {"from": "alice", "plan": "重构方案", "status": "pending"},
    "uvw": {"from": "bob", "plan": "测试方案", "status": "approved"},
}
```

每个请求有唯一 ID，响应时引用同一个 ID。不管同时有多少个请求在 pending，都不会混淆。

---

## 同名工具在领导和队友端的实现不同

| 工具名 | 领导端 | 队友端 |
|--------|--------|--------|
| `shutdown_request` | 向队友发起关机请求 | ❌ 没有 |
| `shutdown_response` | 查询关机请求状态 | 回应关机请求（approve/reject） |
| `plan_approval` | 审批队友提交的计划 | 向领导提交计划 |

同一个工具名，在不同角色手里做不同的事。这是因为领导和队友在协议中的角色不同——关机协议中领导是发起方、队友是响应方；计划审批中反过来。

---

## 队友循环的变化

对比 s09，`_teammate_loop` 多了：

1. **`should_exit` 标记**——同意关机时不立刻退出，等当前操作完成
2. **退出状态区分**——`"shutdown"` vs `"idle"`
3. **system prompt 加了协议规范**——"Submit plans via plan_approval before major work. Respond to shutdown_request with shutdown_response."
4. **多了两个工具**——`shutdown_response` 和 `plan_approval`

---

## 和 s09 的变更对比

| 组件 | s09 | s10 |
|------|-----|-----|
| 工具数量 | 9（领导）/ 6（队友） | 12（领导）/ 8（队友） |
| 关机 | 自然退出（跑完变 idle） | 请求-响应握手 |
| 计划管控 | 无 | 队友提交，领导审批 |
| 请求关联 | 无 | request_id |
| 状态追踪 | 无 | tracker 字典 + 锁 |
| 退出状态 | 只有 idle | idle 或 shutdown |

## 小结

s10 在 s09 的通信通道上加了**结构化协议**。s09 的消息是自由文本，没有约束。s10 定义了两种正式的交互模式，都用同一个 `pending → approved/rejected` 状态机。

这个模式是可扩展的——任何"一方请求、另一方回应"的场景（代码审查、权限申请、任务交接……）都可以套用同样的 request_id + FSM。

下一篇看 s11：队友自己看看板，有活就认领——自治 Agent。

**Go 语言实现**：[s10/main.go](https://github.com/Zhudanya/learn-claude-code/tree/main/agents_go/s10/main.go)

---

*本文是 learn-claude-code 系列学习笔记的第十篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s10 课程。*
