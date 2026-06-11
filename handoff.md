# HANDOFF — HOTFIX: add the new `launchnow.cyberg7.com.sg` origin to the CORS allow-list

> **For:** a Claude Code session with write access to **`Cyberg7tech/website-factory`**.
> **From:** the `intake-launchnow` session.
> **Severity:** 🔴 **production funnel is broken right now** — form submissions on the live site
> are blocked by the browser. This is a **one-line fix** + test update + deploy.
> **Verified broken:** 2026-06-11 (curl evidence below).

---

## Context (why)
The funnel's Vercel project was **renamed** (`intake-launchnow` → `launchnow`) and **re-domained**:
the live site now serves at **`https://launchnow.cyberg7.com.sg`** (the old
`intake-launchnow.cyberg7.com.sg` is detached and 404s).

The funnel's `/intake` form and `/status` page call the shared backend
(`agency.cyberg7.com.sg` → routes `POST /api/intake`, `GET /api/status/<jobId>`) **cross-origin from
the browser**. The backend's CORS allow-list in `src/lib/cors.ts` (added 2026-06-05, commit `070e5cf`)
permits **only the OLD origin**. So the browser preflight from the new domain gets **no
`Access-Control-Allow-Origin`** and every submission fails with the generic error message.

Evidence (today):
```
OPTIONS /api/intake  Origin: https://launchnow.cyberg7.com.sg          → 204, NO access-control headers   ← broken
OPTIONS /api/intake  Origin: https://intake-launchnow.cyberg7.com.sg  → 204 + ACAO (old origin still OK)
```

## Scope guardrails
- Touch **only** `src/lib/cors.ts` and its test (`tests/cors.test.ts`). Nothing else.
- Do **not** use `*` or reflect arbitrary origins — extend the explicit allow-list by exactly one entry.
- Keep the old origin in the list (transition safety if the old domain is re-attached as an alias).

---

## Task 1 — EDIT `src/lib/cors.ts` (one line)
Find the `ALLOWED_ORIGINS` set:

```ts
const ALLOWED_ORIGINS = new Set<string>([
  'https://intake-launchnow.cyberg7.com.sg',
  // 'http://localhost:3000', // uncomment for local dev against the funnel
])
```

Add the new origin:

```ts
const ALLOWED_ORIGINS = new Set<string>([
  'https://intake-launchnow.cyberg7.com.sg',
  'https://launchnow.cyberg7.com.sg', // project re-domained 2026-06 (Vercel project renamed to "launchnow")
  // 'http://localhost:3000', // uncomment for local dev against the funnel
])
```

## Task 2 — UPDATE `tests/cors.test.ts`
Your session previously added 7 tests locking the security boundary. Extend them so the suite covers:
- `https://launchnow.cyberg7.com.sg` → **gets** `Access-Control-Allow-Origin: https://launchnow.cyberg7.com.sg`
- `https://intake-launchnow.cyberg7.com.sg` → still gets its ACAO (unchanged)
- a non-allowed origin (e.g. `https://evil.example`) → still gets **no** `access-control-*` headers

## Task 3 — Build, commit, deploy
1. `npm run verify` (typecheck + tests + build) — expect green.
2. Commit (suggested message:
   `fix(api): allow launchnow.cyberg7.com.sg origin on /api/intake + /api/status (project re-domained)`).
3. Deploy to production **`agency.cyberg7.com.sg`** (push to `main` per your normal flow). Note the **commit SHA**.

---

## Task 4 — VERIFY against the live API and PASTE THE OUTPUT BACK
```bash
NEW="https://launchnow.cyberg7.com.sg"
OLD="https://intake-launchnow.cyberg7.com.sg"

echo "### 1) Preflight, NEW origin (expect 204 + ACAO = $NEW)"
curl -s -i -X OPTIONS -H "Origin: $NEW" \
  -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: content-type" \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control'

echo "### 2) POST empty body, NEW origin (expect 400 + ACAO present)"
curl -s -i -X POST -H "Origin: $NEW" -H "Content-Type: application/json" -d '{}' \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control'

echo "### 3) GET status, NEW origin (expect 200 + ACAO present)"
curl -s -i -H "Origin: $NEW" \
  "https://agency.cyberg7.com.sg/api/status/probe?startedAt=0" | grep -iE '^HTTP/|access-control'

echo "### 4) OLD origin unchanged (expect 204 + ACAO = $OLD)"
curl -s -i -X OPTIONS -H "Origin: $OLD" -H "Access-Control-Request-Method: POST" \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control-allow-origin'

echo "### 5) SECURITY: non-allowed origin gets NO ACAO"
curl -s -i -X OPTIONS -H "Origin: https://evil.example" -H "Access-Control-Request-Method: POST" \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control'
```

### Pass criteria
| Check | Expect |
|---|---|
| 1 | `204` + `access-control-allow-origin: https://launchnow.cyberg7.com.sg` + methods + `Content-Type` |
| 2 | `400` + that same ACAO present |
| 3 | `200` + that same ACAO present |
| 4 | `204` + ACAO for the **old** origin (unchanged) |
| 5 | any status, **no** `access-control-*` line (allow-list stays scoped, not `*`) |

## Report back to the `intake-launchnow` session
1. ✅/❌ `npm run verify` green (+ test count).
2. Deployed **commit SHA** + confirmation `agency.cyberg7.com.sg` is live with it.
3. Raw output of **all 5** curl checks.
4. Anything touched beyond the 2 files (ideally nothing).

Once checks 1–3 show ACAO for the new origin (and 5 shows none), the live funnel at
`https://launchnow.cyberg7.com.sg/intake` submits end-to-end again — the intake-launchnow session
will independently re-verify and then run a live submit test.

---

## FYI for the owner (not this session's task)
- The old domain `intake-launchnow.cyberg7.com.sg` now 404s — any ads/links pointing there are dead.
  Consider re-adding it to the Vercel `launchnow` project as a **redirect** to the new domain.
- The funnel repo's design branch (`claude/busy-curie-pjEii-design-glowup`) already carries the
  funnel-side domain fixes (canonicals + a hostname gate that accepts both hosts).
