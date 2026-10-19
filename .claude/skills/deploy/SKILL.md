---
name: deploy
description: Deploy and verify the extremegooning site — the static pages (GitHub Pages at extremegooning.org), the Cloudflare roast-proxy Worker in worker/, and a pointer to the Firebase rules deploy. Use when asked to deploy, ship, publish, push live, redeploy the worker, launch the site locally, or check why a change isn't showing up on the live site.
---

# Deploying extremegooning

Three independent pieces, each deployed differently. A change to one does
not deploy the others — e.g. editing the roast prompt in
[worker/roast-worker.js](../../../worker/roast-worker.js) does nothing on
the live site until the Worker is redeployed, and editing
[matchups.html](../../../matchups.html) does nothing until it's pushed.

| Piece | Source | Deployed how |
|---|---|---|
| Static site (`*.html`) | repo root, [CNAME](../../../CNAME) = `extremegooning.org` | `git push` to `main` (GitHub Pages) |
| Roast proxy Worker | [worker/](../../../worker/) | `npx wrangler deploy` |
| Firebase RTDB rules | [database.rules.json](../../../database.rules.json) | see the `firebase-security` skill |

**Everything here is outward-facing.** Confirm with the user before pushing
or deploying, unless they've just explicitly asked for it. A prior "deploy"
approval covers that one deploy, not later ones. Never commit or push
unasked.

## Run locally (preview before shipping)

It's a plain static site; no build step.

```bash
cd /Users/pmistry/extremegooning && python3 -m http.server 8000
# open http://localhost:8000/matchups.html
```

Use port **8000** (or 8080): the Worker's `ALLOWED_ORIGINS` only lists
`localhost:8000`, `localhost:8080`, `127.0.0.1:8000` and
`https://extremegooning.org`, so any other port gets `forbidden_origin`
from the Worker and the page silently falls back to the local template
roasts. The site's password gate allows local access.

## Deploying the static site

Pages are served from `main`. Deploy = commit + push:

```bash
git status && git diff --stat      # review what's going out
git add <files> && git commit -m "..."
git push origin main
```

Pages usually updates within a minute or two. Browsers cache aggressively;
hard-refresh (Cmd+Shift+R) before concluding a change didn't ship.

## Deploying the roast Worker

Config: [worker/wrangler.jsonc](../../../worker/wrangler.jsonc) (name
`extremegooning-roast-proxy`, KV `ROAST_LIMITS`, rate limiter
`ROAST_RATE_LIMITER`). Live URL is `ROAST_API_URL` in `matchups.html`.

```bash
cd /Users/pmistry/extremegooning/worker
npm install            # first time / after a fresh clone only
cp roast-worker.js "$TMPDIR/check.mjs" && node --check "$TMPDIR/check.mjs"   # syntax check (ES module)
npx wrangler deploy
```

(`node --check roast-worker.js` directly fails with "Unexpected token
'export'" because the file is an ES module under a CommonJS
`package.json` — copy it to a `.mjs` file in the scratchpad to check it.)

### Authentication

`wrangler deploy` needs Cloudflare credentials. In a non-interactive
session it fails with *"it's necessary to set a CLOUDFLARE_API_TOKEN
environment variable"*. Options:

- The user runs `npx wrangler login` themselves (opens a browser, so only
  they can do it) — in Claude Code they can type `! npx wrangler login`
  to run it in this session. Then retry the deploy.
- The user starts Claude Code with `CLOUDFLARE_API_TOKEN` set (token from
  the "Edit Cloudflare Workers" template). **Never ask for the token in
  chat, and never write it to a file or commit it.**
- **Do not** use `wrangler deploy --temporary` as a workaround: it
  deploys to a throwaway account, not the real worker, so the live site
  keeps the old behavior while looking like it shipped.

### Secrets

The Worker reads two secrets (never in the repo):
- `ANTHROPIC_API_KEY` — required; set with
  `npx wrangler secret put ANTHROPIC_API_KEY`.
- `SITE_KEY` — optional header check; must match `ROAST_SITE_KEY` in
  `matchups.html`. Set with `npx wrangler secret put SITE_KEY`.

List what's set with `npx wrangler secret list` (names only). The user
types secret values themselves; don't ask for them in chat.

### After a Worker deploy

1. **Bump the cache prefix.** `ROAST_CACHE_PREFIX` in `matchups.html`
   (currently `roast_vN_`) caches AI roasts in each visitor's
   localStorage. If the prompt or the stats sent to it changed, increment
   `N` and push the page too, otherwise returning visitors keep seeing
   old-style roasts.
2. **Smoke test** with the site's own origin and key (the key is already
   public in the page source):
   ```bash
   curl -s -X POST https://extremegooning-roast-proxy.royal-water-14cf.workers.dev \
     -H "Origin: https://extremegooning.org" \
     -H "Content-Type: application/json" \
     -H "X-Roast-Key: <ROAST_SITE_KEY from matchups.html>" \
     -d '{"teamA":"Test A","scoreA":101.25,"statsA":"Carried by: Test Player (RB) 30.10.","teamB":"Test B","scoreB":88.5,"statsB":"All starters are done."}'
   ```
   Expect `{"roast":"..."}`. Each smoke test counts against the daily cap
   (`DAILY_ROAST_CAP` in the Worker), so run it once, not in a loop.
3. Failure codes: `forbidden_origin` / `forbidden_missing_site_key`
   (origin or key mismatch), `rate_limited` (10 req/min/IP),
   `daily_cap_reached`, `upstream_error` (Anthropic key or model problem),
   `empty_response`.

### Roll back

`npx wrangler rollback` restores the previous Worker version, or `git
revert` the change and redeploy.

## Firebase rules

Separate flow with its own auth handoff — use the `firebase-security`
skill. Never loosen the rules as part of an unrelated deploy.

## Guardrails

- Don't commit or push, and don't run `wrangler deploy`, without the user
  asking for that deploy in this conversation.
- `worker/node_modules/` and `.wrangler/` are gitignored; keep them that
  way. Never commit API tokens, Anthropic keys, or Firebase service-account
  or CLI credentials.
- Set a hard monthly spend cap on the Anthropic key in the Anthropic
  Console; the Worker's limits are a deterrent, not a financial backstop.
