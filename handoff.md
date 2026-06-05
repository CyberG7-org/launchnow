# LaunchNow Intake Funnel — Completion Handover

> **Status:** ✅ **LIVE in production** · shipped & end-to-end verified **2026-06-05**.
> **What this is:** the native `/intake` → `/status` funnel on `intake-launchnow.cyberg7.com.sg`
> that turns a visitor's brief into an AI-generated website preview via the shared `website-factory`
> backend. This document describes the **completed job** — what was built, how it works, what's live,
> how it was verified, and what's intentionally left as fast-follows.

---

## 0. TL;DR
- Clicking **Get Started** anywhere on `intake-launchnow.cyberg7.com.sg` now lands on a **native
  `/intake`** page **owned by this site** (not a proxy to the agency).
- The form submits directly (browser → CORS) to the shared backend **`agency.cyberg7.com.sg/api/intake`**,
  which returns a `jobId`.
- A native **`/status`** page polls **`agency.cyberg7.com.sg/api/status/<jobId>`**, shows the real
  pipeline, and redirects to the finished preview at **`<slug>-preview.cyberg7.com.sg`**.
- **Free-preview-first** (no payment to see the preview). **No agency page-proxy.** No `tenant` concept.
- **Verified end-to-end** with a real browser submit → job `4bfb2caf-088d-407b-978d-434533e83152` →
  7 pipeline stages in ~3.4 min → live preview `https://pub-echo-preview.cyberg7.com.sg` (HTTP 200).

---

## 1. The goal (achieved)
Original objective: `intake-launchnow.cyberg7.com.sg/intake` must be a page **owned and served by the
intake-launchnow site itself**, that creates a preview by **reusing the existing website-factory
backend** — explicitly **NOT** a redirect/proxy that executes the agency's `/launchnow/intake` page.
✅ Met: `/intake` and `/status` are native static pages in this repo; only JSON data calls cross to the
backend; `vercel.json` contains **no** agency proxy.

---

## 2. Live coordinates
| | |
|---|---|
| Funnel site (prod) | https://intake-launchnow.cyberg7.com.sg  (Vercel project `cyberg7/intake-launchnow`) |
| Funnel repo | `Cyberg7tech/intake-launchnow` |
| Backend (shared engine) | https://agency.cyberg7.com.sg  (Vercel project `cyberg7-agency`) |
| Backend repo | `Cyberg7tech/website-factory` (private) |
| Generated previews | `<slug>-preview.cyberg7.com.sg` (wildcard DNS + backend middleware — host-agnostic) |

---

## 3. How it works (the flow)
```
Visitor on intake-launchnow.cyberg7.com.sg
   │  clicks "Get Started" (all 19 CTAs → /intake)
   ▼
/intake  (native page, this repo — intake.html)
   │  fills brief, ticks PDPA consent, submits
   │  browser POST (CORS) → https://agency.cyberg7.com.sg/api/intake   { ...intakeSchema }
   ▼
backend returns { jobId }   (UUIDv4 — opaque)
   │  redirect → /status?job=<jobId>
   ▼
/status  (native page, this repo — status.html)
   │  polls https://agency.cyberg7.com.sg/api/status/<jobId>  (CORS)
   │  renders snapshot.currentStage: researching → designing → generating_assets
   │            → building → testing → deploying → ready
   ▼
on "ready": redirect → previewUrl  ( https://<slug>-preview.cyberg7.com.sg )
```
Payment (S$288 deposit) happens **after** the preview, on the brand subdomain — **out of scope for this
repo** by design.

---

## 4. What's in THIS repo (`intake-launchnow`)
Pure static HTML, no build step (Vercel framework = null, `cleanUrls` maps `/intake` → `intake.html`).

| File | What changed / is |
|---|---|
| `intake.html` | **NEW.** LaunchNow brief page, cloned from `contact.html` chrome (identical dark/purple Outfit design). Fields match `intakeSchema` exactly; PDPA consent checkbox; off-screen honeypot; `gtag('generate_lead')` on submit; submit handler with `checkoutUrl` open-redirect allow-list + redirect to `/status?job=`. |
| `status.html` | **NEW.** `noindex` progress page. Polls `/api/status/<jobId>` with exponential backoff + jitter, hidden-tab pause, `Retry-After`, 15-min cap; friendly stage labels; redirects to `previewUrl` on ready; shows `failureReason` on failure. |
| `index/about/contact/pdpa/privacy/terms.html` | All **19** "Get Started" CTAs repointed from the old external `https://launchnow.cyberg7.com.sg/intake` → relative **`/intake`**. |
| `vercel.json` | Unchanged from baseline (`cleanUrls`, `/landing` redirect). **No proxy/rewrites** — the rejected agency-proxy approach was never shipped. |

