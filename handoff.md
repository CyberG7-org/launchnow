# HANDOFF — Add CORS to `website-factory` so the LaunchNow intake funnel can call the API

> **For:** a Claude Code session with write access to the **`Cyberg7tech/website-factory`** repo.
> **From:** the `intake-launchnow` session (separate repo/site).
> **Goal:** allow the browser on **`https://intake-launchnow.cyberg7.com.sg`** to call
> `POST /api/intake` and `GET /api/status/<jobId>` on **`agency.cyberg7.com.sg`** cross-origin.
> Today those routes send **no CORS headers**, so the browser blocks the calls. This adds a
> tight allow-list (one origin, **not** `*`). No other behavior changes.

---

## Context (why)
`intake-launchnow.cyberg7.com.sg` now hosts a native `/intake` form + `/status` page that POST the
brief to `agency.cyberg7.com.sg/api/intake` and poll `agency.cyberg7.com.sg/api/status/<jobId>`
directly from the browser. The frontend is built and waiting; the **only** thing missing is CORS on
these two backend routes. The API itself is already live on `main` (verified: `POST /api/intake` → 400
on empty body; `GET /api/status/<id>?startedAt=0` → 200). **No schema/tenant change is needed** — only
the CORS headers below.

## Scope guardrails (please respect)
- **Only** touch the 3 files below. Do **not** change request validation, the pipeline, Airtable, or
  status logic. The original handler bodies must run **unchanged** (we just wrap them).
- Allow-list **exactly** `https://intake-launchnow.cyberg7.com.sg`. Do **not** use `*` or reflect
  arbitrary origins.
- The repo uses the `@/*` path alias → `src/*` (other files import `@/lib/...`), so `src/lib/cors.ts`
  is imported as `@/lib/cors`. Confirm `tsconfig.json` has that mapping (it should).

---

## Task 1 — CREATE new file: `src/lib/cors.ts`
Create it with **exactly** this content:

```ts
/**
 * CORS for the LaunchNow intake funnel.
 *
 * The funnel runs on a SEPARATE origin (https://intake-launchnow.cyberg7.com.sg)
 * and calls `/api/intake` (POST) and `/api/status/<jobId>` (GET) directly from the
 * browser. Those routes otherwise send no CORS headers, so the browser blocks the
 * cross-origin calls. This adds an explicit allow-list (NOT `*`) for that one origin.
 *
 * Used by:
 *   - src/app/api/intake/route.ts
 *   - src/app/api/status/[jobId]/route.ts
 */

const ALLOWED_ORIGINS = new Set<string>([
  'https://intake-launchnow.cyberg7.com.sg',
  // 'http://localhost:3000', // uncomment for local dev against the funnel
])

/** CORS headers for a given request Origin (empty CORS unless the origin is allow-listed). */
export function corsHeaders(origin: string | null): Record<string, string> {
  const headers: Record<string, string> = { Vary: 'Origin' }
  if (origin && ALLOWED_ORIGINS.has(origin)) {
    headers['Access-Control-Allow-Origin'] = origin
    headers['Access-Control-Allow-Methods'] = 'GET, POST, OPTIONS'
    headers['Access-Control-Allow-Headers'] = 'Content-Type'
    headers['Access-Control-Max-Age'] = '86400'
  }
  return headers
}

/** Copy the CORS headers onto an existing Response. */
export function withCors(res: Response, origin: string | null): Response {
  for (const [key, value] of Object.entries(corsHeaders(origin))) {
    res.headers.set(key, value)
  }
  return res
}

/** Preflight (OPTIONS) response. */
export function preflight(request: Request): Response {
  return new Response(null, {
    status: 204,
    headers: corsHeaders(request.headers.get('origin')),
  })
}
```

---

## Task 2 — PATCH `src/app/api/intake/route.ts`
Two surgical changes. Do not alter anything else.

**(a) Add the import** right after the existing `slugify` import:
```ts
import { slugify } from '@/lib/slugify'
import { withCors, preflight } from '@/lib/cors'   // ← ADD THIS LINE
```

**(b) Wrap the handler.** Find this line:
```ts
export async function POST(request: Request): Promise<Response> {
```
…and replace **only that one line** with this block (the rest of the function body stays exactly as-is —
it just becomes the body of `handlePost`):
```ts
export function OPTIONS(request: Request): Response {
  return preflight(request)
}

export async function POST(request: Request): Promise<Response> {
  return withCors(await handlePost(request), request.headers.get('origin'))
}

async function handlePost(request: Request): Promise<Response> {
```
(Net effect: original `POST` body is renamed to `handlePost`; a thin `POST` wrapper adds CORS headers to
every response it returns; `OPTIONS` answers the preflight.)

