---
title: "learn-claude-code 学习笔记（十二）：Worktree + Task Isolation —— 各干各的目录，互不干扰"
date: 2026-03-30 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, worktree, task-isolation]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的最后一篇。s11 让队友自组织，但所有人共享一个目录。s12 给每个任务一个独立的工作空间——从此并行不冲突。

## 问题：共享目录导致文件冲突

到 s11，Agent 已经能自主认领和完成任务了。但所有 Agent 共享一个工作目录。两个 Agent 同时重构不同模块——A 改 `config.py`，B 也改 `config.py`，未提交的改动互相污染，谁也没法干净回滚。

任务板管"做什么"，但不管"在哪做"。

## 解决方案：控制面 + 执行面分离

```
控制面（.tasks/）                执行面（.worktrees/）
+-------------------+           +------------------------+
| task_1.json       |           | auth-refactor/         |
|   status: in_progress <---->    branch: wt/auth-refactor
|   worktree: "auth-refactor"    task_id: 1             |
+-------------------+           +------------------------+
| task_2.json       |           | ui-login/              |
|   status: pending    <---->     branch: wt/ui-login
|   worktree: "ui-login"        task_id: 2             |
+-------------------+           +------------------------+
                                  index.json（注册表）
                                  events.jsonl（事件流）
```

- **任务**是控制面——管目标、状态、谁在做
- **Worktree**是执行面——管目录、分支、在哪做
- **task_id** 把两边关联起来

---

## 什么是 git worktree

Git 的一个功能：在同一个仓库下创建多个工作目录，每个有独立的分支和暂存区：

```bash
git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
```

执行后 `.worktrees/auth-refactor/` 就是一个完整的工作目录，有自己的分支 `wt/auth-refactor`，文件改动不影响主目录。两个 Agent 各自在自己的 worktree 里改 `config.py`，互不干扰。

---

## 三个新组件

### EventBus——生命周期事件日志

```python
class EventBus:
    def emit(self, event, task=None, worktree=None, error=None):
        # 追加一行到 events.jsonl
    def list_recent(self, limit=20):
        # 读最近 N 条事件
```

每个重要操作都写一条事件到 `.worktrees/events.jsonl`：

```json
{"event": "worktree.create.after", "task": {"id": 1}, "worktree": {"name": "auth-refactor", "status": "active"}, "ts": 1730000000}
{"event": "task.completed", "task": {"id": 1, "status": "completed"}, "worktree": {"name": "auth-refactor"}, "ts": 1730001000}
{"event": "worktree.remove.after", "task": {"id": 1}, "worktree": {"name": "auth-refactor", "status": "removed"}, "ts": 1730001001}
```

崩溃后可以从事件流推断发生了什么——哪些 worktree 创建了没删除，哪些任务开始了没完成。

### TaskManager（增强版）——多了 worktree 绑定

比 s07 多了两个方法：

```python
def bind_worktree(self, task_id, worktree):
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"    # 绑定时自动推进状态

def unbind_worktree(self, task_id):
    task["worktree"] = ""
```

任务和 worktree 双向绑定——任务记录"我在哪个 worktree 里做"，worktree 记录"我为哪个任务服务"。

### WorktreeManager——worktree 的完整生命周期

```python
class WorktreeManager:
    def create(self, name, task_id=None):    # 创建目录 + 绑定任务
    def run(self, name, command):            # 在 worktree 目录里跑命令
    def keep(self, name):                    # 标记保留（不删除）
    def remove(self, name, complete_task):   # 删除目录 + 可选完成任务
    def list_all(self):                      # 列出所有 worktree
    def status(self, name):                  # git status
```

`run` 方法是关键——`subprocess.run(command, cwd=worktree_path)`，命令在隔离目录里执行，不影响主目录。

---

## 完整的工作流

```
1. 创建任务
   task_create("Implement auth refactor")
   → task_1.json: status=pending, worktree=""

2. 创建 worktree 并绑定
   worktree_create("auth-refactor", task_id=1)
   → git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
   → index.json 新增条目
   → task_1.json: status=in_progress, worktree="auth-refactor"
   → events.jsonl: worktree.create.before + worktree.create.after

3. 在 worktree 中工作
   worktree_run("auth-refactor", "python -m pytest")
   → subprocess 在 .worktrees/auth-refactor/ 里跑
   → 完全隔离，改动不影响主目录

4. 收尾（二选一）
   worktree_keep("auth-refactor")
     → 目录保留，状态标 kept
   worktree_remove("auth-refactor", complete_task=True)
     → git worktree remove
     → task_1.json: status=completed, worktree=""
     → events.jsonl: task.completed + worktree.remove.after
```

