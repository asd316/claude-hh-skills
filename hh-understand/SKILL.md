---
name: hh-understand
description: Project architecture knowledge graph — understand how intent flows through implementation. Query the graph to trace goals→pipelines→data flow, or init/update the graph from project docs and git history. Use when user asks "这个项目怎么实现的", "数据流怎么走的", "为什么这么设计", or wants to understand project structure without reading all code.
---

# hh-understand — Project Architecture Understanding

> **IDE Environment Note:** Throughout this skill, `CLAUDE.md` refers to the project root instruction file. If running in **Claude Code**, use `CLAUDE.md`. If running in **Cursor / Windsurf / TRAE / other IDEs**, use `AGENTS.md` instead. Detection: if `CLAUDE.md` exists in project root, use it; otherwise fall back to `AGENTS.md`.

Maintain and query a high-level **knowledge graph** of project architecture. This is NOT code-level detail — it captures the skeleton: what the project aims to do, how intent flows through pipelines and stages, what key decisions and constraints shape it.

Graph lives at `docs/graph/project-graph.json`. Query it to understand architecture without reading every file. Update it to keep the skeleton current as the project evolves.

## When to Use

- `/hh-understand` (no args) — init the graph (first time) or update it (subsequent)
- `/hh-understand "how does X work?"` — query the graph to understand a specific part
- User asks: "这个项目怎么实现的", "数据是怎么流的", "为什么这么设计", "哪里可能有问题"

## Graph Location

| File | Purpose |
|------|---------|
| `docs/graph/project-graph.json` | The graph — nodes and edges |
| `docs/graph/.graph-state.json` | Internal state (last commit, last update) |
| `docs/graph/README.md` | Human-readable explanation of this directory |

## Graph Schema

### Node Types (7 types — skeleton only, no implementation details)

| Type | Meaning | When to create | Example |
|------|---------|---------------|---------|
| **Goal** | User/business objective — WHY the project exists | Top-level purpose, 1-3 per project | "24/7 monitor group chat messages" |
| **Pipeline** | End-to-end processing chain — the big picture flow | Each distinct end-to-end flow | "Screenshot→VLM→Merge→Dedup→Store→FAQ" |
| **Stage** | One phase/module within a pipeline — a distinct job | Each major processing unit | "Phase 2: Dedup & Storage" |
| **Transform** | Data changing form between stages | Each significant format/representation change | "screenshot.png → messages.json" |
| **Decision** | Key design choice + WHY it was made | Non-obvious tradeoffs with rationale | "Screenshots over API: 4 approaches all failed" |
| **Constraint** | Hard rule that if violated breaks the project | ONLY the most critical ones (≤7 total) | "VLM=image→text only, LLM=text→text, pure code first" |
| **Entry** | How the system is triggered/started | Each distinct way to invoke the system | "python src/main.py", "launchd timer" |

### Edge Relations

| Relation | From → To | Meaning |
|----------|-----------|---------|
| `drives` | Goal → Pipeline | This goal motivates this pipeline |
| `contains` | Pipeline → Stage | Pipeline is composed of these stages |
| `produces` | Stage → Transform | Stage outputs data in this form |
| `consumes` | Stage → Transform | Stage reads data in this form |
| `constrains` | Constraint/Decision → Stage/Transform | This rule limits how this works |
| `depends_on` | Stage → Stage | This stage cannot run before that one |
| `triggers` | Entry → Stage/Pipeline | This entry point starts this |

---

## Mode Detection

Run IMMEDIATELY upon invocation — do NOT ask the user which mode.

```
Is project-graph.json present?
├─ NO  → Mode A (Initial Generation)
├─ YES, no question arg → Mode B (Incremental Update)
└─ YES, question provided → Query Mode
```

---

## Mode A: Initial Graph Generation

**When:** First time running in a project. No `docs/graph/project-graph.json` exists.

