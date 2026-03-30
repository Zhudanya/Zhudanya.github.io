---
title: "learn-claude-code 学习笔记（五）：Skill Loading —— 用到什么知识，临时加载什么知识"
date: 2026-03-23 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, skill-loading]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第五篇。s04 用子 Agent 解决了上下文噪声污染的问题，这篇看 s05 怎么解决另一个问题：怎么给 Agent 领域知识。

## 问题：把所有知识塞进 system prompt 太浪费

你希望 Agent 掌握特定领域的工作流——git 约定、测试规范、代码审查清单、PDF 处理指南。但如果全塞进 system prompt：10 个技能，每个 2000 token，就是 20,000 token。每次 API 调用都要带着，但当前任务可能只用到其中一个。

这就像一个工程师随身背着 10 本技术手册上班。大部分时候只用得到一本，但每天都要背着全部。

## 解决方案：两层注入

```
第一层（system prompt）—— 只放目录（便宜）
  "Skills available:
    - pdf: Process PDF files
    - code-review: Review code
    - agent-builder: Build AI agents"
  每个技能 ~20 token，全部加起来 ~100 token

第二层（tool_result）—— 按需加载完整内容
  模型调 load_skill("pdf") 时，才把 ~2000 token 的完整手册注入上下文
```

模型知道有什么技能（便宜），需要时再加载完整内容（贵但精准）。

---

## 技能文件长什么样

每个技能是一个目录，包含一个 `SKILL.md` 文件：

```
skills/
  pdf/
    SKILL.md
  code-review/
    SKILL.md
  agent-builder/
    SKILL.md
  mcp-builder/
    SKILL.md
```

SKILL.md 的格式：

```markdown
---
name: pdf
description: Process PDF files - extract text, create PDFs, merge documents.
---

# PDF Processing Skill

You now have expertise in PDF manipulation. Follow these workflows:

## Reading PDFs
...（完整的操作指南、代码示例）

## Creating PDFs
...

## Merging PDFs
...
```

两个 `---` 之间是 frontmatter（元数据：名字和描述），之后是正文（完整的领域知识）。frontmatter 用于第一层目录，正文用于第二层按需加载。

---

## SkillLoader——技能管理器

### 整体结构

```python
class SkillLoader:
    def __init__(self, skills_dir):     # 启动时扫描所有 SKILL.md
    def _load_all(self):                # 读文件，解析 frontmatter
    def _parse_frontmatter(self, text): # 拆分元数据和正文
    def get_descriptions(self):         # 第一层：生成目录
    def get_content(self, name):        # 第二层：返回完整内容
```

### 启动时加载

```python
SKILLS_DIR = WORKDIR / "skills"
SKILL_LOADER = SkillLoader(SKILLS_DIR)
```

程序启动时，SkillLoader 递归扫描 skills 目录下所有 `SKILL.md`，把每个文件拆成元数据（name、description）和正文，存进 `self.skills` 字典：

```python
def _load_all(self):
    for f in sorted(self.skills_dir.rglob("SKILL.md")):
        text = f.read_text()
        meta, body = self._parse_frontmatter(text)
        name = meta.get("name", f.parent.name)
        self.skills[name] = {"meta": meta, "body": body, "path": str(f)}
```

加载完后 `self.skills` 长这样：

```python
{
    "pdf": {
        "meta": {"name": "pdf", "description": "Process PDF files..."},
        "body": "# PDF Processing Skill\n\n...（完整 113 行内容）"
    },
    "code-review": {
        "meta": {"name": "code-review", "description": "Review code..."},
        "body": "...（完整内容）"
    },
    # ...
}
```

所有文件只读一次，之后从内存里取。

### frontmatter 解析

```python
def _parse_frontmatter(self, text: str) -> tuple:
    match = re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)
    if not match:
        return {}, text
    meta = {}
    for line in match.group(1).strip().splitlines():
        if ":" in line:
            key, val = line.split(":", 1)
            meta[key.strip()] = val.strip()
    return meta, match.group(2).strip()
```