**The `BACKEND_READY` switch:** both `intake.html` and `status.html` have a single
`var BACKEND_READY = …` constant. `true` = live cross-origin calls (production). `false` = the page
shows a friendly "previews going live shortly" message and fires nothing (used while CORS/deploy was
pending). **Production `main` is `true`.**

---

## 5. The backend dependency (`website-factory`) — CORS patch (DEPLOYED)
The funnel calls the backend cross-origin, which required CORS (the routes had none). A second session
applied and deployed this (commit **`070e5cf`** on `website-factory` `main`, live on `agency.cyberg7.com.sg`):
- **NEW** `src/lib/cors.ts` — allow-list for **only** `https://intake-launchnow.cyberg7.com.sg` (not `*`),
  with `Vary: Origin`, methods `GET, POST, OPTIONS`, header `Content-Type`.
- **`src/app/api/intake/route.ts`** & **`src/app/api/status/[jobId]/route.ts`** — added an `OPTIONS`
  preflight export and wrapped the original handlers (`handlePost`/`handleGet`) so every response carries
  CORS. Original logic unchanged. (+ a `cors.test.ts` locking the "non-allowed origin → no ACAO" boundary.)
- No other backend change was needed — `/api/intake` and `/api/status/<jobId>` were already live on `main`;
  there is **no funnel-branch merge and no Airtable `tenant` column** involved.

---

## 6. The contract (pinned from website-factory `main`)
**`POST /api/intake`** — body validated by Zod `intakeSchema`:
- **Required:** `brand`, `email`, `industry`, `competitorUrls` (array of 1–3 public http(s) URLs, ≥1),
  `goal`, `audience`.
- **Optional:** `businessType`, `primaryGoal` (enums), `vibe` (`professional|editorial|minimal|bold`),
  `primaryColor`/`accentColor` (hex), `logoFileName`, `clerkUserId`, `siteUrl`.
- Unknown keys are stripped (`z.object`). **No `tenant` field** (that was the unmerged
  `LaunchNow-HTML-connect` branch, not `main`). **No `phone`.**
- **Returns** `{ jobId }` (UUIDv4). **No `checkoutUrl`** at intake (deposit is post-preview).

**`GET /api/status/<jobId>`** → `{ jobId, startedAt, snapshot, previewUrl?, failureReason? }`.
- `snapshot.currentStage` ∈ `pending_payment|pending|researching|designing|generating_assets|building|testing|deploying|ready|failed`.
- **Ready** ⇒ `currentStage === 'ready'` (+ `previewUrl`). **Failed** ⇒ `currentStage === 'failed'` (+ `failureReason`).
- ⚠️ **Trap:** do **NOT** send `?startedAt=…` — if the job isn't found it falls back to a time-based
  demo animation that returns a canned `logistics-demo.cyberg7.com.sg` preview. The funnel polls
  **without** `startedAt` so it always uses the real Airtable-backed status.

---

## 7. Branches & key commits (`intake-launchnow`)
| Branch | Commit | State |
|---|---|---|
| `main` (production) | `d8cdee8` | **LIVE.** Funnel HTML with `BACKEND_READY=true` + repointed CTAs. Internal docs deliberately excluded. |
| `claude/busy-curie-pjEii` (dev) | — | Full work history + internal docs (`PLAN.md`, `PLAN-REVIEW-LOG.md`, this `handoff.md`). `BACKEND_READY=false`. |
| `claude/busy-curie-pjEii-go-live` | `15779fe` | The flip-to-`true` staging branch used to ship. **Now redundant** — safe to delete. |

> ⚠️ **Branch drift to fix:** `main` is `true` (live) but the dev branch is `false`. Before the next
> ship from dev, set dev's `BACKEND_READY=true` (or it will re-gate the live funnel). The simplest
> long-term state: dev == live; retire the `…-go-live` branch.

---

## 8. Verification evidence
**CORS (independently re-checked against live `agency.cyberg7.com.sg`):**
- `OPTIONS /api/intake` → `204` + `access-control-allow-origin: https://intake-launchnow.cyberg7.com.sg` + methods + `Content-Type`.
- `POST /api/intake` (empty) → `400` + ACAO present.
- `GET /api/status/…` → `200` + ACAO present.
- `Origin: https://evil.example` → `204` with **no** `access-control-*` (scoped, not `*`).

