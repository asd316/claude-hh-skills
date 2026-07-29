---
name: hh-visualize
description: Generate self-contained interactive HTML dashboards for data exploration. Supports tables, filters, toggle switches, charts. Use when user asks for "网页", "可视化", "dashboard", "报表", or any interactive data view.
---

# hh-visualize — Interactive HTML Reports

Generate single-file, self-contained HTML pages for data exploration. No server, no build step — open in browser directly.

## When to Use

- User asks for "一个网页来展示", "可视化报表", "dashboard"
- User wants interactive data exploration (filters, toggles, sorting)
- User says "能看看数据吗" and text isn't enough
- After running `hh-diagnose` — visualize the before/after comparison

## Output Location

All reports go to `docs/visualization/report-{topic}-{YYYYMMDD}.html`.
After generation, auto-open with `open <file>`.

## Cloud Deployment (Remote Access)

All reports should ALSO be deployed to Cloudflare Pages so the user can access them from any device (phone, remote computer).

### Deploy Decision

**Default: always deploy to cloud.** It costs virtually nothing (free plan: unlimited bandwidth, unlimited requests, 500 deploys/month, HTML files are only ~KB each).

No need to detect whether the user is on their Mac Mini or a remote device — just deploy every time.

### How to Deploy

After generating and saving the HTML file:

```bash
# 1. Copy the latest report to the deploy directory
cp docs/visualization/report-{topic}-{YYYYMMDD}.html /tmp/cloudflare-deploy/index.html

# 2. Deploy to Cloudflare Pages (production)
npx wrangler pages deploy /tmp/cloudflare-deploy --project-name my-html-reports --branch main
```

### Environment Variables (already configured in ~/.zshrc)

```
CLOUDFLARE_API_TOKEN  — API token with Pages read/write permission
CLOUDFLARE_ACCOUNT_ID — 3406f783194fbb3f35058ffe8dbc8a52
```

### Return Both Links

After deploy, tell the user:
- Local: `computer:///Users/admin/.../report-{topic}-{YYYYMMDD}.html`
- Cloud: `https://my-html-reports.pages.dev`

All reports go to `docs/visualization/report-{topic}-{YYYYMMDD}.html`.
After generation, auto-open with `open <file>`.

## Information Architecture (MANDATORY)

The #1 failure mode: **dumping all data onto the page with no hierarchy.** Users open the page, see a wall of numbers and tables, can't find the point, and give up.

Every report MUST follow this structure:

```
┌─────────────────────────────────────┐
│ L0: VERDICT (most prominent)        │  ← ONE sentence answer/conclusion
├─────────────────────────────────────┤
│ L1: KEY METRICS (≤6 cards)          │  ← Only the metrics that matter
├─────────────────────────────────────┤
│ L2: VISUAL OVERVIEW                 │  ← Chart, timeline, or comparison
├─────────────────────────────────────┤
│ L3: DETAILS (collapsed by default)  │  ← Full tables, raw data, per-item breakdown
└─────────────────────────────────────┘
```

### L0: Verdict (结论先行)

EVERY report MUST have a verdict. This is the first thing the user reads.

```html
<div class="verdict">
  <span class="verdict-label">结论</span>
  <p><!-- ONE sentence that answers "so what?" --></p>
</div>
```

Examples:
- ✅ "本周共采集 847 条消息，FAQ 新增 12 条，系统运行正常"
- ✅ "Phase 2 去重率 94%，3 个群的 store 文件已更新"
- ❌ "以下为本周数据汇总" (not a conclusion)
- ❌ "Report generated on 2026-06-09" (metadata, not a conclusion)

### L1: Key Metrics (关键指标 ≤6 cards)

Show ONLY the metrics that support the verdict. Maximum 6 stat cards — if you have more metrics, pick the top 6.

Each card: number + label + optional trend indicator (↑↓→).

```html
<div class="stats-grid">
  <div class="stat-card">
    <div class="value">847</div>
    <div class="label">本周消息</div>
  </div>
  <!-- max 6 cards total -->
</div>
```

### L2: Visual Overview

