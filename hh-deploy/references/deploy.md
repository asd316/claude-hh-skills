# Production Deployment

## Recommended: git push auto-deploy

```bash
git add .
git commit -m "your message"
git push origin main
```

Vercel detects the push → builds → deploys to production. No manual steps needed.

Wait ~1-2 minutes, then check.

## Alternative: CLI manual deploy

```bash
vercel --prod
```

Use this when:
- You want to deploy without pushing to GitHub
- You need `--force` to bypass cache
- Testing a quick fix

## Post-deploy verification

```bash
# 1. Check page
curl -s -o /dev/null -w "HTTP %{http_code}\n" https://english-dictory.vercel.app/

# 2. Check audio is real mp3 (not LFS pointer)
curl -s https://english-dictory.vercel.app/audio/tts/tencent-wejack/abase-7cd2908b34da.mp3 | head -1
# Should NOT contain "version https://git-lfs.github.com"

# 3. Check deployment status
vercel ls
vercel inspect <deployment-url>
```

## Before deploying

- [ ] Unpushed commits? Run `git log origin/main..HEAD --oneline`
- [ ] Vercel Git LFS enabled? Cannot auto-check — remind user to verify in Dashboard
- [ ] Audio files committed? `git lfs ls-files | wc -l` should be > 0

## How it works

```
git push → GitHub webhook → Vercel build container:
  1. git clone (LFS pointers, fast)
  2. git lfs pull (download ~200MB mp3 from GitHub LFS, ~10-30s)
  3. npm install
  4. npm run build
  5. Deploy static + serverless to CDN
```

Total build time: ~2-4 minutes (mostly LFS pull + Next.js build).
