---
name: hh-remember
description: Persist constraints and project knowledge across sessions. Primary focus is "DON'T do X" rules and foot-guns. Writes to Constraints.md, .claude/rules/ (auto-loaded by Claude Code), error-log.md, Context.md, and CLAUDE.md. Use when user says "记住", "以后别", "绝对不能", "一定要", "这是个教训", "踩坑了", "remember to never", "always check", "don't forget", when a bug was just fixed and should be logged, or when any non-obvious constraint is discovered that future agents must know. Also use proactively before major changes to recall existing constraints. Note: hh-runbook may suggest running this at the end of its session; this skill does NOT suggest hh-runbook.
---

# hh-remember — Cross-Session Project Memory

> **IDE Environment Note:** Throughout this skill, `CLAUDE.md` refers to the project root instruction file. If running in **Claude Code**, use `CLAUDE.md`. If running in **Cursor / Windsurf / TRAE / other IDEs**, use `AGENTS.md` instead. Detection: if `CLAUDE.md` exists in project root, use it; otherwise fall back to `AGENTS.md`.

Persistent knowledge that survives beyond the current conversation. The agent's job is to capture, categorize, format, and write it to the right file so future sessions benefit from it.

## Core Philosophy

The biggest waste in AI-assisted development is re-discovering the same constraints and re-making the same mistakes. This skill turns "I wish I had known that" into "the project already told me."

**The typical usage pattern:** a long conversation (几十轮) accumulates many implicit lessons — corrections, discoveries, bug fixes, design decisions. At the end (or at a natural pause), the user invokes this skill to "harvest" everything worth persisting before it evaporates.

## The 5 Target Files

These are the project's long-term memory. They exist in any project (or are created as needed). Their purposes are distinct and non-overlapping:

| File | Purpose | Loaded When | Question It Answers |
|------|---------|------------|---------------------|
| `.claude/rules/*.md` | **Critical constraints** auto-loaded every session by Claude Code | Every session | "What must I NEVER forget?" |
| `docs/Constraints.md` | Hard rules, invariants, gotchas | On-demand (read by agents) | "What must I NOT do?" |
| `docs/error-log.md` | Bug forensics | On-demand (read by agents) | "What broke before and how was it fixed?" |
| `docs/Context.md` | Domain knowledge, background | On-demand (read by agents) | "What is this project about?" |
| `CLAUDE.md` (or `AGENTS.md` in non-Claude Code IDEs) | Agent instructions, architecture | Every session (if present) | "How do I work on this project?" |

**Key distinction**:
- `.claude/rules/` = constraints that are SO critical they must be in context every session (auto-loaded by Claude Code). Use for "绝对不能删除 data/raw", "必须用 conda Python 3.13" level rules.
- `Constraints.md` = constraints worth documenting but not session-critical. Use for detailed explanations, category-organized rules.
- `error-log.md` = "X broke BECAUSE Z." Forensics, not rules.
- `Context.md` = "this project IS Y." Background, not rules.
- `CLAUDE.md` = "work THIS way." Architecture, commands, behavioral guidelines.

### When to use `.claude/rules/` vs `Constraints.md`

| Constraint severity | Write to |
|--------------------|----------|
| Violating this causes **data loss or corruption** | `.claude/rules/critical-constraints.md` |
| Violating this causes **wasted hours of debugging** | `.claude/rules/critical-constraints.md` |
| Violating this causes **wrong but recoverable output** | `docs/Constraints.md` |
| This is a **convention preference**, not a hard rule | `docs/Constraints.md` or `CLAUDE.md` |

Rule of thumb: if you'd put it on a sticky note on the monitor, it goes in `.claude/rules/`. If it belongs in the project handbook, it goes in `docs/Constraints.md`.

## Primary Mode: Session Scan (conversation → memory)

**This is the default mode.** When invoked without a specific "remember X" command, scan the entire conversation for knowledge worth persisting across sessions.

### Step 1: Scan the Conversation

Read through the conversation and identify items with long-term value. Use these heuristics to spot them:

#### Constraint Patterns (→ `.claude/rules/` or `Constraints.md`)
Look for moments where:
- User **corrected** the agent's approach: "不对，你应该先...", "不要这样，会..."
- User stated an **explicit prohibition**: "千万别...", "绝对不能...", "以后都别..."
- User explained **WHY something broke**: "因为 --force --no-api 组合会把数据清空"
- A **foot-gun** was discovered and confirmed: a flag combo, a file that shouldn't be touched, an order dependency
- User said **"记住"** or **"别忘了"** casually during the conversation (not just as an explicit command)
- A **convention** was established: "以后改 config 之前先跑个 diff", "改完 annotator 必须跑测试"