A chart, timeline, Mermaid diagram, or comparison table that shows the big picture. This is the "at a glance" view.

### L3: Details (collapsed)

ALL detailed data goes in `<details>` elements, collapsed by default:

```html
<details>
  <summary>按群查看详情 (3 个群)</summary>
  <!-- table or chart here -->
</details>

<details>
  <summary>完整消息列表 (847 条)</summary>
  <!-- full data table here -->
</details>
```

If the user wants to drill in, they click. If not, they see only L0-L2 and are done.

## Anti-Patterns (DO NOT DO)

| ❌ Don't | ✅ Do |
|----------|------|
| Wall of stat cards (10+ metrics) | ≤6 cards, pick the important ones |
| Full data table above the fold | Collapsed in `<details>` |
| No verdict, just a title | ONE sentence conclusion at top |
| "以下为数据汇总" as verdict | State what the data MEANS |
| Show all groups/dimensions equally | Highlight anomalies, show Top N only |
| Grid of 20 detail cards | Collapse to expandable sections |
| Metadata as the first heading | Metadata at the bottom, verdict at top |

## Design Principles

1. **Verdict first** — conclusion before evidence, always
2. **Hierarchy** — L0→L1→L2→L3, each layer adds detail
3. **Single file** — all CSS/JS inlined, no dependencies
4. **Interactive where it adds value** — filters, toggles, sorting for L3 details
5. **Minimalist** — clean layout, no heavy frameworks
6. **Self-documenting** — title, data source, generation date at bottom
7. **Anomaly-first** — highlight what's unusual, not what's normal

## Common Report Types

### Type A: Data Table with Filters
When user wants to explore a dataset.
```
L0: Verdict about what the data shows
L1: Row count, unique categories, date range (3-4 cards)
L2: Summary chart or distribution
L3: Full table with search, sort, filter (collapsed)
Template: dark header, striped rows, sticky filters
```

### Type B: Comparison Dashboard
When comparing A vs B (before/after, two approaches, two time periods).
```
L0: Verdict about which is better / what changed
L1: Key diff metrics (3-4 cards with delta)
L2: Side-by-side summary
L3: Full comparison table (collapsed)
```

### Type C: Interactive Configuration Viewer
When user wants to toggle scenarios (e.g. "what if we exclude X from buffer?").
```
L0: Verdict about the impact of toggling
L1: Affected count, impact %, savings (3-4 cards)
L2: Toggle controls (prominent, above the fold)
L3: Full affected rows list (collapsed)
```

### Type D: Timeline / Flow
When showing processes or time-based data.
```
L0: Verdict about the timeline (duration, status)
L1: Start/end, duration, step count, status summary
L2: Mermaid timeline or Gantt-style overview
L3: Per-step details (collapsed)
```

## Workflow

### 1. Understand the Data
- Read the data source(s) to understand structure and volume
- Identify: what's the ONE thing the user needs to know?
- That ONE thing = the verdict
- Decide report type (A/B/C/D)

### 2. Select What Matters
- What metrics directly support the verdict? → L1 cards (≤6)
- What visual tells the story best? → L2 overview
- What raw data might the user want to check? → L3 details (collapsed)

### 3. Build the HTML
- Start from L0 verdict → L1 cards → L2 visual → L3 details
- Populate with real data (embed as JSON in `<script>` tag)
- Add interactivity to L3: search, column sort, filters for tables
- Keep JS simple — vanilla, no frameworks

### 4. Deliver
- Save to `docs/visualization/report-{topic}-{YYYYMMDD}.html`
- Auto-open in browser
- Tell user: "Report saved to [path]. Verdict: [one-liner]. What adjustments needed?"

### 5. Iterate
- User provides feedback → update the same file (or create new version if major change)
- When confirmed, file stays in `docs/visualization/` for future reference

## Technical Constraints

- **No external CDN** — everything inline (the user may be on intranet)
- **Vanilla JS only** — no React, Vue, jQuery
- **Data under 10MB** — if dataset larger, sample or paginate
- **Mermaid.js can be loaded from CDN** for diagrams (it's standard and widely cached)
- **Responsive** — should work on laptop screens (1200px+)
