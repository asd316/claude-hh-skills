---
name: hh:experiment
description: Prototype in isolation before integrating. Creates self-contained experimental scripts that don't touch working code. Use when user wants to try an approach, test a hypothesis, or compare options without risking the existing codebase.
---

# hh:experiment — Isolated Prototyping

**Rule: never modify working code to test an idea.** Build a standalone prototype first. Integrate only after confirmation.

## When to Use

- User says "先搞一个实验版本", "试一下", "能不能做一版纯代码的看看效果"
- User wants to test a hypothesis without risking the codebase
- User wants to compare multiple approaches (A vs B)
- User says "不要直接改现在代码"
- Before adding a new dependency or architectural change

## Workflow

### 1. Define the Experiment

Clarify with user:
- **What to test**: specific hypothesis or approach
- **Input data**: use EXISTING data, never regenerate (e.g. `data/messages/`, `data/store/`)
- **Success criteria**: how will we know it worked?

### 2. Create Isolated Script

- Place in `experimental/` directory (create if needed)
- Name: `experimental/{descriptive_name}.py`
- Self-contained: reads existing data → processes → outputs results
- No imports from `src/`, `phase2/`, or `phase3_faq/` unless absolutely necessary

### 3. Run and Observe

- Execute the script
- Present results clearly: what worked, what didn't, what's surprising
- If comparing approaches: show side-by-side comparison

### 4. Decide

Present options:
- **Integrate**: results look good → user confirms → integrate into main code
- **Iterate**: promising but needs adjustment → refine the experiment
- **Discard**: approach doesn't work → document why in experiment comments, move on

## Integration (when confirmed)

When user says "集成" or "merge":
1. Identify the right phase/module for the new code
2. Adapt experimental code to match existing patterns (naming, error handling, config)
3. Add to the correct directory (`src/`, `phase2/`, `phase3_faq/`)
4. Delete the experimental script (or move to `.save/` if worth keeping)
5. Run `hh:runbook` to document the new feature

## Multiple Approaches

When user wants A vs B comparison:
- Create separate scripts: `experimental/approach_a.py`, `experimental/approach_b.py`
- Run both on same input
- Present: approach, result, pros/cons for each
- Let user pick

## Red Flags

- Modifying existing code in `src/`, `phase2/`, `phase3_faq/` before confirming
- Running `python src/main.py` to generate test data (use existing data!)
- Creating complex experimental scripts with many dependencies — keep it minimal
- Integrating without explicit user confirmation
