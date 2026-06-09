---
name: hh:sync-skills
description: Sync hh-series skills to GitHub and inspect for issues. Auto-detects intent — inspect first, then sync. Use when user says "sync skills", "上传skill", "同步到github", "检查skill", or wants to publish/push hh skills.
---

# hh:sync-skills — GitHub Sync & Inspection

Pushes all hh-series skills to `asd316/claude-hh-skills` on GitHub, and runs quality inspections to catch frontmatter issues, broken references, and structural problems.

## Repository

| Item | Value |
|------|-------|
| GitHub | `https://github.com/asd316/claude-hh-skills` |
| Local clone | `~/.claude/hh-skills-repo/` |
| Source skills | `~/.claude/skills/hh-*` and `~/.claude/skills/hh:*` |
| Git identity | `asd316` / `992358904@qq.com` |
| SSH key | `~/.ssh/id_ed25519_personal` |

## Mode Detection

Read the user's message and pick ONE mode:

| Keywords (CN/EN) | Mode |
|------------------|------|
| 初始化, setup, 首次, 第一次, clone, 关联 | 🆕 Setup |
| 上传, 同步, sync, push, 更新, 发布 | 🔵 Sync |
| 检查, 巡检, inspect, check, 问题, 诊断, audit | 🟢 Inspect |

**Default (no clear keyword):** Run Inspect first, then offer to Sync if there are changes to push.

**If the local repo `~/.claude/hh-skills-repo/` does not exist**, redirect to Setup first regardless of keywords.

---

## 🟢 Mode: Inspect (Patrol)

Scan all hh skills and produce a structured report. **Never modify files during inspection.**

### Step 1: Collect inventory

List all skill directories:
```bash
ls -d ~/.claude/skills/hh-* ~/.claude/skills/hh:* 2>/dev/null
```

For each skill directory, check:
- Does `SKILL.md` exist?
- List all files in the directory (excluding `.DS_Store`)

### Step 2: Run checks

For EACH skill, perform these checks:

#### C1: Frontmatter validity
Read the first 10 lines of `SKILL.md`. Verify:
- `---` opens and closes the frontmatter block
- `name:` field exists and is non-empty
- `description:` field exists and is non-empty
- No duplicate or malformed YAML keys

#### C2: Name consistency
The frontmatter `name:` should match the directory name:
- `hh-sync` directory → `name: hh:sync-skills` (directory uses `-`, name uses `:`)
- `hh:deploy` directory → `name: hh:deploy` (both use `:`)
- **Rule:** Replace `-` with `:` in directory name, or keep `:` as-is → should equal frontmatter `name`

#### C3: Local file references
Scan SKILL.md for markdown links referencing LOCAL files (not http/https):
- Pattern: `[text](relative/path.md)` or `[text](filename.md)`
- For each local reference, verify the target file exists relative to the skill directory
- Flag any broken links

#### C4: Cross-skill references
If a skill references another hh skill by name (e.g., "Run `hh:diagnose`" or "after `hh:runbook`"), verify that skill directory exists.

#### C5: Orphan files
Any file in the skill directory NOT named `SKILL.md` and NOT referenced by SKILL.md is flagged as a potential orphan (WARNING level, not ERROR — it might be intentionally standalone, like `HTML_TEMPLATE.md` which is referenced in documentation text rather than a markdown link).

### Step 3: Produce report

Output format:

```
## 🔍 hh Skills Inspection Report — YYYY-MM-DD

### Results by Skill

| Status | Skill | Files | Issues |
|--------|-------|-------|--------|
| ✅ | hh-diagnose | 1 | — |
| ✅ | hh-experiment | 1 | — |
| ⚠️ | hh-think | 2 | HTML_TEMPLATE.md not linked in SKILL.md (intentional template) |
| ✅ | hh-onboard | 1 | — |
| ✅ | hh-runbook | 1 | — |
| ✅ | hh-schedule | 1 | — |
| ✅ | hh-undo | 1 | — |
| ✅ | hh-visualize | 1 | — |
| ✅ | hh:deploy | 8 | — |
| ✅ | hh-sync | 1 | — |

### Issue Details

**Warnings (2):**
- `hh-think`: `HTML_TEMPLATE.md` exists but has no explicit markdown link from SKILL.md → likely intentional (referenced in prose as `[HTML_TEMPLATE.md](HTML_TEMPLATE.md)`)
- `hh-onboard`: references `hh:think hh:diagnose hh:experiment hh:visualize hh:schedule hh:undo hh:runbook` → verify these all exist ✅ (all present)

**Errors (0):**
None.

### Cross-Skill Reference Map
[Show which skills reference which other skills — useful for understanding dependencies]

### Summary
- **10 skills** inspected
- **0 errors**, **2 warnings**
- **All frontmatter valid** ✅
- **0 broken file references** ✅
- **0 orphan files** ✅
```

### Red Flags (Inspect)