---

## Task 3 — PATCH `src/app/api/status/[jobId]/route.ts`
Same pattern.

**(a) Add the import** right after the existing `status-fallback` import:
```ts
import { resolveFallbackPreview } from '@/lib/status-fallback'
import { withCors, preflight } from '@/lib/cors'   // ← ADD THIS LINE
```

**(b) Wrap the handler.** Find this 4-line signature:
```ts
export async function GET(
  request: Request,
  context: { params: Promise<{ jobId: string }> },
): Promise<Response> {
```
…and replace it with this block (the rest of the function body stays exactly as-is — it becomes the body
of `handleGet`):
```ts
export function OPTIONS(request: Request): Response {
  return preflight(request)
}

export async function GET(
  request: Request,
  context: { params: Promise<{ jobId: string }> },
): Promise<Response> {
  return withCors(await handleGet(request, context), request.headers.get('origin'))
}

async function handleGet(
  request: Request,
  context: { params: Promise<{ jobId: string }> },
): Promise<Response> {
```

---

## Task 4 — Build, commit, deploy
1. `npm run build` (or `npx tsc --noEmit`) — confirm it type-checks/builds. Expected: no new errors.
2. Commit (suggested branch `feat/intake-launchnow-cors`, message:
   `feat(api): CORS allow-list for intake-launchnow.cyberg7.com.sg on /api/intake + /api/status`).
3. Deploy to **`agency.cyberg7.com.sg`** via your normal process (merge to the production branch / Vercel
   deploy). Note the deployed **commit SHA**.

---

## Task 5 — VERIFY (run these after the deploy is live) and PASTE THE OUTPUT BACK
```bash
ORIGIN="https://intake-launchnow.cyberg7.com.sg"

echo "### 1) Preflight OPTIONS /api/intake (expect 204 + ACAO for our origin)"
curl -s -i -X OPTIONS -H "Origin: $ORIGIN" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type" \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control'

echo "### 2) POST /api/intake empty body (expect 400 + ACAO present)"
curl -s -i -X POST -H "Origin: $ORIGIN" -H "Content-Type: application/json" -d '{}' \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control'

echo "### 3) GET /api/status (expect 200 + ACAO present)"
curl -s -i -H "Origin: $ORIGIN" \
  "https://agency.cyberg7.com.sg/api/status/probe?startedAt=0" | grep -iE '^HTTP/|access-control'

echo "### 4) SECURITY: non-allowed origin must get NO ACAO header"
curl -s -i -X OPTIONS -H "Origin: https://evil.example" \
  -H "Access-Control-Request-Method: POST" \
  https://agency.cyberg7.com.sg/api/intake | grep -iE '^HTTP/|access-control'
```

### Expected results (this is the PASS criteria)
| Check | Expect |
|---|---|
| 1 | `HTTP/2 204` **and** `access-control-allow-origin: https://intake-launchnow.cyberg7.com.sg` **and** `access-control-allow-methods: GET, POST, OPTIONS` **and** `access-control-allow-headers: Content-Type` |
| 2 | `HTTP/2 400` **and** `access-control-allow-origin: https://intake-launchnow.cyberg7.com.sg` present |
| 3 | `HTTP/2 200` **and** `access-control-allow-origin: https://intake-launchnow.cyberg7.com.sg` present |
| 4 | Any status, but **NO** `access-control-allow-origin` line at all (proves the allow-list is scoped, not `*`) |

---

## Report back to the `intake-launchnow` session
Please reply with:
1. ✅/❌ build type-checks clean.
2. The **commit SHA** + the website-factory branch you deployed, and confirmation the agency deploy is **live**.
3. The **raw output of all 4 curl checks** from Task 5 (status line + every `access-control-*` header).
4. Anything you had to change beyond the 3 files (ideally nothing), or any deviation from the expected table.

Once I (the intake-launchnow session) see checks **1–3 return ACAO for our origin and 4 returns none**,
I will flip `BACKEND_READY=true` (merge the prepared `claude/busy-curie-pjEii-go-live` branch) and we'll
run the end-to-end test (submit a brief → job created → `/status` walks the pipeline → lands on
`<slug>-preview.cyberg7.com.sg`).

> If any check fails — especially missing `access-control-allow-origin` on 1–3, or an unresolved
> `@/lib/cors` import — paste the error and the relevant route file's top 40 lines so I can adjust.
