# Command Quick Reference

## Vercel CLI

| Command | What it does |
|---------|-------------|
| `npx vercel link` | Link local project to Vercel (one-time) |
| `vercel` | Create preview deployment (temporary URL) |
| `vercel --prod` | Deploy to production |
| `vercel --prod --force` | Force redeploy (skip cache) |
| `vercel ls` | List recent deployments |
| `vercel inspect <url>` | Show deployment details (build, config, functions) |
| `vercel logs <url>` | Show deployment logs |
| `vercel logs <url> --follow` | Live tail logs |
| `vercel logs <url> --level error` | Show only errors |
| `vercel env pull` | Download env vars to `.env.local` |
| `vercel env ls` | List configured env var names |
| `vercel env add <name>` | Add an env var (interactive) |
| `vercel env rm <name>` | Remove an env var |
| `vercel dev` | Run local dev with Vercel env vars |
| `vercel whoami` | Show logged-in user |

## Git LFS

| Command | What it does |
|---------|-------------|
| `git lfs track "<pattern>"` | Add LFS tracking rule |
| `git lfs ls-files` | List all LFS-tracked files |
| `git lfs ls-files \| wc -l` | Count LFS files |
| `git lfs pull` | Download real files (replace pointers) |
| `git lfs checkout` | Ensure working tree has real files (same as pull) |
| `git lfs push origin main` | Upload LFS objects to server |
| `git lfs migrate import --include="<pattern>" --everything` | Convert existing files to LFS (rewrites history!) |

## Git (deployment-related)

| Command | What it does |
|---------|-------------|
| `git log origin/main..HEAD --oneline` | Show unpushed commits |
| `git push origin main` | Push to GitHub → triggers Vercel auto-deploy |

## Verification

| Command | What it does |
|---------|-------------|
| `curl -s -o /dev/null -w "%{http_code}\n" <url>` | Check HTTP status |
| `curl -s <audio-url> \| head -1` | Verify audio is real mp3 (not LFS pointer) |
| `test -f .vercel/project.json && echo linked` | Check if project is linked |
| `grep 'filter=lfs' .gitattributes` | Check if LFS is configured |