**Routing**: Critical constraints (data loss, hours wasted) → `.claude/rules/critical-constraints.md`. General constraints (conventions, recoverable issues) → `docs/Constraints.md`. See the severity table in §The 5 Target Files.

#### Context Patterns (→ Context.md)
Look for moments where:
- User explained **project background**: what the project is, who uses it, why it exists
- User defined **terminology**: "小数是...", "9N 平台是..."
- User described **data flow** or architecture that isn't in CLAUDE.md
- User explained **stakeholder relationships** or organizational context
- User clarified a **misconception** about the project's purpose

#### Error Patterns (→ error-log.md)
Look for moments where:
- A **bug was diagnosed** during the conversation (root cause found)
- A **fix was applied and verified**
- Something **unexpected happened** and was resolved
- User described a **past incident** that agents should know about
- An **intermittent failure** was traced to its cause

#### Behavioral Patterns (→ CLAUDE.md)
Look for moments where:
- User established a **workflow preference**: "以后改 pipeline 之前先..."
- User corrected the agent's **coding style** or approach
- A **project-specific convention** was taught: "这个项目的测试都放在 tests/ 不用 test_ 前缀"
- New **run commands** or configurations were added

#### What to IGNORE (not worth persisting)
- Debugging detours that went nowhere
- Temporary workarounds for one-off situations
- Inconclusive discussions
- Trivial syntax fixes
- Things already documented in the target files (always check before proposing)

### Step 2: Categorize and Format

For each item found, determine which file it belongs in and format it according to that file's conventions (see Format Reference below).

If an item spans multiple categories, propose entries in multiple files with cross-references.

### Step 3: Present the Harvest

Show a structured summary. This is the key output format:

```
## 本轮可沉淀的内容

### .claude/rules/critical-constraints.md (关键约束 — 每 session 自动加载)
1. **Never combine --force with --no-api** — 会静默清除 bot_api_answer 数据
   → 来源: 本轮修复的 bug，根因是这两个 flag 组合没有保护
   → 严重性: 数据丢失

### Constraints.md (硬约束 — 项目手册)
1. **Always use conda base Python 3.13** — 系统 Python 3.9 不兼容 str | None 语法
   → 来源: 环境配置问题排查

### Context.md (项目背景 — 新 agent 需要知道)
1. **小数 API 调用的双重目的** — 获取回答 + 推送监控平台（数据闭环）
   → 来源: 解释为什么必须本地调 API

### error-log.md (错误记录 — 踩过的坑)
1. **2026-06-09: --force --no-api 覆盖 bot_api_answer**
   现象/根因/解决/预防 完整记录
   → 来源: 本轮诊断 + 修复 + 验证

### CLAUDE.md (行为准则 — agent 怎么工作)
(本轮无新增)

### 跨文件关联
- error-log 条目 #1 → .claude/rules 条目 #1 (同一根因)
- .claude/rules 条目 #1 → Constraints.md (详细说明)

---
是否写入以上全部？[Y/n/选部分]
```

### Step 4: Write After Confirmation

After user confirms:
1. Read each target file (to find insertion points and avoid duplicates)
2. Write each entry following the file's format conventions
3. Add cross-references between related entries
4. Report what was written and where

## Secondary Mode: Quick Capture (user says "remember X")

When the user explicitly asks to remember something specific:

```
1. Categorize: which file does X belong in? (Use decision tree below)
2. Check: does this already exist in that file?
   - If yes and identical → tell user, skip
   - If yes but outdated → propose update
   - If no → proceed
3. Format: write the entry following the file's format conventions
4. Confirm: show the formatted entry, ask "Add this to <file>?"
5. Write: after confirmation, append to the file
```

### Quick Decision Tree

```
User says "remember X"
  ├─ X is a critical "NEVER/ALWAYS" rule (data loss, hours wasted) → .claude/rules/critical-constraints.md
  ├─ X is a general constraint/convention → Constraints.md
  ├─ X describes a bug/symptom/fix → error-log.md
  ├─ X describes project background/domain/purpose → Context.md
  ├─ X describes how to work on the code → CLAUDE.md
  └─ Ambiguous → present options, let user decide
```

## Tertiary Mode: Recall (before starting work)

When starting a significant task, proactively check safety rails:

```
1. .claude/rules/*.md — auto-loaded by Claude Code, so these are already in context. Review them.
2. Read docs/Constraints.md (if exists) → know what NOT to break
3. Read docs/error-log.md (if exists) → know what broke before
4. If constraints exist → actively check your plan against them
5. Flag any plan that violates a documented constraint BEFORE executing
```

This mode is lightweight — just reading, not writing. It's the "safety check" before making changes.

## File Format Reference

### .claude/rules/ (Critical Constraints — auto-loaded every session)

Claude Code automatically loads all `.md` files from `.claude/rules/` into every session context. Use this for constraints so critical that forgetting them would be catastrophic.

