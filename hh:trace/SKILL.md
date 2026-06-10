---
name: hh:trace
description: 从 Claude Code 会话 transcript 生成交互式 HTML 执行追踪报告。以火焰图可视化 tool call、skill 调用、agent spawn。检测卡点（时间间隔、重复调用、慢 agent）。包含 LLM 对执行效能的反思和改进建议。
---

# hh:trace — 执行追踪可视化

分析 Claude Code 会话 transcript，生成自包含交互式 HTML 报告，展示执行时间线、tool call 分布和卡点检测。

## 何时使用

- 用户运行 `/hh:trace` —— 分析当前会话
- 用户说 "trace this session"、"执行追踪"、"看看这次执行怎么样"
- **Stop hook** —— 会话结束时自动生成报告

## 通用设计（适用于任意项目）

零项目依赖。直接从 `~/.claude/projects/` 读取 transcript，生成自包含 HTML。不依赖项目语言、结构或配置。

## 阶段 1 —— 发现数据源

### 1a：确定会话 ID

如果用户传了 `--session-id <id>`，使用它。否则自动检测：

```bash
# 当前会话的 transcript 正在实时写入
# 找到本项目目录下最近修改的 transcript
PROJECT_SLUG=$(echo "$PWD" | sed 's|/|-|g')
TRANSCRIPT_DIR="$HOME/.claude/projects/$PROJECT_SLUG"
ls -t "$TRANSCRIPT_DIR"/*.jsonl 2>/dev/null | head -1
```

如果存在多个 transcript 且不知道会话 ID，分析最新的那个。

### 1b：验证 transcript 存在

如果 transcript 为空或尚不存在（会话刚开始就运行 trace），报告："该会话尚无足够的执行记录。等待更多操作后再运行 /hh:trace。"

## 阶段 2 —— 解析 Transcript

逐行读取 JSONL 文件。每行是一个 JSON 对象，含 `type` 字段。

### 与 trace 相关的事件类型

| type | 记录内容 | 关键字段 |
|------|---------|---------|
| `assistant` | Claude 的响应，内嵌 tool_use 块 | `message.content[]` — 每块 type 为 `tool_use`，含 `name`（工具名）、`id`（调用 ID）、`input` |
| `attachment` | 工具结果（或 hook 输出） | `timestamp`（ISO 8601）、`parentUuid`（关联到发起调用的 assistant 消息）、`attachment`（结果内容） |
| `user` | 用户提示词 | `message.content` — 用户文本 |
| `system` | 系统消息 / 提醒 | `content` |
| `mode` / `permission-mode` | 会话模式变化 | `mode`、`permissionMode` |
| `worktree-state` | Git worktree 事件 | 工作树名称、分支 |

### 构建事件时间线 —— 按对话轮次分组（关键）

**禁止每个 tool call 创建一个事件。** 100+ 个独立 tool call 行在火焰图中完全不可读。

必须将事件分组为**对话轮次**：每条 `user` 消息开启一个新轮次，后跟一个或多个 `assistant` 响应及其 tool call。

**伪代码（必须严格遵循）：**

```python
turns = []
current_turn = None

for record in transcript_lines:
    type = record['type']
    ts = record.get('timestamp', '')
    
    if type == 'user':
        # 关闭上一轮，开始新一轮
        if current_turn: turns.append(current_turn)
        user_text = extract_text(record['message']['content'])
        current_turn = {
            'user_ts': ts,
            'user_msg': user_text,
            'end_ts': ts,        # 初始化为用户消息时间戳
            'tools': [],          # tool_use 块列表
        }
    
    elif type == 'assistant' and current_turn:
        # 收集此 assistant 消息中所有 tool_use 块
        for block in record['message'].get('content', []):
            if block.get('type') == 'tool_use':
                preview = extract_preview(block)
                current_turn['tools'].append({
                    'name': block['name'],
                    'preview': preview,
                })
    
    elif type == 'attachment' and current_turn:
        # 更新 end_ts 为最新的工具结果时间
        if ts:
            current_turn['end_ts'] = ts

if current_turn: turns.append(current_turn)
```

**构建 turns 后，过滤为 phases（仅保留含 tool call 的轮次）：**
- `phases = [t for t in turns if t['tools']]` — 跳过空轮次
- 每个 phase 保留：`turn_idx`（turns 数组中的索引）、`label`、`user_ts`、`end_ts`、`user_msg`、`tool_count`、`skills[]`、`agents[]`、`tools[]`

