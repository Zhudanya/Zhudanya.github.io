---
title: "learn-claude-code 学习笔记（二）：Tool Use —— 给 Agent 换一套专业工具箱"
date: 2026-03-20 20:00:00 +0800
categories: [AI, learn-claude-code]
tags: [ai-agent, harness-engineering, claude-code, tool-use]
---

这是 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 系列学习笔记的第二篇。上一篇我们搭了一个最小 Agent——一个 while 循环加一个 bash 工具。这篇来看 s02 做了什么：在不改循环的前提下，给 Agent 加更多工具。

## s01 的问题：一把瑞士军刀的困境

s01 只有一个 bash 工具，所有操作都走 shell。能用，但处处是坑。

**读文件——cat 截断不可控：**

```bash
cat very_large_file.py
```

cat 一口气输出整个文件。几万行的文件直接撑爆上下文窗口。s01 的 `run_bash` 虽然有 `[:50000]` 截断，但按字符数截——可能从某行中间砍断，模型看到的代码是残缺的。

**改文件——sed 怕特殊字符：**

```bash
sed -i 's/price = $100/price = $200/g' config.py
```

`$` 在 sed 里是正则特殊字符（表示行尾），`$1` 会被当成反向引用。`/`、`\`、`&`、`*` 全都会被误解。模型经常忘记转义，改错文件或命令报错。

**写文件——shell 引号是噩梦：**

```bash
echo "def hello():
    print('hello')
    return True" > hello.py
