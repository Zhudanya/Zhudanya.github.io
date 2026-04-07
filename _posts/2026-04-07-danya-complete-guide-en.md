---
title: "Danya Complete Guide — Game Dev AI Coding Assistant"
date: 2026-04-07 10:00:00 +0800
categories: [AI Agent, Game Development]
tags: [danya, ai-agent, game-dev, harness, cli]
pin: true
toc: true
lang: en
---

## What is Danya

Danya is an AI coding assistant that runs in the terminal, **designed specifically for game development**. It is not a generic code completion tool — it is an Agent that understands game project architecture, enforces quality standards, and can automate entire development workflows.

> [中文版](../danya-complete-guide/)

**Out of the box** — enter a game project and launch Danya, and it will automatically detect the engine type (Unity / Unreal / Godot / Go Server / C++ Server / Java Server / Node.js Server), generating a complete Harness governance system (rules, commands, Hooks, memory) with no manual setup required.

**Core positioning**: Embed battle-tested Game Harness Engineering into an AI Agent, ready to use out of the box for game developers.

> GitHub: [https://github.com/Zhudanya/danya](https://github.com/Zhudanya/danya)
> npm: `npm install -g @danya-ai/cli`

---

## Installation & Updates

### Installation

```bash
# Option 1: npm install (recommended)
npm install -g @danya-ai/cli

# Option 2: Install from source
git clone https://github.com/Zhudanya/danya.git
cd danya
bun install && bun run build
npm install -g .
```

```bash
# Verify installation
danya --version
```

### Update

```bash
npm install -g @danya-ai/cli@latest
```

---

## Getting Started

### 1. Launch Danya

```bash
cd <your-game-project>
danya
```

On first launch, you will be guided to configure an AI model. It will also automatically detect the project engine type and generate the `.danya/` directory.

### 2. Configure AI Model

Type `/model` in the conversation:

1. Select **Manage Model List** → Add a new model
2. Choose Provider:
   - **Custom Messages API** — Claude series (Anthropic official API)
   - **Custom OpenAI-Compatible API** — GPT / DeepSeek / Qwen / GLM, etc.
   - **Ollama** — Local models
3. Paste API Key → Press Enter through the remaining steps (use defaults)
4. Go back to the `/model` page and select the model to use

#### Common Model Configurations

| Model | Provider | Base URL | Model ID |
|-------|----------|----------|----------|
| Claude Opus 4.6 | Messages API | `https://api.anthropic.com` | `claude-opus-4-6` |
| Claude Sonnet 4.6 | Messages API | `https://api.anthropic.com` | `claude-sonnet-4-6` |
| DeepSeek V3 | Messages API | `https://api.deepseek.com/anthropic` | `deepseek-chat` |
| GPT-4o | OpenAI-Compatible | `https://api.openai.com/v1` | `gpt-4o` |
| Qwen Max | OpenAI-Compatible | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-max` |

Model configuration is saved in `~/.danya/config.json` — no manual editing needed.

#### Multi-Model Coordination (Multi-Agent Model Configuration)

Danya uses 4 semantic pointers to control which model is used for different tasks:

| Pointer | Purpose | When Used |
|---------|---------|-----------|
| `main` | Main conversation, coding, review | Direct user conversation, coding stage of /auto-work |
| `task` | Sub-Agents (subagent, background task) | Agent spawns subagent to search code, analyze files |
| `compact` | Context compression summaries | Auto-compresses when conversation nears context window |
| `quick` | Quick classification decisions | Stage 0 classification in /auto-work |

**Configuration Method 1: In-conversation configuration**

Type `/model` in Danya, add multiple models, then assign pointer roles to each model in the model list.

**Configuration Method 2: Edit configuration file directly**

Edit `~/.danya/config.json`:

```json
{
  "modelPointers": {
    "main": "DeepSeek V3",
    "task": "Qwen Max",
    "compact": "GLM-4",
    "quick": "GLM-4"
  }
}
```

**Cost-saving strategies**:

| Strategy | main | task | compact | quick |
|----------|------|------|---------|-------|
| Full power | Claude Opus | Claude Sonnet | Claude Haiku | Claude Haiku |
| Budget-friendly | DeepSeek V3 | Qwen Max | GLM-4 | GLM-4 |
| Single model | Same | Same | Same | Same |

If only one model is configured, all pointers automatically point to it — no extra configuration needed.

### 3. Initialize Harness

Danya automatically initializes the `.danya/` directory when entering a project for the first time. You can also manually run:

```
/init
```

This will:
- Detect the project engine type
- Deploy the complete Harness system (rules, commands, Hooks, memory templates)
- If `.claude/` or `.codex/` directories exist, automatically integrate them into `.danya/`
- Generate `CLAUDE.md` (for Claude models) or `AGENTS.md` (for other models) based on the AI model type

### 4. Check Environment Dependencies

```bash
danya check-env
```

Output example:

```
=== Danya Environment Check ===
  [OK] danya
  [OK] git
  [OK] make
  [OK] python3
  [OK] go
  [SKIP] dotnet (only needed for C# syntax checking)

All required dependencies found.
```

---

## CLI Startup Parameters

### Basic Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `[prompt]` | Pass a prompt directly | `danya "analyze this project"` |
| `--cwd <path>` | Specify working directory | `danya --cwd /path/to/project` |
| `-p, --print` | Non-interactive mode (output result and exit, suitable for scripts/pipes) | `danya -p "explain this function"` |
| `--model <name>` | Specify the model to use | `danya --model opus` |
| `--system-prompt <prompt>` | Custom system prompt | `danya --system-prompt "You are a Go expert"` |
| `--append-system-prompt <prompt>` | Append content after the default system prompt | `danya --append-system-prompt "Reply only in English"` |
| `--verbose` | Verbose output mode | `danya --verbose` |

### Permissions & Security

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--dangerously-skip-permissions` | Skip all permission confirmations (fully automatic Agent, no confirmation dialogs) | `danya -p "refactor code" --dangerously-skip-permissions` |
| `--permission-mode <mode>` | Set permission mode | `danya --permission-mode dontAsk` |
| `--max-budget-usd <amount>` | Maximum API spend limit (USD), `--print` mode only | `danya -p "write tests" --max-budget-usd 5` |

**Permission mode options**:

| Mode | Description |
|------|-------------|
| `default` | Default mode, dangerous operations require confirmation |
| `acceptEdits` | Auto-accept file edits, other operations still require confirmation |
| `dontAsk` | No confirmation dialogs (equivalent to `--dangerously-skip-permissions`) |
| `plan` | Planning mode, analyze only without executing |
| `bypassPermissions` | Bypass permission checks |

### Tool Control

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--allowedTools <tools>` | List of allowed tools | `danya -p "search code" --allowedTools "Read,Grep,Glob"` |
| `--disallowedTools <tools>` | List of disallowed tools | `danya --disallowedTools "Bash,Write"` |
| `--tools <tools>` | Specify available tool set (`--print` mode only) | `danya -p "read file" --tools "Read,Grep"` |

### Output Format

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--output-format <format>` | Output format: text / json / stream-json | `danya -p "analyze" --output-format json` |
| `--input-format <format>` | Input format: text / stream-json | `danya -p --input-format stream-json` |
| `--json-schema <schema>` | JSON Schema for structured output | `danya -p "extract info" --json-schema '{"type":"object"}'` |
| `--include-partial-messages` | Include intermediate messages in streaming output | `danya -p --output-format stream-json --include-partial-messages` |

### MCP Configuration

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--mcp-config <config>` | Load MCP Server configuration | `danya --mcp-config mcp-servers.json` |

### Debugging

| Parameter | Description | Example |
|-----------|-------------|---------|
| `-d, --debug [filter]` | Debug mode (can filter by category) | `danya --debug "api,hooks"` |
| `--debug-verbose` | Verbose debug output | `danya --debug-verbose` |

### Common Usage Examples

```bash
# Fully automatic unattended development (skip permissions, no confirmations)
danya -p "implement weapon upgrade system" --dangerously-skip-permissions

# Allow only read and search (safe mode, no file modifications)
danya -p "analyze project architecture" --allowedTools "Read,Grep,Glob,Bash"

# Structured JSON output (suitable for script processing)
danya -p "list all TODOs" --output-format json

# Limit spend to $5
danya -p "write complete tests" --dangerously-skip-permissions --max-budget-usd 5

# Planning mode (analyze only, no execution)
danya --permission-mode plan

# Custom system prompt
danya --append-system-prompt "Answer everything in English, comments in English too"
```

---

## .danya/ Directory Structure

```
.danya/
├── rules/                 — Constraint rules (auto-loaded every session)
│   ├── constitution.md    — Forbidden zone rules (auto-generated code that must not be edited)
│   ├── golden-principles.md — Coding golden principles (UniTask, error wrapping, etc.)
│   ├── known-pitfalls.md  — Known pitfalls (accumulated through self-evolution)
│   ├── architecture-boundaries.md — Architecture dependency direction
│   └── <engine>-style.md  — Engine-specific style guide
│
├── commands/              — Workflow commands (/auto-work, /review, etc.)
│   ├── auto-work.md
│   ├── auto-bugfix.md
│   ├── review.md
│   ├── fix-harness.md
│   ├── plan.md
│   ├── verify.md
│   └── parallel-execute.md
│
├── memory/                — Persistent domain knowledge (not lost during context compression)
│   ├── MEMORY.md          — Index
│   └── <module>.md        — Project knowledge learned by the Agent
│
├── hooks/                 — Mechanically enforced scripts (Agent cannot bypass)
│   ├── constitution-guard.sh  — Gate 0: Forbidden zone guard
│   ├── pre-commit.sh          — Gate 3: Pre-commit lint+test
│   ├── post-commit.sh         — Gate 4: Post-commit review reminder
│   ├── push-gate.sh           — Gate 5: Pre-push check for push-approved
│   ├── harness-evolution.sh   — Self-evolution detection
│   └── syntax-check.sh        — Syntax check
│
├── agents/                — Role Agent specifications
│   ├── code-writer.md     — Coding Agent
│   ├── code-reviewer.md   — Review Agent
│   ├── red-team.md        — Red Team Agent (find bugs)
│   ├── blue-team.md       — Blue Team Agent (fix bugs)
│   └── skill-extractor.md — Experience extraction Agent
│
├── scripts/               — Shell orchestration scripts
│   ├── auto-work-loop.sh  — Fully automatic pipeline (Shell-enforced version)
│   ├── parallel-wave.sh   — Multi-Agent wave-based parallelism
│   ├── red-blue-loop.sh   — Red-blue adversarial loop
│   ├── orchestrator.sh    — Auto-iteration score grinding
│   ├── verify-server.sh   — Server-side quantitative verification (0-100 score)
│   ├── verify-client.sh   — Client-side quantitative verification
│   ├── check-env.sh       — Environment check
│   └── monthly-report.sh  — Monthly report
│
├── monitor/               — Monitoring data collection
│   ├── log-tool-use.py    — Tool usage logging
│   ├── log-session-end.py — Session end logging
│   ├── log-verify.py      — Verification timing logging
│   ├── log-bugfix.py      — Bug fix logging
│   ├── log-review.py      — Review score logging
│   ├── analyze.py         — Data analysis (8 metrics)
│   └── dashboard.py       — Real-time monitoring dashboard
│
├── templates/             — Task definition templates
│   └── program-template.md
│
├── settings.json          — Hook registration
├── gate-chain.json        — Gate Chain configuration
└── guard-rules.json       — Forbidden zone rules
```

**Users can freely add, remove, or modify** any file under `.danya/`. Danya will not overwrite user customizations.

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+G` | Open external editor; content auto-fills when closed |
| `Shift+Enter` | New line in input box without sending |
| `Enter` | Submit |
| `Ctrl+M` | Quick model switch |
| `Shift+Tab` | Toggle input mode (Normal / Bash / Memory) |

---

## In-Conversation Command Reference

### /auto-work — Fully Automatic Pipeline

```
/auto-work "add inventory sorting feature"
```

The Agent automatically completes 7 stages:

1. **Stage 0: Classification** — Determine if it's a bug / feature / refactor
2. **Stage 1: Planning** — Analyze requirements, list affected files
3. **Stage 2: Coding** — Write code, compile-check after each file (fail-fast)
4. **Stage 3: Review** — 100-point scoring, fails if <80 or any CRITICAL
5. **Stage 4: Commit** — git commit (pre-commit hook runs lint+test)
6. **Stage 5: Documentation** — Automatically write docs to Docs/
7. **Stage 6: Self-evolution** — Check for errors during development, update rules

**Termination conditions**:
- Verification fails 3 rounds → abort
- Review score <80 for 3 rounds → abort
- Commit fails 2 times → abort

### /auto-bugfix — Automatic Bug Fix

```
/auto-bugfix "animation glitch during character state transition"
```

**Must reproduce before fixing**:

1. **Reproduce** — Analyze bug, find reproduction steps, verify bug exists
2. **Root cause analysis** — No guessing, read code, check logs
3. **Fix** — Up to 5 rounds: modify code → verify → retry if not passing
4. **Review** — 100-point scoring
5. **Commit** — git commit
6. **Documentation** — Write to Docs/Bugs/

### /review — Score-Based Code Review

```
/review
```

**Scoring rules**:

| Severity | Deduction | Example |
|----------|-----------|---------|
| CRITICAL | -30 | Compilation failure, editing forbidden files, data corruption risk |
| HIGH | -10 | Unhandled error, race condition, missing validation |
| MEDIUM | -3 | Naming violation, missing logging, dead code |

- Pass threshold: **>= 80 with no CRITICAL**
- Quality ratchet: scores can only go up, never down
- On passing, generates a `push-approved` token (single-use)

**Review covers 4 dimensions**:
1. Architecture compliance (forbidden zones, cross-layer references, dependency direction)
2. Coding standards (engine rules, error handling, naming)
3. Logic review (boundaries, concurrency, error propagation)
4. Harness integrity (whether errors have been codified into rules)

### /fix-harness — Self-Evolution

```
/fix-harness "forgot to check nil before use, caused panic at compile time"
```

The Agent will:
1. Analyze the error pattern
2. Route to the correct rule file:
   - Forbidden zone violation → `constitution.md`
   - Coding principle violation → `golden-principles.md`
   - Known pitfall → `known-pitfalls.md`
   - Architecture boundary violation → `architecture-boundaries.md`
3. Add rule: incorrect pattern + correct pattern
4. Keep each rule file < 550 lines

**Auto-triggered**: PostToolUse Hook detects "error-then-fix" patterns and automatically suggests running `/fix-harness`.

### /plan — Requirements Analysis & Planning

```
/plan "implement weapon upgrade system"
```

Output:
1. Requirements analysis (what to change, why)
2. File manifest (each file + one-line change intent + risk level)
3. Execution order (dependencies, what can be parallelized)
4. Verification strategy

### /verify — Mechanical Verification

```
/verify          # default: quick
/verify build    # compile
/verify full     # compile + test
```

| Level | Content |
|-------|---------|
| quick | lint + syntax check |
| build | quick + full compilation |
| full | build + run tests |

### /parallel-execute — Wave-Based Parallel Execution

```
/parallel-execute prepare "add slot machine multiplier feature"
```

The Agent will:
1. Analyze requirements, break down into atomic tasks (task-01.md, task-02.md...)
2. Declare dependencies
3. Compute waves (topological sort)
4. Display wave plan, wait for user confirmation
5. Launch `parallel-wave.sh`: each task executes in an isolated worktree

**Task file format**:

```markdown
---
depends: []
---

## Task Description
Modify handler.go, add multiplier parameter support.

## Verification Method
go build ./servers/logic_server/...
```

Dependency declarations:
- `depends: []` — No dependencies, first wave
- `depends: [01]` — Depends on task-01
- `depends: [01, 02]` — Depends on 01 and 02

### /orchestrate — Auto-Iteration Score Grinding

```
/orchestrate my-task.md
```

Requires a **task definition file** (based on the `.danya/templates/program-template.md` template):

```markdown
## Goal
Increase test coverage from 15% to 60%

## Modifiable Scope
- servers/logic_server/internal/slot/

## Forbidden Files
- orm/
- common/config/cfg_*.go

## Quantitative Metrics
- make build: 40 points
- make lint: 20 points
- make test: 40 points

## Context
This module handles slot machine game logic
```

**Execution flow**: AI writes code → quantitative verification (0-100 score) → commit if score >= baseline, rollback if < baseline → loop N rounds. Auto-circuit-breaker after 5 consecutive failures.

### /red-blue — Red-Blue Adversarial

```
/red-blue servers/logic_server/
```

1. **Red Team** (read-only): Find all bugs (boundaries, error paths, concurrency, security)
2. **Blue Team** (writable): Fix bugs by priority (CRITICAL → HIGH → MEDIUM)
3. **Compile check**: Must compile after fixes
4. **Loop**: Until red team finds 0 bugs (max 5 rounds)
5. **Experience extraction**: skill-extractor analyzes logs, writes to rules/memory

### /monitor — View Harness Effectiveness

```
/monitor summary 7        # Last 7 days summary report
/monitor tools            # Tool usage distribution
/monitor reviews 30       # Last 30 days review scores
/monitor bugfixes 30      # Bug fix efficiency
/monitor sessions         # Session count
```

---

## Terminal Command Reference

### danya auto-work — Shell-Enforced Pipeline

```bash
danya auto-work "implement weapon upgrade system"
danya auto-work "fix login bug" --model opus
```

Difference from in-conversation `/auto-work`: **Shell enforces each stage with an independent `danya -p` execution; the Agent cannot skip steps, and context is isolated**. Suitable for large tasks and unattended scenarios.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `--model <m>` | Model to use | sonnet |
| `--max-turns <n>` | Max turns per stage | 30 |

### danya parallel — Multi-Agent Parallelism

```bash
danya parallel Docs/Version/v1.3/weapon/tasks
```

Reads task-NN.md files from the task directory, divides into waves by dependency, and launches an independent danya instance in an isolated worktree for each task. Auto-merges on successful compilation, auto-rolls back on failure.

### danya red-blue — Red-Blue Adversarial (Unattended)

```bash
danya red-blue .
danya red-blue servers/logic_server/ --rounds 3 --model opus
```

| Parameter | Description | Default |
|-----------|-------------|---------|
| `-n, --rounds <n>` | Max rounds | 5 |
| `--model <m>` | Model | sonnet |

### danya orchestrate — Auto-Iteration

```bash
danya orchestrate my-task.md
danya orchestrate my-task.md -n 30 --model opus
```

| Parameter | Description | Default |
|-----------|-------------|---------|
| `-n, --iterations <n>` | Max iterations | 20 |
| `--model <m>` | Model | sonnet |

### danya analyze — Data Analysis

```bash
danya analyze --metric summary --days 7       # Summary report
danya analyze --metric compare --days 7       # This week vs last week comparison
danya analyze --metric tool-usage --days 7    # Tool usage distribution
danya analyze --metric top-tools --top 10     # Top 10 tools
danya analyze --metric session-count --days 30 # Session count
danya analyze --metric verify-time --days 7   # Verification timing
danya analyze --metric bugfix-rounds --days 30 # Bug fix efficiency
danya analyze --metric review-scores --days 30 # Review score trends
```

8 metrics:

| Metric | What It Shows |
|--------|---------------|
| `summary` | Overview (verification, bug fixes, reviews, tools, sessions) |
| `compare` | Current period vs previous period comparison (with trend arrows) |
| `tool-usage` | How many times each tool was called |
| `top-tools` | Top N tool ranking |
| `session-count` | Daily session count distribution |
| `verify-time` | Verification timing (by type, avg/fastest/slowest/pass rate) |
| `bugfix-rounds` | Bug fix rounds, success rate |
| `review-scores` | Review average score, pass rate, CRITICAL/HIGH/MEDIUM count trends |

### danya dashboard — Real-Time Monitoring

```bash
danya dashboard          # View a single snapshot
danya dashboard -w       # Continuous monitoring, refresh every 5 seconds
danya dashboard -w 3     # Refresh every 3 seconds
danya dashboard -v       # Verbose mode (show PID, memory, recent tool calls)
danya dashboard -w -v    # Continuous + verbose
```

Displays: number of running Agent processes, active conversations, background tasks.

### danya report — Monthly Report

```bash
danya report
```

Summarizes all orchestrator iteration results: total sessions, iteration rounds, success rate, highest baseline score.

---

## Gate Chain

Every code change passes through 6 quality gates:

```
Edit → Guard → Syntax → Verify → Commit → Review → Push
       Hook     Hook     Tool     Hook    AI+Tool   Hook
       (hard    (instant (compile (lint+  (score-   (token-
        block)   check)   +test)   test)   based)    based)
```

- **Gate 0 Guard**: PreToolUse Hook, blocks edits to forbidden zone files
- **Gate 1 Syntax**: PostToolUse Hook, instant syntax check after edits
- **Gate 2 Verify**: Tool call, compile + test
- **Gate 3 Commit**: PreToolUse Hook, runs lint + test before commit
- **Gate 3.5 AssetGuard**: Pre-commit Hook, blocks binary files >5MB not tracked by Git LFS (threshold configurable via `ASSET_GUARD_THRESHOLD` environment variable; warns when `.gitattributes` is missing)
- **Gate 4 Review**: AI + ScoreReview tool, 100-point scoring
- **Gate 5 Push**: PreToolUse Hook, checks for push-approved token

**Push token system**: `/review` generates a `.danya/push-approved` file on passing; push checks and consumes the token (single-use).

---

## Engine Detection & Variants

Danya automatically detects the project engine type on startup:

| Engine | Detection Method | Injected Domain Knowledge |
|--------|-----------------|--------------------------|
| Unity | `ProjectSettings/` + `Assets/` | MonoBehaviour lifecycle, UniTask, object pooling, event pairing, architecture layers |
| Unreal | `*.uproject` | UPROPERTY, UE_LOG, FRunnable, naming conventions (F/U/A/E) |
| Godot | `project.godot` | Type hints, signal pairing, _physics_process, resource loading |
| Go Server | `go.mod` | Error wrapping, safego.Go, RPC handler standards, ECS constraints |
| C++ Server | `CMakeLists.txt` | Memory management, concurrency model, build system knowledge |
| Java Server | `pom.xml` / `build.gradle` + game keywords | GC tuning, Netty Pipeline, concurrency model |
| Node.js Server | `package.json` + game framework detection | Event loop, WebSocket, state synchronization |

### Workspace Three-Layer Isolation

When a project contains client + server subdirectories, Danya automatically recognizes it as workspace mode:

```
workspace/
├── .danya/            ← Layer 1: Cross-project rules (protocol sync, version management)
├── client/
│   └── .danya/        ← Layer 2: Client-specific (Unity rules, C# Hooks)
└── server/
    └── .danya/        ← Layer 2: Server-specific (Go rules, RPC constraints)
```

Layer 3 is session-level git worktree isolation (used during parallel-execute).

---

## Self-Evolution Mechanism

When the Agent makes an error and fixes it, the system automatically detects the "error-then-fix" pattern:

```
Agent compilation error → fix → compilation success
  → PostToolUse Hook detects error-then-fix
  → Prompt: "Error was fixed. Consider running /fix-harness"
  → Agent runs /fix-harness
  → Routes to known-pitfalls.md
  → Adds: ❌ incorrect pattern + ✅ correct pattern
  → Won't make the same mistake again
```

**Rule file constraint**: Each file stays < 550 lines. Merged and condensed when exceeded.

---

## Context Compression

Danya combines two compression strategies:

1. **Selective compression** (from Codex): Only compresses the oldest messages, preserving the 4 most recent messages verbatim
2. **8-section structured summary** (from Kode): Technical context, project overview, code changes, debugging issues, current state, pending tasks, user preferences, key decisions
3. **Automatic key file recovery**: After compression, automatically restores the 5 most recently accessed files
4. **Smart model switching**: Prefers compact model for compression, auto-switches to main when it doesn't fit

**Trigger condition**: Token usage >= 90% of context window.

**Manual trigger**: `/compact`

**Adjust threshold**: `/compact-threshold 0.85` (set to 85%)

---

## 5 Role Agents

These Agents are automatically invoked by the orchestrator and red-blue scripts:

| Role | Tool Permissions | Responsibility |
|------|-----------------|----------------|
| **code-writer** | Edit, Write, Read, Bash, Grep, Glob | Only modify code within specified scope, compile after each change, minimal modifications |
| **code-reviewer** | Read, Grep, Glob, Bash (read-only) | Read-only review, 100-point scoring |
| **red-team** | Read, Grep, Glob, Bash (read-only) | Assume code has bugs, find boundary/error path/concurrency/security issues |
| **blue-team** | Edit, Write, Read, Bash, Grep, Glob | Fix bugs by priority, defensive programming, minimal fixes |
| **skill-extractor** | Read, Write, Grep, Glob | Extract patterns from logs into rules/memory (requires 2+ occurrences) |

---

## Data Monitoring

### Automatic Collection (Transparent to Users)

Danya automatically collects data to `.danya/monitor/data/` via Hooks:

| Data | Hook Type | Collected Content |
|------|-----------|-------------------|
| `tool-usage.jsonl` | PostToolUse | Tool name, timestamp, session_id |
| `sessions.jsonl` | Stop | Session end time, reason |
| `verify-metrics.jsonl` | Script call | Verification type, result, duration |
| `bugfix-metrics.jsonl` | Script call | Bug description, rounds, result |
| `review-metrics.jsonl` | Script call | Score, issue count (CRITICAL/HIGH/MEDIUM) |

### Analysis Commands

```bash
# Summary report
danya analyze --metric summary --days 7

# This week vs last week comparison (with trend arrows)
danya analyze --metric compare --days 7

# Output example:
# ========================================
#   Harness Compare | last 7d vs prev 7d
# ========================================
#
#   Verify:
#     Avg duration: 12.3s (DOWN 15% OK)
#     Pass rate: 92% (UP 8% OK)
#
#   Bugfix:
#     Avg rounds: 1.5 (DOWN 25% OK)
#     Success: 90% (UP 10% OK)
#
#   Review:
#     Avg score: 91.2 (UP 5% OK)
#     CRITICAL: 0 (DOWN 100% OK)
# ========================================
```

---

## Migrating from Claude Code / Codex

If you previously used Claude Code or Codex and already have `.claude/` or `.codex/` configurations:

| Scenario | Danya Behavior |
|----------|---------------|
| Only `.claude/` | Automatically integrates into `.danya/`, custom content preserved |
| Only `.codex/` | Same as above |
| Both exist | Both integrated, `.claude/` takes priority over `.codex/` |
| Filename conflict | `.danya/` version preserved, legacy skipped |
| At runtime | Only reads `.danya/`, no longer reads `.claude/` or `.codex/` |

---

## Common Use Case Examples

### Scenario 1: Small Feature Development

```
danya
> /auto-work "add inventory sorting feature"
```

The Agent automatically completes all 7 stages with no manual intervention needed.

### Scenario 2: Large Feature Unattended

```bash
danya auto-work "implement weapon upgrade system" --model opus
```

Shell-enforced orchestration, each stage runs an independent danya -p, no step-skipping allowed.

### Scenario 3: Complex Feature Parallel Development

```
danya
> /parallel-execute prepare "add slot machine multiplier feature"
```

Agent breaks down tasks → displays wave plan → user confirms → multi-Agent parallelism.

### Scenario 4: Code Quality Improvement

```bash
danya red-blue servers/logic_server/
```

Red team finds bugs → blue team fixes → loop until zero bugs → extract experience.

### Scenario 5: Auto-Iterate to Boost Coverage

```bash
danya orchestrate test-coverage.md -n 30
```

AI writes tests → quantitative scoring → commit on pass → baseline goes from 15% all the way to 60%.

### Scenario 6: View Harness Effectiveness

```bash
danya analyze --metric compare --days 7
```

This week vs last week: how much verification time decreased, how much review scores improved, how many fewer bug fix rounds.

---

## Supported Engines

| Engine | Detection Method | Build Tool | Review Rule Count |
|--------|-----------------|------------|-------------------|
| Unity | `ProjectSettings/` + `Assets/` | UnityBuild, CSharpSyntaxCheck | 9 rules |
| Unreal Engine | `*.uproject` | UnrealBuild | 6 rules |
| Godot | `project.godot` | GodotBuild | 5 rules |
| Go Game Server | `go.mod` | GameServerBuild | 9 rules |
| C++ Game Server | `CMakeLists.txt` | CppServerBuild | 6 rules |
| Java Game Server | `pom.xml` / `build.gradle` + game keywords | JavaServerBuild | 6 rules |
| Node.js Game Server | `package.json` + game framework detection | NodeServerBuild | 5 rules |

4 universal review rules are automatically applied to all projects.

---

## 18 Game-Specific Tools

| Tool | Purpose | Applicable Engines |
|------|---------|-------------------|
| CSharpSyntaxCheck | Roslyn instant syntax check | Unity, Godot |
| UnityBuild | Unity build pipeline | Unity |
| UnrealBuild | UBT compilation | UE |
| GodotBuild | GDScript/C# check | Godot |
| GameServerBuild | Tiered verification (lint/build/test) | Go Server |
| CppServerBuild | C++ server build (CMake/Ninja + cppcheck + ctest) | C++ Server |
| JavaServerBuild | Java server build (Maven/Gradle + checkstyle + JUnit) | Java Server |
| NodeServerBuild | Node.js server build (tsc + ESLint + Vitest) | Node.js Server |
| ProtoCompile | Protobuf compilation + stub code generation | Cross-engine |
| ProtoCompat | Protobuf breaking change detection | Cross-engine |
| PerfLint | Game hot-path static performance analysis | Unity, UE, Godot |
| ShaderCheck | Shader static validation (variant explosion, sampler limits, syntax) | Unity, UE, Godot |
| ConfigGenerate | Config table generation | Cross-engine |
| OrmGenerate | ORM code generation | Go Server |
| ScoreReview | 100-point scoring review | Cross-engine |
| GateChain | Gate Chain orchestration | Cross-engine |
| KnowledgeSediment | Automatic knowledge documentation | Cross-engine |
| ArchitectureGuard | Dependency direction check | Cross-engine |
| AssetCheck | Asset reference integrity + naming conventions + deep nesting detection | Unity, UE, Godot |

---

## Phase 4: Performance & Multi-Language Optimization (v0.2.0)

v0.2.0 adds 5 new tools, enhances 1 tool, extends support for 3 server-side languages, and adds 1 new Git Hook.

### New Tools

#### PerfLint — Game Hot-Path Static Performance Analysis

Automatically detects performance anti-patterns in hot paths like Update/Tick/_process:

**Unity (7 rules)**:

```csharp
// ❌ PerfLint will flag: GetComponent in Update
void Update() {
    var rb = GetComponent<Rigidbody>();  // GetComponent every frame
    rb.AddForce(Vector3.up);
}

// ✅ Correct: cache the reference
private Rigidbody _rb;
void Awake() { _rb = GetComponent<Rigidbody>(); }
void Update() { _rb.AddForce(Vector3.up); }
```

Detections: `GetComponent`, `Camera.main`, `Instantiate`, `Find`, LINQ in Update, uncached `WaitForSeconds`.

**Unreal (6 rules)**: `FindActor`, `Cast`, `NewObject`, `FString +` in Tick, `LineTrace` frequency.

**Godot (5 rules)**: `get_node`, `instantiate`, `get_children` in `_process`, signal connect/disconnect pairing.

#### ProtoCompat — Protobuf Breaking Change Detection

Analyzes git diff of `.proto` files to detect changes that may break client compatibility:

```bash
# Usage example: check proto changes on current branch
danya -p "use ProtoCompat to check compatibility of the proto/ directory"
```

| Severity | Detections |
|----------|------------|
| CRITICAL | Field number change, field type change, enum value renumbering |
| HIGH | Deleted field without reserve, deleted RPC method, deleted service |

#### ShaderCheck — Shader Static Validation

Supports Unity (ShaderLab/HLSL), Unreal (HLSL/USF), Godot (GDScript Shader):

```
# Detection report example
[CRITICAL] weapon_skin.shader: multi_compile combination count 512 > 256 limit (variant explosion)
[HIGH] water_surface.shader: sampler count 18 > 16 limit
[MEDIUM] particle_effect.shader: fragment shader function complexity too high (nested loops >3 levels)
```

#### CppServerBuild — C++ Server Build

Supports quick/build/full three-tier verification, auto-detects CMakeLists.txt:

| Level | Execution |
|-------|-----------|
| quick | cppcheck static analysis |
| build | CMake + Ninja compilation |
| full | build + ctest tests |

#### JavaServerBuild — Java Server Build

Auto-detects Maven (pom.xml) vs Gradle (build.gradle):

| Level | Maven | Gradle |
|-------|-------|--------|
| quick | `mvn checkstyle:check` | `gradle checkstyleMain` |
| build | `mvn compile` | `gradle build` |
| full | build + `mvn test` | build + `gradle test` |

#### NodeServerBuild — Node.js Server Build

Auto-detects bun vs npm, supports colyseus / socket.io and other game frameworks:

| Level | Execution |
|-------|-----------|
| quick | ESLint |
| build | tsc compilation |
| full | build + Vitest tests |

### AssetCheck Enhancements

**New Unreal Engine support**:
- Naming convention checks: SM_ (static mesh), T_ (texture), M_ (material), BP_ (blueprint) prefixes
- Source asset size warnings: textures >50MB, meshes >100MB
- Soft reference validation

**Deep nesting detection** (all engines):
- Unity: Prefab Transform parent chain >10 levels
- Godot: Scene node depth >10 levels

**Inactive large object detection**:
- Unity scenes: inactive GameObjects with >20 children

### AssetGuard Hook

Pre-commit stage automatically blocks large binary files not tracked by Git LFS (default >5MB):

```bash
# Custom threshold (in bytes)
export ASSET_GUARD_THRESHOLD=10485760  # 10MB

# When .gitattributes is missing, a warning is issued:
# [WARN] No .gitattributes found — consider configuring Git LFS
# [BLOCK] Assets/Textures/hero.psd (12.3MB) not tracked by Git LFS
```

---

## Complete Command Quick Reference

### In-Conversation (AI-Driven Mode)

| Command | Action |
|---------|--------|
| `/auto-work "requirement"` | Fully automatic pipeline |
| `/auto-bugfix "bug"` | Automatic bug fix |
| `/review` | Score-based review |
| `/fix-harness` | Self-evolution |
| `/plan "requirement"` | Planning |
| `/verify [level]` | Verification |
| `/parallel-execute prepare "feature"` | Parallel development |
| `/orchestrate <task.md>` | Auto-iteration |
| `/red-blue [scope]` | Red-blue adversarial |
| `/monitor [metric] [days]` | View effectiveness |
| `/model` | Model management |
| `/compact` | Manual compression |
| `/compact-threshold <ratio>` | Adjust compression threshold |
| `/init` | Initialize Harness |

### Terminal (Shell-Enforced Mode)

| Command | Action |
|---------|--------|
| `danya auto-work "requirement"` | Shell-enforced pipeline |
| `danya parallel <tasks-dir>` | Multi-Agent parallelism |
| `danya red-blue [scope]` | Red-blue adversarial |
| `danya orchestrate <task.md>` | Auto-iteration |
| `danya analyze --metric <m>` | Data analysis |
| `danya dashboard [-w] [-v]` | Real-time monitoring |
| `danya report` | Monthly report |
| `danya check-env` | Environment check |
