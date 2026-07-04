---
name: hh-diagnose
description: Structured diagnostic pipeline for data quality issues. Understand context → audit output → trace root cause (code/prompt/config) → propose fix → apply fix → verify → report. Use when data/output looks wrong, when something that should work doesn't, or when user asks "why is this broken".
---

# hh-diagnose — Structured Diagnostic Pipeline

> **IDE Environment Note:** Throughout this skill, `CLAUDE.md` refers to the project root instruction file. If running in **Claude Code**, use `CLAUDE.md`. If running in **Cursor / Windsurf / TRAE / other IDEs**, use `AGENTS.md` instead. Detection: if `CLAUDE.md` exists in project root, use it; otherwise fall back to `AGENTS.md`.

Six-phase investigation workflow. Never skip phases. Never fix before understanding.

## The Iron Rule

**NO FIXES BEFORE ROOT CAUSE IS IDENTIFIED AND CONFIRMED.**

The user's most frequent frustration: AI jumping to code changes when the user asked "why is this happening?" Explain the mechanism FIRST.

## When to Use

- User says "这个逻辑有问题", "怎么解决", "为什么没有生效"
- Data output looks wrong or incomplete
- Something that should work doesn't
- User pastes error output and asks for diagnosis
- User says "检查一下" or "review the results"

## The 6 Phases

### Phase 1: Understand Context
- Read relevant docs (`CLAUDE.md` — especially documented constraints/known issues, `docs/error-log.md` if exists, `docs/context/error-patterns.md` if exists)
- **Check project's error log FIRST** (if exists) — does a known error pattern match these symptoms? If yes, skip to applying the documented fix (still explain the mechanism).
- Understand what the data SHOULD look like
- Identify which phase/module produces this data
- **Gate:** Can you explain the expected behavior in 2 sentences? If not, re-read.

### Phase 2: Audit Output
- Read the actual output files (JSON, logs, etc.)
- Compare expected vs. actual — where EXACTLY is the divergence?
- Categorize issues: missing data? wrong data? duplicate data? format error?
- **Gate:** Can you list specific, concrete problems with file paths and line numbers? If not, dig deeper.

### Phase 3: Trace Root Cause
- For each issue: is it a code bug, a prompt problem, a config gap, or an input data issue?
- Read the relevant source code. Don't guess.
- Check concurrency/race conditions — a common source of silent data loss
- **Cross-module/phase format change**: did a format change in one module break a downstream consumer? If the project has multiple phases/modules with shared data formats, this is a common cause of multi-day outages.
- **Project's documented rules**: cross-reference the symptom against any documented constraints or "rules from repeated mistakes" in CLAUDE.md — violations of these are the most common root causes
- **Gate:** Can you explain the FULL causal chain from root cause → symptom? Test your theory against the data.

### Phase 4: Propose Fix
- Present findings: "Problem: [symptom]. Root cause: [mechanism]. Fix: [approach]."
- For each issue, propose the simplest fix that addresses the root cause.
- If multiple fixes possible, present options with tradeoffs.
- **Gate:** User confirms approach before any code changes.

### Phase 5: Apply Fix
- Implement the confirmed fix.
- If fix affects multiple phases, check for downstream impacts.
- **Gate:** Fix is applied. Ready to verify.

### Phase 6: Verify & Report
- Run the fixed code on the SAME input data.
- Compare before/after output.
- Report: "Root cause was [X]. Fix was [Y]. Verified by [Z]. Before: [symptom]. After: [resolved]."
- **MANDATORY if this is a new error pattern**: update the project's error log (e.g. `docs/error-log.md`) with the error + root cause + fix. This is how the project learns.
- If the error matches an existing pattern but adds new detail, append to the existing entry.

## Quick Mode (for simple issues)

If the issue is clearly a single-file, single-cause problem, compress to:
1. **Explain** the root cause (always first!)
2. **Propose** the fix
3. **Apply** after confirmation
4. **Verify**

But NEVER skip the "explain first" step. The user wants to understand.

## Red Flags

- Jumping to Phase 5 (apply fix) before Phase 3 (trace root cause)
- Not checking the project's error log (if exists) for known patterns with matching symptoms
- Proposing fixes without reading the relevant source code
- Guessing the root cause instead of tracing through code
- Not comparing before/after when verifying
- Ignoring concurrency as a potential cause in multi-threaded code
- Fixing a format/API change in one module without checking all downstream consumers