**Goal:** Scan project documentation and structure → infer the skeleton graph → present for human review via HTML → save after approval.

### Step A1: Gather Information

Read these in parallel (don't open files that don't exist):

1. **CLAUDE.md** — Primary source. Look for: project purpose, architecture section, phase/stage descriptions, key conventions, "rules from repeated mistakes"
2. **docs/Context.md** — Domain knowledge, terminology (if exists)
3. **docs/Constraints.md** — Hard constraints (if exists)
4. **docs/adr/INDEX.md** — Architectural decisions (if exists)
5. **README.md** — Project overview (if exists)
6. **Git log** — `git log --oneline -50` for project evolution context
7. **Top-level directory listing** — Understand module structure

### Step A2: Infer Nodes

Extract from gathered information — focus on INTENT and FLOW, not code:

**Goals (1-3):**
- What problem does this project solve? Find the purpose statement.
- What user need drives the project's existence?

**Pipelines (1-3):**
- What is the end-to-end flow? Look for phase descriptions, data flow descriptions.
- Each distinct processing chain is one Pipeline.
- Label should be a short chain: "A → B → C → D"

**Stages (per Pipeline):**
- What are the major processing units? Each phase or module with a distinct job.
- Include: what it does (one sentence), what it reads, what it writes.
- Do NOT include sub-steps within a stage — keep at the phase/module level.

**Transforms (between Stages):**
- Where does data change form? Each significant format change.
- Label as "input.format → output.format" with a brief description.

**Decisions:**
- What non-obvious tradeoffs were made? Why was approach X chosen over Y?
- Each Decision must have a "why" field.

**Constraints (≤7 total):**
- Extract ONLY the most critical constraints — rules that if violated will break the project.
- Look for markers: "NEVER", "必须", "不可行", "MUST NOT", rules in CLAUDE.md §5.x.
- Skip conventions and style preferences — only architectural constraints.

**Entries:**
- How is the system invoked? CLI commands, scheduled jobs, API endpoints.

### Step A3: Generate Review HTML

Create `docs/visualization/understand-init-{YYYYMMDD}.html`.

Follow **HTML Output Rules** (see bottom of this skill). The review page must show:

1. **Verdict:** "从 {source} 推断出 {N} 个目标、{M} 条流水线、{K} 个阶段"
2. **Mermaid graph:** All nodes and edges in one diagram. Use `graph TD` layout.
3. **Node inventory:** Each node with: type badge, label, summary, why (for decisions/constraints), confidence badge
4. **Confidence badges on every node:**
   - 🟢 **High** — explicitly stated in project docs
   - 🟡 **Medium** — inferred from structure/context
   - 🔴 **Low** — guessed, needs user confirmation

### Step A4: Review Loop

1. Auto-open the HTML
2. Tell user: "图谱草案已生成。请审核 — 哪些节点不对、缺了、多了？"
3. User provides feedback → update the draft → regenerate HTML (same file, overwrite) → repeat
4. When user approves → proceed to Step A5

### Step A5: Save Graph

Create `docs/graph/` directory. Write:

**`project-graph.json`:**
```json
{
  "project": "<project name>",
  "generated": "<ISO timestamp>",
  "last_commit": "<HEAD commit hash>",
  "nodes": [ ... ],
  "edges": [ ... ]
}
```

**`.graph-state.json`:**
```json
{
  "last_commit": "<HEAD commit hash>",
  "last_update": "<ISO timestamp>",
  "version": 1
}
```

**`README.md`:**
```markdown
# Project Knowledge Graph

This directory contains a high-level knowledge graph of the project's architecture.
It is maintained by `/hh-understand`.

- `project-graph.json` — The graph (nodes + edges)
- `.graph-state.json` — Internal state for incremental updates
- `README.md` — This file

To update: run `/hh-understand` (no args).
To query: run `/hh-understand "your question"`.
```