- Skipping a skill directory because "it doesn't look like a skill"
- Not actually reading the SKILL.md frontmatter — always read the file
- Flagging `HTML_TEMPLATE.md` as an error (it's a known companion file for hh-think)

---

## 🔵 Mode: Sync (Upload)

Copy all hh skills from `~/.claude/skills/` to the local repo and push to GitHub.

### Prerequisites check

```bash
# Verify repo exists
test -d ~/.claude/hh-skills-repo/.git || echo "REPO_MISSING"

# Verify SSH key exists
test -f ~/.ssh/id_ed25519_personal || echo "SSH_KEY_MISSING"
```

If REPO_MISSING → redirect to Setup mode.

### Step 1: Run quick inspect

Before syncing, run a lightweight version of Inspect (C1 + C2 only — frontmatter validity and name consistency). If there are ERRORS (not warnings), report them and ask: "There are inspection errors. Sync anyway? (y/n)"

### Step 2: Rsync skills to repo

```bash
export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519_personal -o StrictHostKeyChecking=accept-new"

# Sync each hh skill directory (excluding the sync skill itself from the repo)
for d in ~/.claude/skills/hh-* ~/.claude/skills/hh:*; do
    skill_name=$(basename "$d")
    rsync -a --delete "$d/" ~/.claude/hh-skills-repo/"$skill_name"/
done
```

`--delete` ensures removed files in source are also removed in repo.

### Step 3: Detect changes

```bash
cd ~/.claude/hh-skills-repo
git status --porcelain
```

If no changes → "✅ All skills up to date. Nothing to push."

### Step 4: Commit and push

If there are changes:
```bash
cd ~/.claude/hh-skills-repo
git add -A
git diff --cached --stat   # Show summary to user
git commit -m "sync: update hh skills — $(date +%Y-%m-%d)"

# Describe what changed in the commit message:
# - Modified files
# - Added/deleted skills
git push origin main
```

### Step 5: Report

```
## ✅ Sync Complete

**Pushed to:** https://github.com/asd316/claude-hh-skills
**Changes:**
- Modified: hh-diagnose/SKILL.md (frontmatter updated)
- Added: hh-sync/SKILL.md (new skill)

**Commit:** abc1234 — "sync: update hh skills — 2026-06-09"
```

### Red Flags (Sync)

- Pushing without checking inspection results first
- Forgetting to use the personal SSH key (will fail to auth)
- Not pulling before pushing (could miss remote changes)
- Sync-ing `hh:sync-skills` itself — this IS expected, the skill manages itself

---

## 🆕 Mode: Setup (First Time)

Complete initialization from scratch.

### Step 1: Check prerequisites

```bash
# Check SSH key
test -f ~/.ssh/id_ed25519_personal && echo "SSH_KEY_OK" || echo "SSH_KEY_MISSING"

# Check git
which git && echo "GIT_OK"
```

### Step 2: Clone or init repo

```bash
export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519_personal -o StrictHostKeyChecking=accept-new"

# Remove any existing broken directory
rm -rf ~/.claude/hh-skills-repo

# Clone
git clone git@github.com:asd316/claude-hh-skills.git ~/.claude/hh-skills-repo

cd ~/.claude/hh-skills-repo
git config user.name "asd316"
git config user.email "992358904@qq.com"
git config --local url."git@github.com:".insteadOf https://github.com/
```

### Step 3: First sync

Run the Sync mode steps (rsync + commit + push).

### Step 4: Report

```
## 🆕 Setup Complete

- **Repo:** ~/.claude/hh-skills-repo/
- **Remote:** https://github.com/asd316/claude-hh-skills
- **Skills uploaded:** [list]

**Next:** Skills are now live on GitHub. Use `hh:sync-skills` anytime to sync changes.
```

---

## Auto-Sync Hook (Optional)

To auto-sync on every Claude Code session stop, add this hook to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'export GIT_SSH_COMMAND=\"ssh -i ~/.ssh/id_ed25519_personal\"; cd ~/.claude/hh-skills-repo && rsync -a --delete ~/.claude/skills/hh-*/ . /tmp/hh-skills-sync; git add -A && git diff --cached --quiet || git commit -m \"auto-sync: $(date +%Y-%m-%d)\" && git push origin main'"
          }
        ]
      }
    ]
  }
}
```

⚠️ **Caveat:** This will push EVERY session stop, even if you're mid-experiment. Recommend manual sync instead unless you're actively publishing skills.

---

## File Structure Reference

```
~/.claude/
├── skills/                     ← Active skills (source of truth)
│   ├── hh-diagnose/SKILL.md
│   ├── hh-experiment/SKILL.md
│   ├── hh-onboard/SKILL.md
│   ├── hh-runbook/SKILL.md
│   ├── hh-schedule/SKILL.md
│   ├── hh-sync/SKILL.md        ← This skill
│   ├── hh-think/SKILL.md
│   ├── hh-think/HTML_TEMPLATE.md
│   ├── hh-undo/SKILL.md
│   ├── hh-visualize/SKILL.md
│   └── hh:deploy/
│       ├── SKILL.md
│       └── references/
│           ├── commands.md
│           ├── deploy.md
│           ├── env-vars.md
│           ├── git-lfs.md
│           ├── preview.md
│           ├── setup.md
│           └── troubleshoot.md
└── hh-skills-repo/             ← Git working copy (synced to GitHub)
    └── (mirrors skills/ structure)
```