```markdown
# Critical Project Constraints

<!-- Each rule: one line, imperative, includes consequence -->
- **NEVER** combine `--force` with `--no-api` — silently clears all `bot_api_answer` data
- **ALWAYS** use conda base Python 3.13 — system Python 3.9 incompatible with `str | None` syntax
- **NEVER** delete `data/raw/store` — it's a symlink; modifying originals corrupts source data
```

Rules for `.claude/rules/`:
- **One file is usually enough**: `critical-constraints.md`. Only split if you have 20+ rules.
- **Each rule is one line**: imperative, bold keyword (NEVER/ALWAYS), includes WHY
- **Keep it short**: this file loads into EVERY session. Don't bloat it. If a constraint has long explanation, put the one-liner here and link to `docs/Constraints.md#section` for details.
- **Only truly critical rules**: data loss, hours wasted, unrecoverable states. If violating it is merely annoying, it goes in `docs/Constraints.md`.

### Constraints.md

```markdown
# 项目约束

## <Category Name>
- <constraint — specific, actionable, includes the WHY>
- <constraint>

## <Another Category>
- <constraint>
```

Rules:
- **Specific + why**: "Never combine --force with --no-api — 会静默清除 bot_api_answer" NOT "Be careful with flags"
- **One per bullet**
- **Categories are nouns**: "数据约束", "处理约束", "API约束", "部署约束"
- If file doesn't exist → create with heading + first category
- If file exists → find the best category section, append; create new category if needed

### Context.md

```markdown
# 项目上下文

## <Topic>
<explanatory text — what, why, how it matters>

## <Another Topic>
<explanatory text>
```

Rules:
- **Explain WHY it matters**, not just what it is
- **Define terminology** for newcomers
- **Use tables/diagrams** when they help
- Append new topics at end, or merge into existing related topics

### error-log.md

```markdown
# Error Log

## YYYY-MM-DD: <One-line symptom summary>

**现象**: <What was observed — concrete, specific, with data>

**根因**: <Root cause — the mechanism, not just the trigger>

**解决**: <What fixed it — specific actions taken>

**修复位置**: <Files changed, commit hashes if available>

**验证**: <How we confirmed the fix worked>

**预防**: <What rule or check would prevent this from happening again>

---
```

Rules:
- **Every section mandatory** — write "待确认" if unknown
- **Root cause ≠ trigger** — "I ran --force --no-api" is the trigger, not the root cause
- **Prevention is actionable** — "Add a guard" not "Be more careful"
- **Newest first** (reverse chronological)
- **Separator**: `---` between entries

### CLAUDE.md

CLAUDE.md has two zones:
1. **Behavioral guidelines** (top) — how agents should think and act
2. **Project architecture** (bottom, after `---`) — module map, data flow, run commands

When adding:
- Behavioral rules → top section, numbered heading
- Architecture/commands → bottom section
- Match existing style (language, heading levels)
- Don't duplicate existing content

## Relationship with hh-runbook

**One-way dependency**: `hh-runbook` → `hh-remember` only. hh-runbook may suggest running hh-remember at the end; hh-remember does NOT suggest hh-runbook.

| Aspect | hh-remember | hh-runbook |
|--------|-------------|------------|
| **Focus** | Constraints, errors, critical rules (what NOT to do) | Operational docs (how to use features) |
| **Output** | Append to `.claude/rules/`, Constraints.md, error-log.md, Context.md, CLAUDE.md | Generate `docs/quick-start/*.md`, usage guides |
| **Scan targets** | Foot-guns, bug forensics, prohibitions, conventions | Features, commands, debug procedures, FAQs |
| **When to use** | After discovering constraints, fixing bugs, clarifying hard rules | After implementing features, when user asks "怎么写文档" |
| **Invocation chain** | Standalone (doesn't call anything else) | May ask "要不要也执行 hh-remember?" at end |

## Cross-Platform Behavior

Works in ANY project:
- Finds existing files: `docs/Constraints.md`, `docs/Context.md`, `docs/error-log.md`, `CLAUDE.md` (also `GEMINI.md`, `AGENTS.md`)
- Creates missing files as needed
- Language follows project's existing docs (Chinese, English, mixed)
- No hardcoded paths

## Red Flags

- **Writing without scanning for duplicates** — always check the target file first
- **Vague constraints** — "Be careful" is useless; explain exactly what and why
- **Skipping the WHY** — every constraint and error entry must explain the reason
- **Wrong file** — a bug fix doesn't go in Context.md; a domain explanation doesn't go in error-log.md
- **Missing cross-references** — if a bug fix reveals a constraint, both files should point to each other
- **Over-harvesting** — not every correction is a constraint; not every bug is worth logging. If the lesson is obvious or the fix is trivial, skip it.