```

内容里有单引号、双引号、反引号？shell 会解释它们而不是当作普通文本写入。多行文本的换行和缩进也可能被 shell 吞掉。

**安全——bash 能做任何事：**

```bash
cat /etc/passwd                     # 读系统文件
cat ../../other_project/secret.py   # 逃逸工作目录
rm -rf ./*                          # s01 只拦截了 rm -rf /，这个拦不住
```

一句话总结：**bash 是瑞士军刀，什么都能切，但切不好。**

---

## s02 的解决方案：一套专业工具箱

加三个专用工具：`read_file`、`write_file`、`edit_file`，每个只干一件事，但干得精准、可控、安全。

核心洞察：**加工具不需要改循环。**

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+
```

---

## 路径沙箱——所有文件工具的安全地基

```python
WORKDIR = Path.cwd()

def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

三个文件工具都经过这个函数。它做了什么：把相对路径拼上工作目录，`resolve()` 解析掉 `../` 之类的路径穿越，然后检查结果是否还在工作目录内。模型试图访问 `../../etc/passwd`？resolve 后跑到工作目录外，直接拦截。

bash 做不到这个——`cat ../../etc/passwd` 对 shell 来说是合法命令。

---

## 三个新工具

### read_file——可控的文件读取

```python
def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
    return "\n".join(lines)[:50000]
```

和 `cat` 的区别：按**行**截断而不是按字符截断，不会砍断某一行。支持 `limit` 参数只读前 N 行，还会告诉模型"还有多少行没显示"。

### write_file——不经过 shell 的写入

```python
def run_write(path: str, content: str) -> str:
    fp = safe_path(path)
    fp.parent.mkdir(parents=True, exist_ok=True)
    fp.write_text(content)
    return f"Wrote {len(content)} bytes to {path}"
```

Python 直接写字符串到文件，不经过 shell 解释。模型传什么内容就写什么内容——引号、反斜杠、特殊字符全是普通文本。还自动创建父目录。

### edit_file——精确的文本替换

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    fp = safe_path(path)
    content = fp.read_text()
    if old_text not in content:
        return f"Error: Text not found in {path}"
    fp.write_text(content.replace(old_text, new_text, 1))
    return f"Edited {path}"
```

纯字符串匹配和替换，没有正则，`$`、`/`、`\` 都是普通字符。`replace(..., 1)` 只替换第一次出现，防止误改。找不到时返回错误而不是静默失败。

### 对比总结

| 问题 | bash 的缺陷 | 专用工具的解决方式 |
|------|------------|------------------|
| 读大文件 | cat 全量输出，截断不可控 | read_file 按行截断，支持 limit |
| 改文件 | sed 有正则/特殊字符问题 | edit_file 纯字符串替换 |
| 写文件 | echo 受 shell 引号和转义影响 | write_file 直接写入 |
| 安全 | bash 能做任何事 | safe_path 从路径层面限制范围 |

---

## dispatch map——s02 的核心设计模式

工具多了，怎么路由？s01 只有一个 bash 工具，直接硬编码调 `run_bash` 就行。但现在有四个工具，总不能写一堆 if/elif：

```python
# 丑：每加一个工具就要改循环
if block.name == "bash":
    output = run_bash(block.input["command"])
elif block.name == "read_file":
    output = run_read(block.input["path"], block.input.get("limit"))
elif block.name == "edit_file":
    output = run_edit(...)
```

s02 的做法——用字典做映射：

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

循环里永远只有一行查找：

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)
```

**加一个新工具只需要两步：写一个 handler 函数，在字典里加一行。循环永远不用动。**

### lambda 和 **kw 在干什么

这里涉及两个 Python 语法，可能不太直观。

**lambda** 就是匿名函数，以下两种写法完全等价：

```python
# lambda 写法
"bash": lambda **kw: run_bash(kw["command"])

# 等价的普通函数写法
def bash_handler(**kw):
    return run_bash(kw["command"])
"bash": bash_handler
```

用 lambda 纯粹是因为简洁——四个工具四行搞定。

**`**kw`** 会把所有关键字参数打包成字典。看实际执行过程：

```python
# 模型调用 edit_file，block.input 是：
block.input = {"path": "hello.py", "old_text": "foo", "new_text": "bar"}

# 循环里执行：
handler(**block.input)
# ** 把字典展开：等价于 handler(path="hello.py", old_text="foo", new_text="bar")

# lambda 的 **kw 又收回成字典：
kw = {"path": "hello.py", "old_text": "foo", "new_text": "bar"}

# 然后取值传给 run_edit：
run_edit("hello.py", "foo", "bar")
```

为什么要这么绕？因为四个工具的参数不一样——bash 有 1 个参数，edit_file 有 3 个。`**kw` 让所有 handler 都能用同一种方式 `handler(**block.input)` 调用，各自按需取值。

还有个细节：`kw["command"]` 和 `kw.get("limit")` 的区别：

```python
kw["command"]     # key 不存在就报错 → 用于必填参数
kw.get("limit")   # key 不存在返回 None → 用于可选参数
```

`limit` 不在 read_file 的 required 里，模型可能不传。用 `.get()` 安全取值避免 KeyError。

---

## TOOLS 列表——给模型看的"说明书"

```python
TOOLS = [
    {"name": "bash",
     "description": "Run a shell command.",
     "input_schema": {
         "type": "object",
         "properties": {"command": {"type": "string"}},
         "required": ["command"]}},

    {"name": "read_file",
     "description": "Read file contents.",
     "input_schema": {
         "type": "object",
         "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}},
         "required": ["path"]}},
    ...
]
```

每个工具有三个字段：

| 字段 | 作用 | 给谁看的 |
|------|------|---------|
| name | 工具的唯一标识 | 模型用这个名字调用工具 |
| description | 功能描述 | 模型靠这个判断什么时候用 |
| input_schema | 参数格式（JSON Schema） | 模型按这个格式生成参数 |

`input_schema` 是标准的 JSON Schema，不是 Anthropic 自创的格式。`properties` 定义有哪些参数及其类型，`required` 定义哪些必填。

关键：**TOOLS（说明书）和 TOOL_HANDLERS（执行代码）是分开定义的。** 模型只看 TOOLS 里的 name、description、schema 来决策；TOOL_HANDLERS 是代码内部路由，模型完全不知道。两边靠 `name` 连接，参数名必须一致。

```
TOOLS（模型看的）                  TOOL_HANDLERS（代码跑的）
name: "edit_file"       ────────→  "edit_file": run_edit()
  path: string          ────────→    def run_edit(path,
  old_text: string      ────────→                old_text,
  new_text: string      ────────→                new_text):
```

---

## 循环对比——几乎没变

把 s01 和 s02 的 agent_loop 放一起看，唯一的区别就是工具执行那几行：

```python
# s01：硬编码
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])

# s02：查字典
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
```

while True、client.messages.create、stop_reason 判断、messages 追加——全都一模一样。

---

## 小结

s02 的本质：**给 Agent 的身体加了更多器官。**

s01 的 Agent 只有一只手（bash），什么都得用这只手干。s02 给了它专门读文件的眼睛（read_file）、专门写文件的手（write_file）、专门改文件的手术刀（edit_file），每个器官都自带安全边界（safe_path）。

大脑（模型）没变，循环没变。变的是 Harness 给模型提供的工具集——从一把瑞士军刀换成了一套专业工具箱。

而这个替换过程，靠的就是 dispatch map 这个设计模式：一个字典，把工具名映射到执行函数。以后不管加多少工具，循环都不用改。

> 加工具 = 加 handler + 加 schema。循环永远不变。

下一篇我们看 s03：没有计划的 Agent 走哪算哪——怎么给 Agent 加一个规划系统。

---

*本文是 learn-claude-code 系列学习笔记的第二篇，基于 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的 s02 课程。*
