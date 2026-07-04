---
name: hh:runbook
description: >
  Generate operational documentation and sync conversation insights to project docs. Two modes: (A) quick-start docs for a feature with Run/Stop/Debug/FAQ sections, (B) full doc sync scanning conversation for decisions/knowledge to persist. Use after implementing a feature, when user says "写文档", "怎么用", or wants to sync conversation to docs.
---

# hh:runbook — Documentation & Sync

Two modes, auto-detected from context.

---

## Mode A: Quick-Start Doc (after implementation)

**Trigger:** User just implemented a feature, or says "使用方法写在 docs/quick-start", "怎么运行", "写个操作文档".

### Template

Generate `docs/quick-start/{feature-name}.md`:

```markdown
# {Feature Name}

## 前置条件
- [Prerequisite 1]
- [Prerequisite 2]

## 运行

```bash
python {entry_point}                  # Normal run
python {entry_point} --group 群名     # Specific group
```

## 停止

```bash
launchctl stop {label}                # If running as scheduled task
# OR
Ctrl+C                                # If running manually
```

## 调试

```bash
tail -f logs/{log_file}               # Watch logs
python {entry_point} --dry-run        # Preview without writing (if supported)
python {entry_point} --undo           # Rollback (if supported)
```

## 常见问题

### Q: {common issue 1}?
A: {solution}

### Q: {common issue 2}?
A: {solution}

## 相关文档
- [Related doc 1](path)
```

### After Writing

Update `docs/quick-start/INDEX.md` with the new entry.

---

## Mode B: Full Doc Sync (conversation → docs)

**Trigger:** End of a substantial conversation, user says "同步文档", "sync", or runs `/hh:runbook` with no specific feature.

### Step 1: Scan Existing Docs

Read `CLAUDE.md` and browse `docs/` to understand current structure and style.

### Step 2: Review Conversation

Identify items with long-term value (ignore debugging detours, temporary fixes, inconclusive discussions):

- **Concepts & terminology**: new terms introduced, domain vocabulary
- **Logic & flow**: data processing steps, algorithms, pipelines
- **Design decisions**: why X over Y, tradeoffs made, alternatives rejected
- **Conventions & agreements**: coding patterns, naming rules, team practices
- **Pitfalls & edge cases**: non-obvious constraints, foot-guns discovered
- **Errors discovered & fixed**: new error patterns with root cause + fix → MUST update `docs/error-log.md`
- **Project guidance**: new commands, tech stack changes, architecture shifts → `CLAUDE.md`
- **Graph impact**: if project skeleton changed (new phase, new pipeline, new constraint) → suggest running `/hh:understand` to update the knowledge graph

If nothing worth persisting, say so and stop.

### Step 3: Propose Changes (don't execute yet)

```
## 同步建议

### 本轮沉淀要点
- [要点 1]
- [要点 2]

### 建议更新的文档
1. [file path]
   - 类型：新建 / 追加 / 修改
   - 内容概要：...

2. [file path]
   ...

### 是否执行？[Y/n/选部分]
```

**Attribution rules:**
- Has related existing doc → append at appropriate section
- No fitting doc → create new file in `docs/` (clear English or pinyin filename)
- Project-level index/status/command changes → update `CLAUDE.md`
- New error discovered → update `docs/error-log.md`
- Created a new doc → add an index entry in `CLAUDE.md`
- Architecture/pipeline/stage change → suggest running `/hh:understand` to update `docs/graph/project-graph.json`

### Step 4: Execute After Confirmation

- **Incremental write**: only add new content, no duplication
- **Match style**: follow target doc's heading levels, format, tone
- **Update todo/INDEX.md**: if new tasks were identified during sync
- **Update error-log.md**: if new error patterns were discovered and fixed — follow the existing format (现象 → 根因 → 解决 → 修复位置)
- **Update adr/**: if a significant architectural decision was made

### Step 5: Offer hh-remember (constraint/error harvesting)

After sync is complete, check if the conversation revealed any **constraints or error patterns** that should be persisted:

> 本轮是否发现了需要记住的约束或踩坑经验？如果需要，可以执行 `/hh-remember` 来收割这些内容到 Constraints.md / error-log.md / .claude/rules/。

- Only ask if there are actual constraints or errors worth persisting (not every sync needs it)
- Don't force it — this is a genuine offer, not a requirement
- If user says yes → invoke `hh-remember` skill
- If user says no → move on

Note: hh-remember does NOT reciprocate (it won't suggest hh-runbook). The dependency is one-way.

### Step 6: Check User Profile freshness

If `docs/user-profile.md` exists, check whether this conversation revealed new information about the user's goals, fears, or hidden knowledge that contradicts or extends the profile. If so:

> 本轮对话中发现了新的用户背景/目标/约束信息。是否更新 `docs/user-profile.md`？

If `docs/user-profile.md` does NOT exist and this was a complex multi-turn session with visible goal drift or repeated corrections:

> 建议运行 `/hh-user-profile-extraction` 来建立用户画像，减少未来 agent 走偏。

---

## Shared Rules

- **Prefer simplicity**: docs should be skimmable, not comprehensive
- **Ask before writing**: important changes require user confirmation
- **Index discipline**: new files → update CLAUDE.md doc navigation table
- **No duplication**: check existing docs before adding content
