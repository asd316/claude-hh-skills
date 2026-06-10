---
name: hh:trace
description: Generate an interactive HTML execution trace report from a Claude Code session transcript. Visualize tool calls, skill invocations, agent spawns as a flame chart. Detect bottlenecks (time gaps, repeated calls, slow agents). Include LLM reflection on effectiveness and improvement suggestions.
---

# hh:trace — Execution Trace Visualizer

Analyze a Claude Code session transcript and generate a self-contained interactive HTML report showing the execution timeline, tool call distribution, and bottleneck detection.

## When to Use

- User runs `/hh:trace` — analyze the current session
- User says "trace this session", "执行追踪", "看看这次执行怎么样"
- **Stop hook** — auto-generate report when session ends

## Universal Design (Works in ANY Project)

This skill has zero project dependencies. It reads directly from `~/.claude/projects/` and generates self-contained HTML. It works regardless of the project's language, structure, or config.

## Phase 1 — Discover Data Source

### 1a: Determine the session ID

If user passed `--session-id <id>`, use it. Otherwise, auto-detect:

```bash
# The current session's transcript is being written as we run.
# Find it by looking at the most recently modified transcript in this project's dir.
PROJECT_SLUG=$(echo "$PWD" | sed 's|/|-|g')
TRANSCRIPT_DIR="$HOME/.claude/projects/$PROJECT_SLUG"
ls -t "$TRANSCRIPT_DIR"/*.jsonl 2>/dev/null | head -1
```

If multiple transcripts exist and the session ID is unknown, analyze the newest one.

### 1b: Verify the transcript exists

If the transcript is empty or doesn't exist yet (trace called too early in session), report: "该会话尚无足够的执行记录。等待更多操作后再运行 /hh:trace。"

## Phase 2 — Parse Transcript

Read the JSONL file line by line. Each line is a JSON object with a `type` field.

### Event types relevant to trace

| type | What it records | Key fields |
|------|----------------|------------|
| `assistant` | Claude's response with embedded tool_use blocks | `message.content[]` — each block of type `tool_use` has `name` (tool name), `id` (call ID), `input` |
| `attachment` | Tool result (or hook output) | `timestamp` (ISO 8601), `parentUuid` (links to the assistant message that made the call), `attachment` (result content) |
| `user` | User prompt | `message.content` — the user's text |
| `system` | System messages / reminders | `content` |
| `mode` / `permission-mode` | Session mode changes | `mode`, `permissionMode` |
| `worktree-state` | Git worktree events | Worktree name, branch |

### Build the event timeline — GROUP BY CONVERSATION TURNS (CRITICAL)

**Do NOT create one event per tool call.** The flame chart becomes unreadable with 100+ individual tool call rows.

Instead, group events into **conversation turns**: each `user` message starts a new turn, followed by one or more `assistant` responses with their tool calls.

**Pseudocode (MUST follow this exactly):**

```python
turns = []
current_turn = None

for record in transcript_lines:
    type = record['type']
    ts = record.get('timestamp', '')
    
    if type == 'user':
        # Close previous turn, start new one
        if current_turn: turns.append(current_turn)
        user_text = extract_text(record['message']['content'])
        current_turn = {
            'user_ts': ts,
            'user_msg': user_text,
            'end_ts': ts,        # Initialize with user timestamp
            'tools': [],          # List of tool_use blocks
        }
    
    elif type == 'assistant' and current_turn:
        # Collect ALL tool_use blocks from this assistant message
        for block in record['message'].get('content', []):
            if block.get('type') == 'tool_use':
                preview = extract_preview(block)
                current_turn['tools'].append({
                    'name': block['name'],
                    'preview': preview,
                })
    
    elif type == 'attachment' and current_turn:
        # Update end_ts to the LATEST tool result time
        if ts:
            current_turn['end_ts'] = ts

if current_turn: turns.append(current_turn)
```

