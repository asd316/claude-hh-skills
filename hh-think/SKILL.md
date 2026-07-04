---
name: hh-think
description: Turn vague ideas, doubts, or "something feels off" into structured, visual exploration. Collaborative thinking mode — NOT command execution. Outputs text advice for simple questions, interactive HTML for multi-element relationships. Use when user says "I think...", "Should we...", "Something feels off", "帮我看看这个方案", or any exploratory question.
---

# hh-think — Idea Validation & Design Exploration

> **IDE Environment Note:** Throughout this skill, `CLAUDE.md` refers to the project root instruction file. If running in **Claude Code**, use `CLAUDE.md`. If running in **Cursor / Windsurf / TRAE / other IDEs**, use `AGENTS.md` instead. Detection: if `CLAUDE.md` exists in project root, use it; otherwise fall back to `AGENTS.md`.

Turn vague ideas into reviewable, visual artifacts — or concise text advice when the problem is simple.

## Core Mindset

| Dimension | What it means |
|-----------|---------------|
| **Intent clarification** | What is the user really asking? Not just the surface question. |
| **Empathetic understanding** | Stand in the user's shoes. What pain point drives this? |
| **Independent thinking** | Don't just ask questions — analyze project context, think through the problem, proactively propose solutions. |
| **Visual output** | Produce something the user can review and validate against their core needs. |

**Not** a passive Q&A session. **Not** a command execution mode. This is collaborative exploration.

## Output Decision

```
User's idea/doubt
    │
    ├─ Involves relationships, structure, or paths between multiple elements?
    │   ├─ YES → Generate HTML with diagrams → save to docs/visualization/idea-{topic}-{YYYYMMDD}.html → auto-open browser
    │   └─ NO  → Text advice directly in conversation
```

| Question | Output | Why |
|----------|--------|-----|
| "Should we use JSON or YAML?" | Text | Single-dimension choice |
| "Should we add caching?" | Text | Feasibility judgment (unless evaluating full impact chain) |
| "How to split modules and their dependencies?" | HTML | Structure + relationships |
| "A vs B: impact, dependencies, risk" | HTML | Multi-dimensional relationships |
| "Is our data pipeline architecture correct?" | HTML | Flow + structure review |
| "Something feels off about error handling" | HTML | Decision paths + failure categories |

## Workflow

### 1. Read Intent
What surface question did the user ask? What's beneath it? Why are they asking this now?

### 2. Read Project Context
- Read `CLAUDE.md` for architecture and constraints
- Read relevant code/config/docs for the topic area
- Check `docs/adr/` for related past decisions

### 3. Evaluate
Judge the idea against three lenses simultaneously:
- **Project constraints**: Does existing architecture support this? What code would be affected?
- **Engineering principles**: Simplicity, maintainability, performance, correctness.
- **Underlying user need**: Is the idea solving the real problem, or just a symptom?

### 4. Decide Output Format
Use the decision flowchart above. Err toward HTML when any multi-element structure is involved.

### 5. Produce Output

**Text path:** Concise advice directly in conversation.
- State your understanding first (verify alignment)
- Present recommendations with rationale
- No boilerplate

**HTML path:** Generate self-contained HTML to `docs/visualization/idea-{topic}-{YYYYMMDD}.html`.
- Use [HTML_TEMPLATE.md](HTML_TEMPLATE.md) for structure
- Auto-open with `open <file>` command
- Name examples: `idea-module-split-20260603.html`, `idea-caching-strategy-20260603.html`

## HTML Generation Rules

- **Minimal text, maximal diagram** — Every mermaid diagram should replace a paragraph of explanation.
- **Diagrams chosen by problem type**: flowchart (process), sequence (interaction), graph (dependency), decision tree (choices).
- **Interactive only when complexity warrants** — tabs for comparing options, collapsible nodes for details.
- **Always new file** — don't overwrite previous ones.
- **Auto-open browser** after generation.

## Red Flags — You're Doing It Wrong

- Asking 5+ clarifying questions before reading any code
- Producing 3+ paragraphs of text in an HTML section
- Giving a generic answer that doesn't reference actual project files
- Creating HTML for a single-dimension choice
- Starting to implement code when the user asked for a review
- Agreeing with every idea without challenging

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Asking 10 clarifying questions instead of thinking independently | Analyze code first, propose solution first, then ask only what you genuinely can't determine |
| Producing a wall of text in the HTML | Replace text blocks with diagrams. Keep prose to 1-2 lines per section. |
| Treating this as command execution | Slow down. Explore. The user doesn't have a clear answer yet. |
| Skipping project context analysis | Always read relevant code before evaluating. |
| Making HTML for a simple yes/no question | Use the output decision flowchart. Simple questions get text. |
| Jumping to implement instead of reviewing | This skill produces a review artifact, not code. User validates first. |