一次 `worktree_remove(complete_task=True)` 搞定拆除 + 完成——删除目录、完成任务、解绑 worktree、发出事件，一个调用搞定。

---

## 两个状态机

```
任务：     pending → in_progress → completed
Worktree： absent  → active      → removed | kept
```

绑定时任务从 pending 推进到 in_progress。remove 时可选推进到 completed。两个状态机通过 task_id 关联，但各自独立变化。

---

## 做完后怎么合并到主分支

s12 的代码**没有实现合并**——它只管隔离执行和收尾。合并是标准的 git 操作：

```bash
# 方式一：直接合并
git merge wt/auth-refactor

# 方式二：创建 PR（更常见）
git push origin wt/auth-refactor
# 在 GitHub 上创建 PR，review 后合并
```

Agent 可以在 worktree 里用 `worktree_run` 执行 commit、push，然后通过 PR 流程合并。

为什么不自动合并？因为合并可能有冲突——两个 Agent 都改了同一个文件的不同部分，需要人工解决。自动合并是危险操作，和 s10 的设计哲学一致：高风险操作应该有审批。

---

## 16 个工具——12 课的巅峰

s12 是工具最多的一课：

| 类别 | 工具 | 数量 |
|------|------|------|
| 基础 | bash, read_file, write_file, edit_file | 4 |
| 任务 | task_create, task_list, task_get, task_update, task_bind_worktree | 5 |
| Worktree | worktree_create, worktree_list, worktree_status, worktree_run, worktree_keep, worktree_remove, worktree_events | 7 |

全部走标准 dispatch map。agent_loop 回归最简洁的形式——没有任何工具需要特殊处理。从 s01 的 1 个工具到 s12 的 16 个工具，循环本身始终不变。

---

## 和 s11 的变更对比

| 组件 | s11 | s12 |
|------|-----|-----|
| 执行范围 | 共享一个目录 | 每个任务独立 worktree |
| 文件冲突 | 可能 | 不可能（目录隔离） |
| 协调 | 任务板（owner/status） | 任务板 + worktree 绑定 |
| 收尾 | 只标任务完成 | 任务完成 + worktree keep/remove |
| 可观测性 | 隐式 | events.jsonl 显式事件流 |
| 可恢复性 | 只有任务状态 | 任务 + worktree 索引 + 事件流 |

---

## 12 课全景回顾

从 s01 到 s12，我们从零搭建了一个完整的 Agent Harness：

```
第一阶段：基础能力
  s01 循环      → 骨架
  s02 工具      → 双手

第二阶段：智能管理
  s03 规划      → 备忘录
  s04 子 Agent  → 分身
  s05 技能加载  → 工作手册
  s06 上下文压缩 → 记忆管理

第三阶段：持久化
  s07 任务系统  → 磁盘上的目标
  s08 后台执行  → 并行等待

第四阶段：团队协作
  s09 Agent 团队 → 身份 + 邮箱
  s10 团队协议   → 握手 + 审批
  s11 自治 Agent → 自组织
  s12 Worktree   → 目录隔离
```

**循环从未改变。** 每课都是在同一个 `while True → create → stop_reason → execute → append` 循环之上叠加 Harness 机制。从 1 个工具到 16 个工具，从单线程到多 goroutine/线程，从内存状态到磁盘持久化——变的永远是 Harness，不是循环。

这就是这个项目的核心论点：**模型是 Agent 的大脑，Harness 是 Agent 的身体和环境。** 造好 Harness，Agent 会完成剩下的。

**Go 语言实现**：[s12/main.go](https://github.com/Zhudanya/learn-claude-code/tree/main/agents_go/s12/main.go)

---

*本文是 learn-claude-code 系列学习笔记的最后一篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s12 课程。感谢 [shareAI-lab](https://github.com/shareAI-lab) 提供的优秀教学仓库。*