After saving: auto-run **Query Mode** with question "项目整体架构" to generate a reference HTML (`understand-overview-{YYYYMMDD}.html`).

---

## Mode B: Incremental Update

**When:** Graph exists. No question provided. User wants to bring the graph up to date.

**Goal:** Find what changed since last update → understand the impact → update the graph → report.

### Step B1: Gather Signals (得知信息)

Read in parallel:

1. **`.graph-state.json`** — get `last_commit`, `last_update`
2. **`git log {last_commit}..HEAD --oneline`** — new commits
3. **`git diff {last_commit}..HEAD --stat`** — changed files overview
4. **Current session's conversation** — what was discussed, what decisions emerged, what changed
5. **`docs/todo/INDEX.md`** — recently completed tasks (optional, may be stale; use as supplementary signal only)

### Step B2: Explore Truth (探索真相)

For each significant signal (new modules, deleted files, config changes, major refactors):

1. **What changed?** — Read changed files. Identify the nature of the change.
2. **Why?** — From commit messages, conversation context, or file content — what problem did this solve?
3. **Graph impact?** — New node needed? Existing node modified? Obsolete node to remove?
4. **Is this skeleton-level?** — Filter out implementation details. Only capture architectural changes.

### Step B3: Verify and Consolidate (验证收尾)

- Cross-check: new findings don't contradict existing graph?
- Merge overlapping signals: multiple commits for same feature → one graph change
- Mark uncertain findings for user confirmation
- Assign confidence to each proposed change

### Step B4: Report and Apply (总结报告填写)

1. **Update `project-graph.json`** — add/modify/remove nodes and edges
2. **Update `.graph-state.json`** — set `last_commit` to HEAD, bump `version`
3. **Generate HTML report** — `docs/visualization/understand-update-{YYYYMMDD}.html`:
   - Summary: X nodes added, Y modified, Z removed
   - For each change: what, why, confidence badge
   - Updated Mermaid graph (full overview)
4. Auto-open in browser

If no significant changes found: respond with text — "自上次更新以来，项目骨架无重大变化。（{N} 个提交均为实现细节调整）"

---

## Query Mode

**When:** Graph exists. Question provided: `/hh-understand "question"`

**Goal:** Answer the question by tracing the graph, supplemented by code only when necessary.

### Step Q1: Search Graph

Load `project-graph.json`. Find relevant nodes by matching the question against:
- Node labels
- Node summaries
- Node `why` fields (for decisions/constraints)

### Step Q2: Trace Relationships

From matched nodes, follow edges to build the relevant subgraph:
- **"How does X work?" / "X 怎么实现"** → follow `contains` and `produces` edges downstream
- **"Where does X come from?" / "X 数据从哪来"** → follow `consumes` and `depends_on` edges upstream
- **"Why is X this way?"** → find `constrains` edges pointing at the relevant stage
- **"What triggers X?"** → follow `triggers` edges
- **"What would break if X changes?"** → follow `depends_on` edges downstream + `constrains` edges

### Step Q3: Supplement with Code (if needed)

If the graph lacks enough detail to answer confidently:
- Read the specific source files mentioned in the relevant Stage/Transform nodes
- Do NOT read entire files — target the specific section
- Prioritize graph information over code

### Step Q4: Decide Output Format

Judge complexity:
- **Text answer** — Single node/edge query, answerable in ≤3 sentences. Answer directly in conversation.
- **HTML answer** — Multi-node chain, structural question, or involves relationships. Generate HTML.

HTML: `docs/visualization/understand-{topic-slug}-{YYYYMMDD}.html`. Follow **HTML Output Rules** below. Auto-open.

---

## HTML Output Rules

ALL HTML outputs (init, update, query) MUST follow these rules. These rules exist to prevent information overload — the #1 complaint: "too much data, can't find the point."

### Rule 1: Verdict First (结论先行)

The VERY FIRST thing the user sees:

