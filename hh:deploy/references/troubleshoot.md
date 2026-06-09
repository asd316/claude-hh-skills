# Troubleshooting & Logs

## View logs

```bash
vercel logs <deployment-url>              # Recent logs
vercel logs <deployment-url> --follow     # Live tail
vercel logs <deployment-url> --level error  # Errors only
vercel inspect <deployment-url>           # Build details, config
vercel ls                                 # List recent deployments
```

## Common failures

### 1. Audio files return 130 bytes (LFS pointer text)

**Symptom:**
```bash
curl https://xxx.vercel.app/audio/tts/tencent-wejack/abase-xxx.mp3 | head -1
# Returns: version https://git-lfs.github.com/spec/v1
```

**Cause:** Vercel Git LFS is NOT enabled.

**Fix:** Dashboard → Settings → Git → toggle Git LFS ON → Redeploy.

### 2. Audio file 404

**Symptom:** HTTP 404 on audio URLs that exist locally.

**Cause:** 
- LFS pull failed during build
- Or mp3 not committed/pushed

**Fix:**
```bash
# Check local LFS status
git lfs ls-files | wc -l    # should be 5082+
git lfs ls-files --all      # check * vs - markers
```
If files are `-` (local only): `git lfs push origin main`
If files are missing: run `npm run generate:tts` then commit + push.

### 3. Build fails with "Module not found"

**Symptom:** Build error, can't find a dependency.

**Fix:**
```bash
rm -rf node_modules
npm install
git add package-lock.json
git commit -m "fix: update dependencies"
git push
```

### 4. Environment variable not working

**Symptom:** App behaves differently on Vercel vs local.

**Fix:**
```bash
# Check what's configured
vercel env ls

# Pull to compare
vercel env pull
diff .env.local .env.example  # see differences
```
Remember: Vercel has separate vars for Production / Preview / Development.

### 5. Build timeout

**Symptom:** Deployment fails with timeout.

**Cause:** LFS pull of 200MB audio can take time. Default timeout is 300s — should be enough.

**Fix:**
- Check if LFS endpoint is reachable
- Redeploy (transient network issue)
- If persistent: increase function timeout in Vercel Dashboard → Settings → Functions

## Quick diagnostic script

```bash
echo "=== Vercel link ===" && test -f .vercel/project.json && echo "linked" || echo "NOT LINKED"
echo "=== LFS config ===" && grep 'filter=lfs' .gitattributes || echo "NOT CONFIGURED"
echo "=== LFS files ===" && git lfs ls-files | wc -l
echo "=== Unpushed ===" && git log origin/main..HEAD --oneline || echo "up to date"
echo "=== Latest deploy ===" && vercel ls 2>/dev/null | head -3
```