**After building turns, filter to phases (only turns with tool calls):**
- `phases = [t for t in turns if t['tools']]` — skip empty turns
- Each phase keeps: `turn_idx` (index in turns array), `label`, `user_ts`, `end_ts`, `user_msg`, `tool_count`, `skills[]`, `agents[]`, `tools[]`

**Then filter to key_turns for L3 (only noteworthy phases):**
```python
key_turns = []
for p in phases:
    is_user = p['turn_idx'] in intervention_indices
    has_skill = len(p['skills']) > 0
    has_agent = len(p['agents']) > 0
    has_edit = any(t['name'] in ('Edit','Write') for t in p['tools'])
    has_git = any('git ' in t.get('preview','').lower() or 'push' in t.get('preview','').lower() 
                  for t in p['tools'] if t['name'] == 'Bash')
    if is_user or has_skill or has_agent or has_edit or has_git:
        key_turns.append(p)
```

**Finally group key_turns into interaction blocks (user → responses):**
```python
blocks = []
current_block = None
for kt in key_turns:
    if kt['turn_idx'] in intervention_indices:
        if current_block: blocks.append(current_block)
        current_block = {'user': kt, 'responses': []}
    elif current_block:
        current_block['responses'].append(kt)
if current_block: blocks.append(current_block)
```

### Extract user intervention markers (CRITICAL)

Filter turns where the user message is substantive. Skip system-generated messages:

**MUST filter out:**
- Messages matching `<local-command-caveat>...</local-command-caveat>`
- Messages matching `<command-name>...</command-name>` or `<command-message>...</command-message>`
- Messages matching `<local-command-stdout>...</local-command-stdout>`
- "Continue from where you left off." (auto-resume stubs)
- "Commands are in the form /command [args]" (system help)
- `/exit` and `Bye!` (session exit)
- `Base directory for this skill:` (skill-load stubs)
- Pure whitespace or messages ≤ 3 chars

Use regex: filter out messages matching `<[^>]+>` XML tag patterns plus the literal strings above.

**Keep only messages where** `msg.strip()` is non-empty AND passes all filter checks. These become user intervention markers displayed in L2.

### Classify tools

| Category | Tool names |
|----------|-----------|
| **Skill** | `Skill` |
| **Agent** | `Agent` |
| **Read** | `Read`, `mcp__codegraph__codegraph_*` (read-only MCP tools) |
| **Edit** | `Edit`, `Write`, `NotebookEdit` |
| **Bash** | `Bash` |
| **Task** | `TaskCreate`, `TaskUpdate`, `TaskList`, `TaskGet` |
| **Git** | `Bash` calls matching `git *` pattern |
| **Other** | Everything else |

### Collapse rules (medium granularity)