```html
<div class="verdict">
  <span class="verdict-icon">💡</span>
  <span class="verdict-text"><!-- ONE sentence, ≤40 Chinese characters --></span>
</div>
```

This is the answer. Everything below is evidence.

### Rule 2: Confidence + Basis (置信度)

Immediately after verdict, before any diagram or detail:

```html
<div class="confidence">
  <span class="badge high">🟢 高置信度</span>
  <span class="basis">基于 CLAUDE.md 明确声明</span>
</div>
```

Badge levels: 🟢 High (explicit in docs/code) / 🟡 Medium (inferred from structure) / 🔴 Low (guess, needs confirmation).

Basis must state WHERE the information came from, not just "分析得出".

### Rule 3: Three-Layer Information (三层信息)

```
┌──────────────────────────────┐
│ L1: VERDICT (always visible) │  ← One sentence answer
├──────────────────────────────┤
│ L2: SKELETON (always visible)│  ← Mermaid diagram showing relationships
├──────────────────────────────┤
│ L3: DETAILS (collapsed)      │  ← Expandable sections with specifics
└──────────────────────────────┘
```

L3 sections use `<details>` elements, ALL collapsed by default:
```html
<details>
  <summary>节点详情：Phase 2 — 去重与存储</summary>
  <p>...</p>
</details>
```

### Rule 4: Mermaid for Skeleton (Mermaid 做主视觉)

The L2 skeleton MUST use Mermaid.js. Choose diagram type by content:
- **flowchart TD** — for pipelines, data flow, stage sequences
- **graph LR** — for horizontal dependency maps
- **graph TD** — for hierarchical overview

CDN is allowed for Mermaid:
```html
<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>
```

### Rule 5: Only Show What's Relevant (只展示相关的)

- Query mode: show ONLY the nodes/edges relevant to the question, not the full graph.
- Init/Update mode: show the full graph (it's a review/diff).
- Do NOT show unrelated stages, unrelated constraints, or unrelated data flows.

### Rule 6: Clean Minimalist Design (简洁设计)

- Dark theme as default (match project convention from hh-visualize)
- No unnecessary stat cards or decorative elements
- Max width: 1200px for readability
- Mermaid diagram should be the visual focus
- No tables unless comparing ≥3 items

### Rule 7: File Naming

| Mode | Filename |
|------|----------|
| Init | `understand-init-{YYYYMMDD}.html` |
| Update | `understand-update-{YYYYMMDD}.html` |
| Query | `understand-{topic-slug}-{YYYYMMDD}.html` |
| Auto overview | `understand-overview-{YYYYMMDD}.html` |

Topic slug: lowercase, hyphenated, ≤30 chars. From the question's key term.

### Rule 8: Auto-Open

After generation, run `open <file>` to launch in browser. Then tell user the path.

---

## Graph Maintenance Principles

1. **Skeleton only:** If you're describing a function's implementation, you've gone too deep. Delete it.
2. **Intent over mechanics:** A Stage node should say WHAT it does and WHY, not HOW. The code shows HOW.
3. **Stable over volatile:** The graph should change infrequently. Most commits should NOT trigger graph updates.
4. **Constraints are precious:** Only add a Constraint node when violating it has caused ACTUAL problems. Default to ≤7.
5. **Decisions carry rationale:** Every Decision node MUST have a `why` field. "We chose X over Y" without rationale is useless.
6. **Graph + onboard = complete picture:** Onboard docs (CLAUDE.md, ADR, error-log) cover details and context. Graph covers structure and relationships. They complement — don't duplicate.

## Red Flags

- Adding nodes for individual files or functions (too granular)
- Creating a Constraint for every CLAUDE.md rule (only the truly critical ones)
- Generating HTML without a verdict sentence at top
- Showing the full graph for a specific query (noise)
- Describing HOW code works instead of WHAT stage does
- Creating duplicate nodes for concepts already covered by onboard docs
- Saving the graph without user review in Mode A
