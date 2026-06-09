---
name: hh:visualize
description: Generate self-contained interactive HTML dashboards for data exploration. Supports tables, filters, toggle switches, charts. Use when user asks for "网页", "可视化", "dashboard", "报表", or any interactive data view.
---

# hh:visualize — Interactive HTML Reports

Generate single-file, self-contained HTML pages for data exploration. No server, no build step — open in browser directly.

## When to Use

- User asks for "一个网页来展示", "可视化报表", "dashboard"
- User wants interactive data exploration (filters, toggles, sorting)
- User says "能看看数据吗" and text isn't enough
- After running `hh:diagnose` — visualize the before/after comparison

## Output Location

All reports go to `docs/visualization/report-{topic}-{YYYYMMDD}.html`.
After generation, auto-open with `open <file>`.

## Design Principles

1. **Single file** — all CSS/JS inlined, no dependencies
2. **Interactive by default** — if data has categories, add filter toggles. If data has time, add date range.
3. **Minimalist** — clean layout, no heavy frameworks
4. **Readable** — tables with proper headers, charts where they add value
5. **Self-documenting** — title, data source, generation date at top

## Common Report Types

### Type A: Data Table with Filters
When user wants to explore a dataset.
```
Features: search, column sort, category toggle checkboxes, row count
Template: dark header, striped rows, sticky filters
```

### Type B: Comparison Dashboard
When comparing A vs B (before/after, two approaches, two time periods).
```
Features: side-by-side tables, diff highlighting, summary stats at top
```

### Type C: Interactive Configuration Viewer
When user wants to toggle scenarios (e.g. "what if we exclude X from buffer?").
```
Features: checkbox/toggle controls at top, live-updating summary numbers, affected rows highlighted
```

### Type D: Timeline / Flow
When showing processes or time-based data.
```
Features: chronological list, expandable detail sections, status indicators
```

## Workflow

### 1. Understand the Data
- Read the data source(s) to understand structure and volume
- Ask user: what questions should this report answer?
- Decide report type (A/B/C/D)

### 2. Build the HTML
- Start from the appropriate template pattern
- Populate with real data (embed as JSON in `<script>` tag)
- Add interactivity: at minimum, search + column sort for tables
- Keep JS simple — vanilla, no frameworks

### 3. Deliver
- Save to `docs/visualization/report-{topic}-{YYYYMMDD}.html`
- Auto-open in browser
- Tell user: "Report saved to [path]. What adjustments needed?"

### 4. Iterate
- User provides feedback → update the same file (or create new version if major change)
- When confirmed, file stays in `docs/visualization/` for future reference

## Technical Constraints

- **No external CDN** — everything inline (the user may be on intranet)
- **Vanilla JS only** — no React, Vue, jQuery
- **Data under 10MB** — if dataset larger, sample or paginate
- **Mermaid.js can be loaded from CDN** for diagrams (it's standard and widely cached)
- **Responsive** — should work on laptop screens (1200px+)
