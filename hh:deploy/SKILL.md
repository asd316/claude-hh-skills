---
name: hh:deploy
description: Manage english-dictory Vercel deployments. Routes user intent to the right reference doc for setup, deploy, preview, env vars, troubleshooting, or Git LFS.
---

# hh-deploy — Vercel deployment router for english-dictory

You are a thin router. Do NOT carry deployment details in this file. Your job is three steps only: **assess state → recognize intent → load the right reference doc**.

## Step 1: Silent state assessment

Run these checks quietly. Only surface a problem if it would block the user's intent.

| Check | How | If failing |
|-------|-----|------------|
| Project linked to Vercel | `test -f .vercel/project.json` | Route to `references/setup.md` first |
| Git LFS configured | `grep -q 'filter=lfs' .gitattributes` | Route to `references/git-lfs.md` first |
| Unpushed commits | `git log origin/main..HEAD --oneline 2>/dev/null` | Warn user before deploy |
| Vercel Git LFS enabled | ⚠️ Cannot auto-detect | Remind user on every deploy: check Dashboard → Settings → Git → Git LFS |

## Step 2: Intent recognition

Match user message keywords to a route. Priority: troubles first, then actions.

| Keywords (Chinese or English) | Intent | Load |
|-------------------------------|--------|------|
| 报错, 失败, 日志, error, fail, log, 排查, debug | 🔴 Troubleshoot | `references/troubleshoot.md` |
| 第一次, 初次, setup, 怎么开始, 配置, link, 关联 | 🆕 First-time setup | `references/setup.md` |
| 部署, 上线, 发布, deploy, --prod, 推到生产 | 🔵 Production deploy | `references/deploy.md` |
| 预览, 先看看, 测试一下, preview, 试试 | 🟣 Preview deploy | `references/preview.md` |
| 环境变量, env, 密钥, 变量, 配置变量 | 🟡 Environment variables | `references/env-vars.md` |
| LFS, 音频, mp3, large file, 大文件, 指针 | 🟠 Git LFS | `references/git-lfs.md` |
| 命令, 查, 参考, 怎么用, help, 有哪些 | 📘 Command reference | `references/commands.md` |

**Priority when multiple keywords match:** Troubleshoot > Setup > Deploy > Preview > Env vars > LFS > Commands

**Default (no match):** Load `references/commands.md` and ask what the user wants to do.

## Step 3: Load reference + act

After routing, Read the target reference file and follow its instructions. The reference file contains the actual commands, steps, and decision logic.

## Context

- **Project:** `asd316s-projects/english-dictory` (Next.js + Git LFS for 5082 mp3 audio files)
- **Git remote:** `https://github.com/asd316/english-dictory.git`
- **Vercel linked:** `.vercel/project.json` should exist after `npx vercel link`
- **Git LFS pattern:** `public/audio/tts/**/*.mp3` in `.gitattributes`

## File structure

```
.claude/skills/hh-deploy/
├── SKILL.md                      ← You are here (router)
└── references/
    ├── setup.md                  ← First-time setup checklist
    ├── deploy.md                 ← Production deployment flow
    ├── preview.md                ← Preview deployment flow
    ├── env-vars.md               ← Environment variable management
    ├── troubleshoot.md           ← Logs & common failures
    ├── git-lfs.md                ← Git LFS operations
    └── commands.md               ← Quick reference command table
```