**然后过滤为 key_turns（L3 只展示值得关注的轮次）：**
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

**最后将 key_turns 分组为交互块（用户发言 → 响应序列）：**
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

### 提取用户介入标记（关键）

过滤出有实质内容的用户消息。跳过系统生成的消息：

**必须过滤掉：**
- 匹配 `<local-command-caveat>...</local-command-caveat>` 的消息
- 匹配 `<command-name>...</command-name>` 或 `<command-message>...</command-message>` 的消息
- 匹配 `<local-command-stdout>...</local-command-stdout>` 的消息
- `"Continue from where you left off."`（自动续接桩）
- `"Commands are in the form /command [args]"`（系统帮助）
- `/exit` 和 `Bye!`（会话退出）
- `Base directory for this skill:`（skill 加载桩）
- 纯空白或 ≤3 字符的消息

使用正则：过滤匹配 `<[^>]+>` XML 标签模式的消息以及上述字面量字符串。

**仅保留** `msg.strip()` 非空且通过所有过滤检查的消息。这些成为 L2 中显示的用户介入标记。

### 工具分类

| 分类 | 工具名称 |
|------|---------|
| **Skill** | `Skill` |
| **Agent** | `Agent` |
| **Read** | `Read`、`mcp__codegraph__codegraph_*`（只读 MCP 工具） |
| **Edit** | `Edit`、`Write`、`NotebookEdit` |
| **Bash** | `Bash` |
| **Task** | `TaskCreate`、`TaskUpdate`、`TaskList`、`TaskGet` |
| **Git** | 匹配 `git *` 模式的 `Bash` 调用 |
| **Other** | 其他所有 |

### 折叠规则（中粒度）

- 同一会话轮次中连续的 `Read`/`Grep`/`codegraph_*` 调用 → "探索了 N 个文件"
- 对同一文件的连续 `Edit` 调用 → "修改文件 X（N 次编辑）"
- `TaskCreate`/`TaskUpdate`/`TaskList` → 合并为 "任务管理（N 次操作）"
- 单独的 `Bash` 命令和 `git *` 命令 → 单独展示（重要事件）

### 检测卡点

对同一 assistant 轮次中的每对连续 tool call：
- **时间间隔**：调用 N 的结果和调用 N+1 的结果的 `attachment.timestamp` 差距 >30s → 标记为 `slow_gap`
- **重复调用**：同一工具名 + 同一输入连续出现 >3 次 → 标记为 `retry_loop`
- **慢 Agent**：`Agent` 工具调用，结果时间戳 - 调用时间戳 >2min → 标记为 `slow_agent`

### 追踪 skill 调用

对每个 `Skill` 工具调用，提取 `input.skill`（skill 名称）。记录：
- Skill 名称
- 开始时间（调用发起时间）
- 结束时间（结果 attachment 到达时间）
- 所在的 assistant 轮次

## 阶段 3 —— 生成 HTML 报告

### 输出路径

```bash
# 优先：项目的 docs/visualization 目录（如已存在）
if [ -d "docs/visualization" ]; then
    OUTPUT="docs/visualization/trace/trace-$(date +%Y%m%d-%H%M%S).html"
else
    OUTPUT="$HOME/.claude/skills/hh:trace/tmp/trace-$(date +%Y%m%d-%H%M%S).html"
fi
mkdir -p "$(dirname "$OUTPUT")"
```