用正则把文件按 `---` 分隔线拆成两部分。`split(":", 1)` 只按第一个冒号拆——因为 description 的值里可能包含冒号。

### get_descriptions——第一层

```python
def get_descriptions(self) -> str:
    lines = []
    for name, skill in self.skills.items():
        desc = skill["meta"].get("description", "No description")
        line = f"  - {name}: {desc}"
        lines.append(line)
    return "\n".join(lines)
```

**什么时候调用**：程序启动时一次，结果写入 system prompt。

输出类似：

```
  - agent-builder: Build AI agents with tool use and agent loops
  - code-review: Review code for quality, security, and best practices
  - mcp-builder: Build MCP servers
  - pdf: Process PDF files - extract text, create PDFs, merge documents
```

每个技能一行，全部加起来 ~100 token。模型看到后知道"有这些技能可以加载"。

### get_content——第二层

```python
def get_content(self, name: str) -> str:
    skill = self.skills.get(name)
    if not skill:
        return f"Error: Unknown skill '{name}'. Available: {', '.join(self.skills.keys())}"
    return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

**什么时候调用**：模型调 `load_skill("pdf")` 时。

返回完整内容，用 `<skill>` 标签包裹。这段内容作为 tool_result 进入 messages——模型立刻就能按照里面的指南行动。

找不到时返回错误并列出所有可用技能名，模型看到后能修正重试。

### 两个方法的对比

| | get_descriptions | get_content |
|---|---|---|
| 返回什么 | 所有技能的一行摘要 | 一个技能的完整内容 |
| 多少 token | ~100（全部技能） | ~2000（单个技能） |
| 放在哪 | system prompt | tool_result |
| 调用时机 | 程序启动时一次 | 模型调 load_skill 时 |
| 作用 | 让模型知道有什么 | 让模型知道怎么做 |

---

## system prompt 怎么变的

```python
# s04
SYSTEM = "You are a coding agent... Use the task tool to delegate exploration or subtasks."

# s05
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
{SKILL_LOADER.get_descriptions()}"""
```

`SKILL_LOADER.get_descriptions()` 在程序启动时求值，结果嵌入 SYSTEM 字符串。之后 SYSTEM 就是固定的，每次 API 调用都带着这个技能目录。

---

## load_skill 就是一个普通工具

和 s04 的 task 工具不同，load_skill **完全走标准 dispatch map**，不需要循环做任何特殊处理：

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),  # ← 和其他工具一样
}
```

agent_loop 比 s04 反而更简洁了——没有任何工具需要特殊处理。load_skill 的执行函数返回一段文字，和 read_file 返回文件内容没有本质区别。

---

## Claude Code 也是这么做的

学到这里我验证了一下——Claude Code 的真实架构确实用了同样的两层模式。

**第一层**：Claude Code 的 system prompt 里列出了所有可用 skill 的名字和描述：

```
The following skills are available for use with the Skill tool:
- simplify: Review changed code for reuse, quality...
- superpowers:brainstorming: Use before any creative work...
- superpowers:test-driven-development: Use when implementing...
- claude-api: Build apps with the Claude API...
```

**第二层**：当 Claude Code 调用 Skill 工具加载某个技能时，完整内容才通过 tool_result 注入上下文。

原理和 s05 教的完全一样，只是 Claude Code 的实现更完整（更多技能、更复杂的加载逻辑）。

---

## 小结

s05 解决的是 Harness 公式里 **Knowledge** 的部分——怎么给 Agent 领域知识。

核心设计：**不要把所有知识前置塞入，让模型按需加载。** 第一层放目录（便宜，每次都带），第二层放内容（贵，按需加载）。

这个模式的巧妙之处：模型自己判断需要什么知识。你不需要预测"这次任务需要哪个技能"然后帮它提前加载——模型看到技能目录后，会根据当前任务自己决定要不要加载、加载哪个。

> 用到什么知识，临时加载什么知识。

下一篇我们看 s06：上下文总会满，要有办法腾地方——上下文压缩是怎么回事。

---

*本文是 learn-claude-code 系列学习笔记的第五篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s05 课程。*