- Consecutive `Read`/`Grep`/`codegraph_*` calls with the same session turn → "探索了 N 个文件"
- Consecutive `Edit` calls to the same file → "修改文件 X（N 次编辑）"
- `TaskCreate`/`TaskUpdate`/`TaskList` → group as "任务管理（N 次操作）"
- Single `Bash` commands AND `git *` commands → show individually (they're important events)

### Detect bottlenecks

For each consecutive pair of tool calls in the same assistant turn:
- **Time gap**: if `attachment.timestamp` of result N and `attachment.timestamp` of call N+1 differ by >30s → mark as `slow_gap`
- **Repeated call**: same tool name + same input appearing >3 times consecutively → mark as `retry_loop`
- **Slow agent**: `Agent` tool call where result timestamp - call timestamp >2min → mark as `slow_agent`

### Track skill invocations

For each `Skill` tool call, extract `input.skill` (the skill name). Record:
- Skill name
- Start time (when the call was made)
- End time (when the result attachment arrived)
- The assistant turn it was called in

## Phase 3 — Generate HTML Report

### Output path

```bash
# Priority 1: project's existing visualization directory
if [ -d "docs/visualization" ]; then
    OUTPUT="docs/visualization/trace/trace-$(date +%Y%m%d-%H%M%S).html"
else
    OUTPUT="$HOME/.claude/skills/hh:trace/tmp/trace-$(date +%Y%m%d-%H%M%S).html"
fi
mkdir -p "$(dirname "$OUTPUT")"
```

### Information Architecture (L0 → L3)

```
┌──────────────────────────────────────────────────────┐
│ L0: 执行效能评估                                        │
│   - 用户意图达成度                                       │
│   - 关键路径耗时概览                                     │
│   - 偏离/返工情况                                       │
│   - 💡 LLM 反思：如果可以重来，怎么执行会更优              │
├──────────────────────────────────────────────────────┤
│ L1: 关键指标 (≤6 卡片)                                  │
│   会话时长 | Tool Call 总数(N 类) | Skill 调用(N 个)     │
│   Agent 活动(N spawn, M min) | 卡点(N 个) | 交互轮次     │
├──────────────────────────────────────────────────────┤
│ L2: 对话轮次火焰图 + 用户标记 (Canvas)                     │
│   - 左侧：用户对话标记列表（💬），点击跳转到对应轮次           │
│   - 右侧：火焰图，每行 = 一个对话轮次                         │
│   - 行宽 = 该轮 tool call 数量（按比例）                     │
│   - 颜色按轮次主要操作分类：                                   │
│     Skill调用=紫, Agent实施=蓝, 代码修改=绿,                  │
│     Git操作=橙, 探索/诊断=灰, 混合操作=青                    │
│   - 用户消息轮次用 ★ 标记                                    │
│   - 点击轮次行：展开显示该轮所有 tool call 详情                │
│   - 卡点高亮: 黄(间隔>30s) 橙(重复>3) 红(Agent>2min)        │
├──────────────────────────────────────────────────────┤
│ L3: 详情                                                │
│   - Skill 调用卡片 + Agent Spawn 卡片                    │
│   - Tool 分类分布（按类型统计数量）                        │
│   - 所有轮次可展开列表（默认前 3 轮展开，其余折叠）          │
│   - 每个展开轮次显示 tool call 标签云                      │
└──────────────────────────────────────────────────────┘
```

### L0 Verdict — MUST include

1. **执行效能一句话**: "本次会话 [N] 分钟，执行 [N] 个 tool call，调用 [N] 个 Skill，spawn [N] 个 Agent。整体执行 [顺畅/有 N 处可优化]。"
2. **用户意图达成度**: 检查用户最初消息和目标完成状态，判断 "✅ 全部完成" / "⚠️ 部分完成" / "❌ 未完成"
3. **关键路径耗时**: 列出主要阶段及其耗时（如：探索 5min / 修改 3min / 测试 2min）
4. **偏离/返工**: 列出需要重试或返工的操作
5. **LLM 反思建议**: 基于 trace 数据，由当前 LLM 生成 2-3 条具体建议。思考角度：
   - 如果用户一开始就指定 X，可以省掉 Y 步骤
   - 如果先做 A 再做 B，可以避免 C 次返工
   - 如果用 worktree 隔离修改，合并更安全
   - 如果有更明确的目标描述，可以减少探索级调用
   - 如果指定了 effor 级别，可以提高/降低审查深度

### L2 Flame Chart — turn-level Canvas rendering (CRITICAL)

**MUST render at conversation turn level, NOT individual tool call level.** 100+ individual tool call rows are unreadable.

Embed a `<script>` block using Canvas API (zero dependencies).

#### Step A: Build phase-to-DOM mapping (P2D) — MUST INCLUDE

**P2D maps flame chart row index → L3 DOM element ID.** Without it, click navigation is broken for most bars. This is the #1 bug from real-world testing.

Generate the mapping during data preparation (Python), embed as JS constant:

```python
# After building interaction_blocks, create P2D mapping:
p2d = {}
for block in interaction_blocks:
    start = block['user']['turn_idx']
    # End = next block's user turn, or last phase index + 1
    next_starts = [b['user']['turn_idx'] for b in interaction_blocks 
                   if b['user']['turn_idx'] > start]
    end = min(next_starts) if next_starts else max_turn_idx + 1
    for pi in range(start, end):
        p2d[pi] = start  # Maps EVERY phase index to its parent block's DOM id

# Embed as: const P2D = {json.dumps(p2d)};
```

#### Step B: Canvas click handler — MUST USE P2D

```javascript
canvas.addEventListener('click', (e) => {
    const rowIdx = Math.floor((e.clientY - canvas.getBoundingClientRect().top) / (ROW_H + GAP));
    if (rowIdx >= 0 && rowIdx < activePhases.length) {
        const phaseIdx = activePhases[rowIdx].i;  // the turn index
        const domId = P2D[String(phaseIdx)];       // lookup via mapping
        if (domId !== undefined) {
            const target = document.getElementById('block-' + domId);
            if (target) {
                target.scrollIntoView({behavior:'smooth', block:'center'});
                target.style.background = '#fef3c7';
                setTimeout(() => target.style.background = '', 2000);
            }
        }
    }
});
```

**DO NOT do `document.getElementById('block-' + phaseIdx)` — that's the bug.** P2D lookup is mandatory.

#### Step C: Canvas rendering loop

```javascript
let y = 2, prevDate = '';
activePhases.forEach(p => {
    const w = Math.max((p.c / maxTools) * (W - 10), 4);
    // ★ for user intervention turns
    if (userTurnIndices.has(p.i)) {
        ctx.fillStyle = '#f59e0b'; ctx.font = 'bold 10px sans-serif'; ctx.fillText('★', 1, y+11);
    }
    // Cross-day dashed separator
    const curDate = (p.e || p.t || '').substring(0, 10);
    if (prevDate && curDate && curDate !== prevDate) {
        ctx.strokeStyle = '#e2e8f0'; ctx.setLineDash([4, 4]);
        ctx.beginPath(); ctx.moveTo(0, y-1); ctx.lineTo(W, y-1); ctx.stroke();
        ctx.setLineDash([]);
        ctx.fillStyle = '#94a3b8'; ctx.font = '9px sans-serif'; ctx.fillText(curDate, W-80, y-3);
        y += 6;
    }
    prevDate = curDate;
    ctx.fillStyle = CLR[p.l] || '#64748b';
    ctx.fillRect(15, y, Math.min(w, W-15), ROW_H);
    if (w > 25) {
        ctx.fillStyle = '#fff'; ctx.font = '8px monospace'; ctx.fillText('#' + p.i, 18, y+11);
    }
    y += ROW_H + GAP;
});
```

**Turn phase classification** (determines bar color):
- Has `Skill` call and no Agent/Edit → "Skill 调用" (purple)
- Has `Agent` spawn → "Agent 实施" (blue)
- Has `Edit`/`Write` and no Agent → "代码修改" (green)
- Has `Bash` with git commands → "Git 操作" (orange)
- Only `Read`/`Bash` exploration → "探索/诊断" (gray)
- Mixed or other → "混合操作" (cyan)

**Cross-day handling (CRITICAL)**:
- Sessions often span multiple days (user pauses, resumes later)
- `end_ts` is monotonically increasing within same day, but HH:MM:SS display breaks at day boundary (e.g., "06:25 → 03:49" looks like time went backwards)
- **Must detect date changes**: compare `end_ts.substring(0,10)` between consecutive turns
- In flame chart: draw dashed separator line at day boundary with date label
- In L3 responses: show `MM-DD HH:MM` format when date differs from parent user message's date; show date separator badge between turns when date changes

**User intervention markers** — MUST be displayed prominently in L2:
- Left column: scrollable list of user messages with timestamps
- Each marker is clickable → scrolls to and highlights the corresponding turn in L3
- Filter out system messages (skill-load stubs: "Base directory for this skill")
- Show user message preview (first 150 chars)

Color scheme:
- Skill: `#7c3aed` (purple)
- Agent: `#2563eb` (blue)
- Read: `#94a3b8` (slate gray)
- Edit: `#059669` (emerald)
- Bash: `#0891b2` (cyan)
- Git: `#ea580c` (orange)
- Task: `#d97706` (amber)
- Other: `#64748b` (slate)
- Bottleneck gap: yellow stripe pattern
- Bottleneck retry: orange stripe pattern
- Bottleneck agent: red border

### L3: Key Interactions (NOT all turns)

**MUST filter to only noteworthy turns.** Showing all 150+ turns is useless noise.

**Show only turns where:**
- User sent a substantive message (user intervention)
- A `Skill` was called
- An `Agent` was spawned
- Code was edited (`Edit`/`Write`)
- Git operations happened (`git add/commit/push/merge/stash`)

**Filter out** pure exploration turns (only `Read`/`Bash`/`Glob` without edits).

**Group by user intervention blocks**: user said X → Claude responded with Y (sequence of key turns). Each block shows the user message header followed by indented response turns.

**Response turn timestamps**: use `end_ts` (last tool result from `attachment` records), NOT `user_ts`. For non-user turns, `user_ts` is often empty or stale. `end_ts` is the accurate wall-clock time when Claude finished responding.

**Cross-day handling**: when a response turn's date differs from its parent user message, show `MM-DD HH:MM` format. Insert a date separator `<── 2026-06-10 ──>` between turns when the date changes.

### Design Standards (MANDATORY)

- **White background**: `body{background:#fff;color:#1e293b}`
- **Card style**: `background:#f8fafc;border-radius:12px;border:1px solid #e2e8f0`
- **User markers**: `background:#fffbeb;border:1px solid #fde68a` — MUST have visible background on white
- **Tool chips**: use 8-digit hex with transparency for background (`#7c3aed18`), solid color for text. Format: `font-size:10px;padding:2px 8px;border-radius:10px`
- **Interaction blocks**: left-border color coding — user=amber, skill=purple, agent=blue, edit=emerald, git=orange
- **Canvas flame chart**: zero external dependencies, pure Canvas API
- **Phase-to-DOM mapping (P2D)**: `phase_idx → user_turn_idx` for click navigation. Both indices differ — MUST build mapping table, NOT concatenate blindly

### L3: Tool category distribution

## Phase 4 — Post-Generation

1. Write the HTML file to the output path
2. Open with `open <path>` (macOS) or `xdg-open <path>` (Linux)
3. Report the path to the user
4. If this was triggered by Stop hook, also print a summary line to stdout

## Stop Hook Setup (Optional)

To auto-generate trace on session end, add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python3 -c \"\nimport subprocess, sys\n# Let transcript finalize\nimport time; time.sleep(0.5)\n# Find newest transcript\nimport os, glob\nproj = os.getcwd().replace('/', '-')\nd = os.path.expanduser(f'~/.claude/projects/{proj}')\nfiles = glob.glob(f'{d}/*.jsonl')\nif files:\n    latest = max(files, key=os.path.getmtime)\n    print(f'[hh:trace] Session ended. Run /hh:trace --session-id {os.path.basename(latest).replace(chr(46)+chr(106)+chr(115)+chr(111)+chr(110)+chr(108), chr(46)+chr(104)+chr(116)+chr(109)+chr(108))} to see execution trace.')\n\""
      }]
    }]
  }
}
```

Note: The stop hook command above is a simplified reminder. The actual trace generation is done by Claude when the user invokes `/hh:trace` manually. The hook just prints the session ID for reference.

## Edge Cases

- **Empty transcript** (new session): Report "尚无足够的执行记录"
- **Huge transcript** (>10k lines): Sample the most recent 10k lines, note truncation in report
- **No Skill calls**: Show "本次会话未调用任何 Skill" in L1
- **No Agent spawns**: Show "本次会话未 spawn Agent" in L1
- **Session still in progress** (manual trigger mid-session): Analyze up to the latest written line, mark report as "进行中"
- **Missing timestamps**: If `attachment.timestamp` is null, estimate from line ordering