### 信息架构（L0 → L3）

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
│   Agent 活动(N spawn, M min) | 卡点(N 个) | 用户介入     │
├──────────────────────────────────────────────────────┤
│ L2: 对话轮次火焰图 + 用户标记 (Canvas)                     │
│   - 左侧：用户对话标记列表（💬），点击跳转到对应轮次           │
│   - 右侧：火焰图，每行 = 一个对话轮次                         │
│   - 行宽 = 该轮 tool call 数量（按比例）                     │
│   - 颜色按轮次主要操作分类：                                   │
│     Skill调用=紫, Agent实施=蓝, 代码修改=绿,                  │
│     Git操作=橙, 探索/诊断=灰, 混合操作=青                    │
│   - 用户消息轮次用 ★ 标记                                    │
│   - 点击轮次行 → 跳转到 L3 对应交互块并高亮                    │
│   - 卡点高亮: 黄(间隔>30s) 橙(重复>3) 红(Agent>2min)        │
├──────────────────────────────────────────────────────┤
│ L3: 关键交互详情                                          │
│   - 按用户介入分块：用户说了什么 → Claude 做了什么              │
│   - 只展示有 Skill/Agent/Edit/Git 的轮次                    │
│   - 响应轮次用 end_ts 标注时间                               │
│   - 跨天显示日期分隔线                                       │
│   - Tool 分类分布（按类型统计数量）                           │
└──────────────────────────────────────────────────────┘
```

### L0 结论 —— 必须包含

1. **执行效能一句话**："本次会话 [N] 分钟，执行 [N] 个 tool call，调用 [N] 个 Skill，spawn [N] 个 Agent。整体执行 [顺畅/有 N 处可优化]。"
2. **用户意图达成度**: 检查用户最初消息和目标完成状态，判断 "✅ 全部完成" / "⚠️ 部分完成" / "❌ 未完成"
3. **关键路径耗时**: 列出主要阶段及其耗时（如：探索 5min / 修改 3min / 测试 2min）
4. **偏离/返工**: 列出需要重试或返工的操作
5. **LLM 反思建议**: 基于 trace 数据，由当前 LLM 生成 2-3 条具体建议。思考角度：
   - 如果用户一开始就指定 X，可以省掉 Y 步骤
   - 如果先做 A 再做 B，可以避免 C 次返工
   - 如果用 worktree 隔离修改，合并更安全
   - 如果有更明确的目标描述，可以减少探索级调用
   - 如果指定了 effort 级别，可以提高/降低审查深度

### L2 火焰图 —— 轮次级 Canvas 渲染（关键）

**必须按对话轮次级别渲染，禁止按 tool call 级别。** 100+ 个独立 tool call 行完全不可读。

嵌入 `<script>` 块，使用 Canvas API（零依赖）。

#### 步骤 A：构建 phase-to-DOM 映射（P2D）—— 必须包含

**P2D 将火焰图行索引映射到 L3 DOM 元素 ID。** 没有它，大多数 bar 的点击导航静默失效。这是实际测试中的 #1 bug。

在数据准备阶段（Python）生成映射，嵌入为 JS 常量：

```python
# 构建 interaction_blocks 后，创建 P2D 映射：
p2d = {}
for block in interaction_blocks:
    start = block['user']['turn_idx']
    # 结束 = 下一个 block 的用户轮次，或最后一个 phase 索引 + 1
    next_starts = [b['user']['turn_idx'] for b in interaction_blocks 
                   if b['user']['turn_idx'] > start]
    end = min(next_starts) if next_starts else max_turn_idx + 1
    for pi in range(start, end):
        p2d[pi] = start  # 将每个 phase 索引映射到其父 block 的 DOM id

