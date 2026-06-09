# Environment Variables

## CLI commands

```bash
vercel env pull          # Pull all env vars to .env.local (overwrites!)
vercel env ls            # List all env var names
vercel env add KEY       # Add a new variable (interactive)
vercel env rm KEY        # Remove a variable
```

## This project's variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `NEXT_PUBLIC_TTS_PROVIDER` | `browser` | TTS fallback provider |
| `NEXT_PUBLIC_TTS_VOICE` | `en-US` | Browser speech voice |

These are public (`NEXT_PUBLIC_` prefix). They only affect the browser `SpeechSynthesis` fallback — the primary audio comes from pre-generated mp3 files in Git LFS.

## Environments

Vercel has 3 environments for each variable:

| Environment | When used |
|-------------|-----------|
| Production | `vercel --prod` or main branch push |
| Preview | `vercel` or non-main branch push |
| Development | `vercel dev` (local) |

Add variables to both Production and Preview unless they differ.

## Adding a variable

```bash
# Interactive — Vercel asks for value and environment
vercel env add NEXT_PUBLIC_TTS_PROVIDER

# Then select: Production, Preview, Development
```

Or via Dashboard: Settings → Environment Variables → Add

## Syncing local

```bash
# Pull production vars to .env.local
vercel env pull

# The file is gitignored — safe for secrets
```

## Troubleshooting

- "Variable not found" at runtime → Check it's added to the correct environment (Production vs Preview)
- `vercel env pull` overwrites `.env.local` → Back up first if you have local-only vars
- Public vars MUST start with `NEXT_PUBLIC_` to be available in browser
