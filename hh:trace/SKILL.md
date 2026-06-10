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

### Build the event timeline

1. Iterate through all lines, collecting events
2. For each `assistant` message, extract ALL `tool_use` blocks → these are tool calls
3. Match each tool call to its result: tool call has `id`, result `attachment` has `parentUuid` matching the assistant's `uuid`
4. Calculate elapsed time between a tool call and its result (from `attachment.timestamp`)

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
│ L2: 火焰图 (Canvas, 可展开/折叠)                         │
│   - 横轴 = 时间, 每行 = 一个 tool call / skill / agent   │
│   - 颜色按类别: Skill=紫, Agent=蓝, Read=灰,              │
│     Edit=绿, Bash=青, Task=琥珀, Git=橙                  │
│   - 宽度 = 耗时                                         │
│   - 层级: Skill → Agent → tool calls (嵌套展开)          │
│   - 卡点高亮: 黄(间隔>30s) 橙(重复>3) 红(Agent>2min)     │
├──────────────────────────────────────────────────────┤
│ L3: 详情 (默认展开: Skill 卡片 + 卡点表; 默认折叠: 原始日志)│
│   - 每个 Skill: 调用时间、内部 tool 分布、持续时间          │
│   - 卡点详情: 位置、类型、相关 tool call、建议              │
│   - 原始事件列表: 可搜索表格 (时间/类型/工具/耗时)          │
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

### L2 Flame Chart — implementation

Embed a `<script>` block using Canvas API (zero dependencies):

```javascript
// Key rendering logic:
// 1. Parse the trace data (JSON embedded in HTML)
// 2. Calculate time range and layout rows
// 3. Draw rectangles colored by category
// 4. Width = duration proportional to total time
// 5. Nested: Skill creates a group row, click to expand/collapse child rows
// 6. Hover tooltip: tool name, duration, input preview
// 7. Bottleneck highlights: colored borders/stripes
```

The flame chart data is embedded as a JSON blob in the HTML: `<script id="trace-data" type="application/json">...</script>`. The rendering script reads it at page load.

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

### L3 Detail tables — styling

Use Tailwind CDN for table styling. Sortable headers (click to sort). Search input for filtering tool calls by name.

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