# 嵌入为：const P2D = {json.dumps(p2d)};
```

#### 步骤 B：Canvas 点击处理器 —— 必须使用 P2D

```javascript
canvas.addEventListener('click', (e) => {
    const rowIdx = Math.floor((e.clientY - canvas.getBoundingClientRect().top) / (ROW_H + GAP));
    if (rowIdx >= 0 && rowIdx < activePhases.length) {
        const phaseIdx = activePhases[rowIdx].i;  // 轮次索引
        const domId = P2D[String(phaseIdx)];       // 通过映射查找
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

**禁止直接 `document.getElementById('block-' + phaseIdx)` —— 这就是 bug。** P2D 查找是强制性的。

#### 步骤 C：Canvas 渲染循环

```javascript
let y = 2, prevDate = '';
activePhases.forEach(p => {
    const w = Math.max((p.c / maxTools) * (W - 10), 4);
    // ★ 标记用户介入轮次
    if (userTurnIndices.has(p.i)) {
        ctx.fillStyle = '#f59e0b'; ctx.font = 'bold 10px sans-serif'; ctx.fillText('★', 1, y+11);
    }
    // 跨天虚线分隔
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

**轮次阶段分类**（决定 bar 颜色）：
- 有 `Skill` 调用且无 Agent/Edit → "Skill 调用"（紫色）
- 有 `Agent` spawn → "Agent 实施"（蓝色）
- 有 `Edit`/`Write` 且无 Agent → "代码修改"（绿色）
- 有 `Bash` 含 git 命令 → "Git 操作"（橙色）
- 仅 `Read`/`Bash` 探索 → "探索/诊断"（灰色）
- 混合或其他 → "混合操作"（青色）

**跨天处理（关键）**：
- 会话经常跨天（用户暂停后恢复）
- `end_ts` 在同一天内单调递增，但仅显示 HH:MM:SS 在日期边界会断裂（如 "06:25 → 03:49" 看起来像时间倒退）
- **必须检测日期变化**：比较连续轮次的 `end_ts.substring(0,10)`
- 火焰图中：在日期边界画虚线分隔 + 日期标签
- L3 响应中：当日期与父用户消息不同时显示 `MM-DD HH:MM` 格式；日期变化时在轮次间插入日期分隔徽标

**用户介入标记** —— 必须在 L2 中显著展示：
- 左列：可滚动的用户消息列表，带时间戳
- 每个标记可点击 → 滚动到 L3 对应轮次并高亮
- 过滤系统消息（skill 加载桩等）
- 显示用户消息预览（前 150 字符）

配色方案：
- Skill：`#7c3aed`（紫色）
- Agent：`#2563eb`（蓝色）
- Read：`#94a3b8`（灰蓝）
- Edit：`#059669`（翠绿）
- Bash：`#0891b2`（青色）
- Git：`#ea580c`（橙色）
- Task：`#d97706`（琥珀）
- Other：`#64748b`（石板灰）
- 卡点间隔：黄色条纹
- 卡点重试：橙色条纹
- 卡点 Agent：红色边框

### L3：关键交互（非全部轮次）

**必须只过滤到值得关注的轮次。** 展示全部 150+ 轮次是无效噪音。

**仅展示以下轮次：**
- 用户发送了实质性消息（用户介入）
- 调用了 `Skill`
- Spawn 了 `Agent`
- 编辑了代码（`Edit`/`Write`）
- 发生了 Git 操作（`git add/commit/push/merge/stash`）

**过滤掉**纯探索轮次（仅 `Read`/`Bash`/`Glob`，无编辑）。

**按用户介入块分组**：用户说了 X → Claude 响应了 Y（关键轮次序列）。每块显示用户消息头部，后跟缩进的响应轮次。

**响应轮次时间戳**：使用 `end_ts`（来自 `attachment` 记录的最后工具结果时间），而非 `user_ts`。对非用户轮次，`user_ts` 通常为空或过期。`end_ts` 是 Claude 完成响应时的准确墙上时钟时间。

**跨天处理**：当响应轮次的日期与其父用户消息不同时，显示 `MM-DD HH:MM` 格式。日期变化时在轮次间插入日期分隔 `<── 2026-06-10 ──>`。

### 设计规范（强制）

- **白底**：`body{background:#fff;color:#1e293b}`
- **卡片样式**：`background:#f8fafc;border-radius:12px;border:1px solid #e2e8f0`
- **用户标记**：`background:#fffbeb;border:1px solid #fde68a` —— 在白底上必须有可见背景
- **Tool chip**：使用 8 位 hex 透明度作为背景（`#7c3aed18`），纯色文字。格式：`font-size:10px;padding:2px 8px;border-radius:10px`
- **交互块**：左边框颜色编码 —— user=琥珀、skill=紫、agent=蓝、edit=翠绿、git=橙
- **Canvas 火焰图**：零外部依赖，纯 Canvas API
- **P2D 映射**：`phase_idx → user_turn_idx` 用于点击导航。两个索引不同 —— 必须构建映射表，禁止直接拼接

### L3：工具分类分布

展示各类工具的数量统计卡片。

## 阶段 4 —— 生成后操作

1. 将 HTML 文件写入输出路径
2. 用 `open <path>`（macOS）或 `xdg-open <path>`（Linux）打开
3. 向用户报告路径
4. 如果由 Stop hook 触发，同时向 stdout 打印摘要行

## Stop Hook 设置（可选）

会话结束时自动生成 trace，在 `~/.claude/settings.json` 中配置：

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "echo '[hh:trace] 会话已结束。运行 /hh:trace 查看执行追踪。'"
      }]
    }]
  }
}
```

注意：Stop hook 只打印提醒。实际的 trace 生成由用户手动调用 `/hh:trace` 时由 Claude 完成。

## 边界情况

- **空 transcript**（新会话）：报告 "尚无足够的执行记录"
- **超大 transcript**（>10k 行）：采样最近 10k 行，在报告中标注截断
- **无 Skill 调用**：L1 显示 "本次会话未调用任何 Skill"
- **无 Agent spawn**：L1 显示 "本次会话未 spawn Agent"
- **会话进行中**（手动触发，会话未结束）：分析到最新已写入行，标注报告为 "进行中"
- **缺失时间戳**：如果 `attachment.timestamp` 为 null，从行顺序估算
