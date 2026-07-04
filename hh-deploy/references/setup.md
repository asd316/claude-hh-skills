# First-time Vercel Setup

Complete these steps once. After that, `git push` triggers auto-deploy.

## Checklist

### 1. Link local project to Vercel

```bash
npx vercel link
```

This creates `.vercel/project.json`. The project is `asd316s-projects/english-dictory`.

### 2. Connect GitHub repository (Dashboard)

- Go to [vercel.com](https://vercel.com) → your project
- Settings → Git → Connect `asd316/english-dictory`
- This enables auto-deploy on every `git push`

### 3. Enable Git LFS (Dashboard) — CRITICAL

- Settings → Git → toggle **Git LFS** ON
- Without this, mp3 files will be 130-byte LFS pointers instead of real audio
- Must redeploy after toggling

### 4. Configure build settings

Vercel auto-detects Next.js. Defaults should work:

| Setting | Value |
|---------|-------|
| Framework | Next.js |
| Build Command | `npm run build` |
| Output Directory | `.next` |
| Install Command | `npm install` |
| Node.js Version | 20.x (or auto) |

### 5. Add environment variables (Dashboard)

Settings → Environment Variables. Add:

| Name | Value | Environment |
|------|-------|-------------|
| `NEXT_PUBLIC_TTS_PROVIDER` | `browser` | Production, Preview |
| `NEXT_PUBLIC_TTS_VOICE` | `en-US` | Production, Preview |

The app uses pre-generated audio (in Git LFS), so TTS-related variables only affect the browser fallback path.

### 6. Trigger first deploy

- Push to `main` branch → auto-deploy triggers
- Or from Dashboard: Deployments → "Redeploy"

### 7. Verify deployment

```bash
# Check page loads
curl -s -o /dev/null -w "%{http_code}\n" https://english-dictory.vercel.app/

# Check audio file is real mp3 (not LFS pointer)
curl -s https://english-dictory.vercel.app/audio/tts/tencent-wejack/abase-7cd2908b34da.mp3 | head -1
# Should NOT show "version https://git-lfs.github.com..." — if it does, Git LFS is not enabled
```

If the audio returns LFS pointer text (130 bytes), Git LFS is NOT enabled in Vercel Dashboard. Go back to step 3.
