---
name: firebase-security
description: Audit and manage the extremegooning Firebase Realtime Database's security posture — check whether reads/writes are actually locked down, edit database.rules.json, and deploy rules via the Firebase CLI. Use when asked to check if the site/database is secure, tighten Firebase access, deploy rule changes, or investigate why writes to the leaderboard are failing/succeeding.
---

# Firebase security management (extremegooning)

This site (`index.html`) talks directly to a public Firebase Realtime
Database from client-side JS — there is no backend between the browser and
the database. The `firebaseConfig` object (including `apiKey` and
`databaseURL`) is not secret; it ships in every page load and is visible to
anyone who views source. **The only real access control is the Realtime
Database security rules**, deployed from [database.rules.json](../../database.rules.json)
in the repo root, plus the API key's HTTP-referrer restriction (configured
in the Firebase console, not in this repo).

Current model: `entries` is publicly **readable** (it's meant to be a public
leaderboard) but **writes require an authenticated user** —
`.write: "auth != null"`. The site signs visitors in anonymously
(`signInAnonymously`, see `index.html`'s `<script type="module">` block)
before any `push`/`set`/`remove` call, so the UI keeps working without a
login screen, while a bare `curl PUT`/`DELETE` against the database URL with
no token gets rejected.

## Auditing current exposure

Read access (should always succeed — it's intentionally public):
```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  "https://extremegooning-default-rtdb.firebaseio.com/entries.json"
```

Unauthenticated write access (should return 401 — if this ever returns 200,
the database is wide open and needs immediate attention):
```bash
curl -s -X PUT -d '{"probe":"security_check"}' \
  "https://extremegooning-default-rtdb.firebaseio.com/entries/security_test_probe.json" \
  -w "\nHTTP %{http_code}\n"
```
If it returns 200, immediately redeploy the locked-down rules (below), then
delete the stray `security_test_probe` node with the Firebase console or
CLI once writes are locked down again.

Whether Anonymous sign-in is actually enabled for the project (determines
whether the site's own Add/Edit/Delete buttons will work):
```bash
curl -s -X POST \
  "https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=<the apiKey from index.html>" \
  -H "Content-Type: application/json" -H "Referer: https://extremegooning.org/" \
  -d '{"returnSecureToken":true}'
```
- `CONFIGURATION_NOT_FOUND` → Authentication has never been initialized for
  this project. This is a **one-time, console-only** step with no CLI/API
  equivalent found so far: Firebase Console → Build → Authentication →
  "Get started" → Sign-in method → enable **Anonymous**.
- A JSON response with `idToken`/`localId` → anonymous auth is live and
  working.

## Editing and deploying rules

Rules live in [database.rules.json](../../database.rules.json) at the repo
root, referenced by [firebase.json](../../firebase.json) and
[.firebaserc](../../.firebaserc) (project id `extremegooning`). Edit that
file like any other repo file, then deploy:

```bash
npx -y firebase-tools deploy --only database --project extremegooning
```

(`npx -y firebase-tools` avoids needing a global install; if `npm`
complains about a root-owned cache, set `npm_config_cache` to a scratch
directory rather than `sudo chown`-ing the user's real npm cache.)

### Authenticating the CLI

If `npx firebase-tools projects:list` shows "No authorized accounts,"
the CLI needs a login. Interactive `firebase login` expects a local
browser redirect that isn't reachable from this environment, so use the
device-code-style flow instead:

```bash
npx -y firebase-tools login --no-localhost
```
This prints a URL. The user opens it in their own browser (logged into
the Google account that owns the Firebase project), approves access, and
is redirected to a URL containing a `code=...` query parameter — hand that
whole code back to:
```bash
npx -y firebase-tools login "<code>" --no-localhost
```
This is a **credential handoff that only the project owner can perform** —
never fetch or guess that URL yourself, and never reuse a code across
different Google accounts/projects.

## Guardrails

- Don't loosen `.write` back to `true` without the user explicitly asking
  for it and understanding the tradeoff (it reopens the exact vandalism
  vector this setup closes).
- Don't add new writable paths under `entries` (or new top-level nodes)
  without equivalent auth-gated rules — an unguarded sibling node is just
  as exploitable as the original open database was.
- Treat the Firebase `apiKey`, `databaseURL`, and project id as public,
  non-sensitive values (safe to log, paste, commit) — but never touch or
  paste Firebase Admin SDK service-account keys or CLI refresh tokens the
  same way; those are real secrets and must never be committed or logged.
