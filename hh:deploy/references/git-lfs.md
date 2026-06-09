# Git LFS Operations

## How it works

```
Git repository stores:   130-byte pointer (version + oid + size)
GitHub LFS server stores: actual mp3 binary (~30KB each)

On clone:     git lfs replaces pointers with real files
On Vercel:    build container runs git lfs pull automatically
```

`.gitattributes` rule: `public/audio/tts/**/*.mp3 filter=lfs diff=lfs merge=lfs -text`

## Current state (english-dictory)

- **5,082** mp3 files tracked in LFS (wejack voice)
- **~200 MB** stored on GitHub LFS server
- GitHub free tier: 1 GB storage + 1 GB bandwidth/month — well within limits
- `public/audio/tts/tencent-wejames/` is gitignored (experimental voice, subset of wejack)

## Daily commands

```bash
git lfs track "public/audio/tts/**/*.mp3"   # Add tracking pattern
git lfs ls-files                             # List tracked files (* = on server, - = local only)
git lfs ls-files | wc -l                     # Count
git lfs pull                                 # Download all LFS files (replace pointers with real)
git lfs checkout                             # Same as pull — ensure working tree has real files
git lfs push origin main                     # Push LFS objects to server
```

## Adding new audio files

When you run `npm run generate:tts` and get new mp3 files:

```bash
git add public/audio/tts/
# ↑ .gitattributes rule automatically routes mp3 to LFS — no extra step needed

git commit -m "Add new TTS audio"
git push origin main
# ↑ pushes LFS pointers to repo + binary to LFS server
```

## Vercel build-time LFS flow

```
1. Vercel clones repo → gets LFS pointers (fast, 130 bytes each)
2. Vercel runs git lfs pull → downloads ~200MB from GitHub LFS (~10-30s)
3. npm run build → audio files are real mp3s in public/
4. Deploy → audio served from Vercel CDN
```

## Troubleshooting

### Files showing as modified after push
After `git push`, LFS files may show as modified locally. Run:
```bash
git lfs checkout   # Replace pointers with real files
git status         # Should be clean
```

### New mp3s committed as regular files (not LFS)
Check `.gitattributes` exists and has the mp3 pattern:
```bash
cat .gitattributes
# Must contain: public/audio/tts/**/*.mp3 filter=lfs diff=lfs merge=lfs -text
```

### After clone: audio files are 130 bytes
```bash
git lfs pull    # Download actual binaries
```

### Vercel build still shows LFS pointers
Vercel Dashboard → Settings → Git → Toggle **Git LFS** ON → Redeploy.