**End-to-end (real browser submit on the live site):**
- Job `4bfb2caf-088d-407b-978d-434533e83152`; 7 stages all `done` in ~3.4 min (Airtable-backed, not the demo).
- Preview `https://pub-echo-preview.cyberg7.com.sg` → HTTP `200`, `<title>Pub Echo — Pub</title>`, ~87 KB.

**Production content checks:** `/intake` 404→200 serving the real form + `BACKEND_READY=true`; homepage
14 CTAs → `/intake`, 0 old-host; `/status` live; `/PLAN.md`, `/handoff.md` → `404` (internal docs not exposed).

---

## 9. Security & privacy (carried forward)
- **No secrets committed.** Only public client IDs (Google Ads `AW-18194187704`) are embedded, matching the rest of the site.
- **PDPA consent** is a required checkbox before submit; disclosure links `/privacy`, `/terms`, `/pdpa`.
- **`jobId` is a UUIDv4** (opaque/unguessable). `/status` renders only display-safe backend fields.
- **`checkoutUrl` open-redirect guard:** the submit handler only follows a returned URL if it's `https`
  on an allow-listed host (`checkout.stripe.com`, `agency.cyberg7.com.sg`).
- **CORS** is allow-listed to exactly the production origin (not `*`).
- **Internal planning docs are kept off `main`** so they aren't publicly fetchable.

---

## 10. How it was decided (provenance)
The plan was hardened before any code: an interactive **grill** (Approach C, vanilla HTML+JS, direct
CORS, free-preview-first, scope = `/intake`+`/status` only), then a **cross-model adversarial review
with OpenAI Codex** (2 rounds: REVISE → APPROVED) that added Gate-0 (contract-first), the polling policy,
the `checkoutUrl` guard, opaque-token/PDPA requirements, and the abuse caveat. Full record:
`PLAN.md` (the locked plan) + `PLAN-REVIEW-LOG.md` (the round-by-round argument) on the dev branch.

---

## 11. Open fast-follows (NOT done — recommended next)
1. 🔴 **Abuse / cost protection (priority).** Every submit triggers a real, paid AI generation + a live
   subdomain; the only current guard is the client honeypot (bots bypass it). Add **Cloudflare Turnstile**
   on `/intake` (frontend widget + backend token verification) and/or backend rate-limiting. Flagged by Codex.
2. **Google Ads conversion label** — the `generate_lead` gtag event fires but isn't mapped to a conversion.
3. **Meta Pixel** — NOT present in the repo; adding `1961722217792735` is a *new tracker* → needs PDPA
   disclosure + owner go-ahead (do not "mirror" silently).
4. **Logo upload** — the backend supports an optional logo (`/api/upload-logo`, multipart) that seeds brand
   colours; omitted from the native form for now.
5. **Canonical-host cleanup** — site-wide `canonical`/`og`/JSON-LD are inconsistent (`cyberg7.com` vs
   `launchnow.cyberg7.com.sg`; `terms.html` has a `//terms` double slash). Pick one host or explicitly defer.
6. **Branch reconcile** — set dev `BACKEND_READY=true`, delete `…-go-live` (see §7).

---

## 12. How to operate / common tasks
- **Temporarily gate the funnel** (e.g., backend maintenance): set `BACKEND_READY=false` in `intake.html`
  + `status.html`, ship to `main`. The form then shows the "coming online" message and calls nothing.
- **Schema change on the backend:** update the form fields in `intake.html` and the `payload` object in
  its submit script to match the new `intakeSchema`; update stage labels in `status.html` if stages change.
- **Quick live health check (no browser):**
  ```bash
  curl -s -o /dev/null -w "%{http_code}\n" https://intake-launchnow.cyberg7.com.sg/intake   # 200
  curl -s -i -X OPTIONS -H "Origin: https://intake-launchnow.cyberg7.com.sg" \
    -H "Access-Control-Request-Method: POST" https://agency.cyberg7.com.sg/api/intake | grep -i access-control-allow-origin
  ```
- **Inspect a job:** `curl -s https://agency.cyberg7.com.sg/api/status/<jobId>` (no `?startedAt`).
- **Deploys are automatic:** push to `intake-launchnow` `main` → Vercel production; push to `website-factory`
  `main` → agency production.

---

## 13. Key references
- `PLAN.md` — the locked implementation plan (dev branch).
- `PLAN-REVIEW-LOG.md` — grill decisions + the full Codex review transcript (dev branch).
- `website-factory`: `src/components/sections/IntakeForm.tsx`, `src/lib/intake-schema.ts`,
  `src/app/api/intake/route.ts`, `src/app/api/status/[jobId]/route.ts`, `src/lib/job-status.ts`,
  `src/lib/cors.ts` (the CORS patch).
