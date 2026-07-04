# Preview Deployment

Preview deploys create a temporary URL for testing before going to production.

## Create a preview

```bash
vercel
```

This uploads current working directory (even uncommitted changes!) and builds a preview. Returns a URL like:

```
https://english-dictory-abc123.vercel.app
```

## When to use preview

- Large changes you want to verify before merging
- Showing someone a feature for review
- Testing build without affecting production
- Debugging build issues

## Preview vs Production

| | Preview | Production |
|---|---------|------------|
| URL | Random hash `.vercel.app` | Custom domain or project default |
| Env vars | Preview environment | Production environment |
| Trigger | `vercel` CLI or non-main branch push | `vercel --prod` or main branch push |
| Impact | Isolated, no user impact | Live site |

## Promote preview to production

Option 1 — CLI:
```bash
vercel --prod
```

Option 2 — Dashboard:
Go to the preview deployment → "Promote to Production"

## Verify preview

```bash
# Check the preview URL
curl -s -o /dev/null -w "HTTP %{http_code}\n" <preview-url>

# Test audio
curl -s <preview-url>/audio/tts/tencent-wejack/abase-7cd2908b34da.mp3 | head -1
```

## Caveats

- `vercel` uploads from local disk — make sure `git lfs checkout` has been run so mp3 files are real binaries, not LFS pointers
- Preview env vars may differ from production — check Dashboard if things behave differently
