# LaunchNow Intake Funnel — Project Handover

> **Purpose:** Wire the **"Get Started" button** on `https://intake-launchnow.cyberg7.com.sg/`
> to a **native `/intake` form** on that same site, which generates an AI website preview
> by **reusing the existing preview-generation backend** (the website-factory engine).
>
> **Last updated:** 2026-06-05 · Generated at handoff so a new engineer (or a Claude Code
> agent loading this file) can continue exactly where we left off.

---

## 0. TL;DR — read this first (for the next Claude Code agent)

**The goal in one sentence:** `intake-launchnow.cyberg7.com.sg/intake` must be a page **owned by
the intake-launchnow site itself** (LaunchNow-branded form), that creates a preview website by
calling the shared backend. It must **NOT** be a redirect/proxy that runs the agency's
`/launchnow/intake` page.

**State in 3 bullets:**
1. ✅ The static marketing site is live. All "Get Started" CTAs were repointed to a relative
   `/intake` on branch **`Connect-intake-form`** (not yet on `main`).
2. ⚠️ The `/intake` route does **not exist yet** as a native page. The committed branch currently
   fakes it with a **reverse-proxy to agency** (`vercel.json` rewrites) — **the owner rejected this
   approach.** See [§6](#6-important-the-committed-approach-was-rejected).
3. ⛔ The backend funnel pages live in a **different** repo (`website-factory`) on an **unmerged
   branch**, so `agency.cyberg7.com.sg/launchnow/intake` currently returns **404**.

**The ONE decision to make before coding:** *How should `/intake` be hosted natively on
intake-launchnow while reusing the backend?* → See [§7 Approach options](#7-approach-options--decision-required).
Recommended: **Approach C (native form page in this repo + shared backend API).**

**Where to start:** [§10 Start-here runbook](#10-start-here-runbook).

---

## 1. Project coordinates

### System A — `intake-launchnow` (the site you're handing over)
| | |
|---|---|
| **Live URL** | https://intake-launchnow.cyberg7.com.sg/ |
| **Vercel project** | https://vercel.com/cyberg7/intake-launchnow |
| **Vercel project ID** | `prj_fi75XePjHjG7MzkTBmSpYmY7ayhA` (team `cyberg7` / `team_KozfH4AcuxTB6hg44enCxv2e`) |
| **GitHub** | https://github.com/Cyberg7tech/intake-launchnow.git |
| **Branches** | `main` (production, auto-deploys) · **`Connect-intake-form`** ← the 2nd branch, all the work-in-progress config |
| **Local clone** | `C:\Users\cyber\Downloads\intake-launchnow\` |
| **Stack** | **Pure static HTML** — no framework, no `package.json`, no build step. Vercel framework = `null`. Deploys files as-is. |
| **Pages** | `index.html`, `about.html`, `contact.html`, `pdpa.html`, `terms.html`, `privacy.html` + `vercel.json` |

### System B — `website-factory` / `cyberg7-agency` (the backend engine being reused)
| | |
|---|---|
| **Live URL** | https://agency.cyberg7.com.sg/ |
| **Vercel project** | `cyberg7-agency` · ID `prj_SY3U2qQYeZuQ88RxUy2eBJ3slOX8` (same team) |
| **GitHub** | `Cyberg7tech/website-factory` |
| **Funnel branch** | **`LaunchNow-HTML-connect`** (holds the LaunchNow funnel + tenant-aware backend — **not merged to `main` yet**) |
| **Local clone** | `C:\Users\cyber\Downloads\Claude MCP\website-factory\` |
| **Stack** | Next.js 16 (App Router) · React 19 · Tailwind v4 · Zod 4 |

> These are **two separate Vercel projects on two separate domains.** That separation is the whole
> reason this task is non-trivial: the form + backend live in System B, but they must *appear and run*
> on System A.

---

## 2. What we're building (the goal, precisely)

```
Visitor on  intake-launchnow.cyberg7.com.sg
   │  clicks "Get Started"
   ▼
intake-launchnow.cyberg7.com.sg/intake     ← a NATIVE page of THIS site (to build)
   │  fills LaunchNow-branded intake form, submits
   ▼
Shared preview-generation backend          ← REUSE website-factory's engine (do not rebuild)
   │  kicks off async website generation
   ▼
intake-launchnow.cyberg7.com.sg/status/<jobId>   ← progress page (native or shared)
   │  when ready →
   ▼
<slug>-preview.cyberg7.com.sg              ← the generated preview site (already host-agnostic)
```

**Hard requirement from the owner:** the `/intake` page must be served/executed by the
intake-launchnow project — **not** be a transparent proxy that runs `agency.cyberg7.com.sg/launchnow/intake`.

---

## 3. How the preview-generation pipeline works (backend, System B)

This already exists in `website-factory`. You are **reusing** it, not changing how it works.

1. **`POST /api/intake`** — accepts the form payload. It reads a **`tenant`** field **from the request
   body** (values: `agency` | `launchnow`; blank → `agency`). It is **not** keyed off the Host header,
   which is exactly why a form on *any* domain can drive it as long as it sends `tenant: "launchnow"`.
2. The route validates (Zod), writes a job row to **Airtable** (incl. the `tenant`), and triggers the
   **n8n factory orchestrator** webhook to start async generation.
3. Depending on tenant config it may return a **`jobId` / status URL**, and possibly a Stripe
   **`checkoutUrl`** (deposit) — see the open decision in [§9](#9-open-decisions-flag-to-owner).
4. A **status page** polls job progress until the preview is ready.
5. The finished preview is served at **`<slug>-preview.cyberg7.com.sg`** via a wildcard DNS record
   (`*.cyberg7.com.sg → cname.vercel-dns.com`) + middleware in `website-factory/src/proxy.ts`. This
   output is **independent of which host ran the intake** — no work needed here.

### ⭐ Important pre-existing code you should know about
`website-factory/src/proxy.ts` (Next.js middleware) **already contains a block for our exact host**:

```ts
const LAUNCHNOW_INTAKE_HOST = 'intake-launchnow.cyberg7.com.sg'
// if host === LAUNCHNOW_INTAKE_HOST:
//   '/' or '/intake'      → rewrite to /launchnow/intake   (the form)
//   '/status/<id>'        → rewrite to /launchnow/status/<id>
```

This means: **if the `intake-launchnow.cyberg7.com.sg` domain were attached to the `website-factory`
Vercel project, the funnel would already render natively under the intake-launchnow host** (same origin,
URL stays `intake-launchnow...`, zero duplication). That is "Approach B" below. The catch is that the
domain currently points at the *static* project, and the owner wants intake-launchnow to stay its own
repo — so this block is presently **dormant**. Keep it in mind; it may be the fastest path if the owner
relaxes the "separate repo" constraint.

---

## 4. Files & where things live

### In THIS repo (`intake-launchnow`) — what changed on `Connect-intake-form`
`git diff --stat main..Connect-intake-form`:
```
 about.html   | 2 +-     # CTA repointed
 contact.html | 2 +-     # CTA repointed
 index.html   | 28 +-    # 14 CTAs repointed
 pdpa.html    | 2 +-     # CTA repointed
 privacy.html | 2 +-     # CTA repointed
 terms.html   | 2 +-     # CTA repointed
 vercel.json  | 7 +++    # ADDED reverse-proxy rewrites  ← the rejected bit
```
- **19 CTAs** changed from `https://launchnow.cyberg7.com.sg/intake` → relative **`/intake`**
  (same-host). ✅ This part is correct and reusable under any approach.
- **`vercel.json`** gained a `rewrites` array proxying `/intake`, `/status/:jobId`, `/launchnow/*`,
  `/api/*`, `/_next/*` to `agency.cyberg7.com.sg`. ⚠️ **This is the rejected proxy approach — likely to
  be removed/replaced.** Current full file:
  ```json
  {
    "$schema": "https://openapi.vercel.sh/vercel.json",
    "cleanUrls": true,
    "trailingSlash": false,
    "redirects": [{ "source": "/landing", "destination": "/", "permanent": true }],
    "rewrites": [
      { "source": "/intake", "destination": "https://agency.cyberg7.com.sg/launchnow/intake" },
      { "source": "/status/:jobId", "destination": "https://agency.cyberg7.com.sg/launchnow/status/:jobId" },
      { "source": "/launchnow/:path*", "destination": "https://agency.cyberg7.com.sg/launchnow/:path*" },
      { "source": "/api/:path*", "destination": "https://agency.cyberg7.com.sg/api/:path*" },
      { "source": "/_next/:path*", "destination": "https://agency.cyberg7.com.sg/_next/:path*" }
    ]
  }
  ```

### In the backend repo (`website-factory`, branch `LaunchNow-HTML-connect`) — files to read
| File | Why it matters |
|---|---|
| `src/components/sections/IntakeForm.tsx` | The actual React form component (the thing to re-host / port) |
| `src/app/launchnow/intake/page.tsx` | Renders IntakeForm for `tenant="launchnow"` |
| `src/app/launchnow/status/[jobId]/page.tsx` | The progress/status page |
| `src/app/api/intake/route.ts` | The `POST /api/intake` backend (tenant-aware) |
| `src/lib/brands.ts` | Brand/tenant registry (LaunchNow vs agency branding) |
| `src/proxy.ts` | Middleware incl. the dormant `LAUNCHNOW_INTAKE_HOST` block (see §3) |
| (search `src/app/api` for a status endpoint) | What the status page polls |

> ⚠️ **Verify before porting** (the previous session was stopped before it could read these in full):
> the exact npm dependencies of `IntakeForm.tsx`, its Next-specific couplings (`useRouter`, etc.),
> whether `/api/intake` sets **CORS** headers, the **status API path**, and whether it returns a Stripe
> `checkoutUrl` for `launchnow`. These four answers determine how much porting Approach C needs.

---

## 5. Current deploy state

| Item | State |
|---|---|
| intake-launchnow **production** (`main` `f7b0aab`) | Untouched. CTAs still point to old `https://launchnow.cyberg7.com.sg/intake` (a live page on `sites.ludicrous.cloud`). |
| intake-launchnow **preview** (`Connect-intake-form` `49b83e9`) | Built & READY: `https://intake-launchnow-git-connect-intake-form-cyberg7.vercel.app` |
| `agency.cyberg7.com.sg/launchnow/intake` | **404** (funnel branch not merged) |
| `agency.cyberg7.com.sg/api/intake` | **400** (base route exists, rejects empty body — expected) |
| `LaunchNow-HTML-connect` (website-factory) | **Not merged** to agency `main`. Owner reserved this merge. |

---

## 6. ⚠️ IMPORTANT: the committed approach was rejected

The `Connect-intake-form` branch implements **Approach A (reverse-proxy to agency)**: `/intake` on
intake-launchnow transparently rewrites to `agency.cyberg7.com.sg/launchnow/intake`. The URL bar stays
on intake-launchnow, but the page **executes on the agency deployment.**

**The owner explicitly rejected this:** *"I don't want this to execute under
agency.cyberg7.com.sg/launchnow/intake. That's not my intention at all."*

➡️ **Do not merge `Connect-intake-form` to `main` as-is.** The CTA edits (relative `/intake`) are good
to keep; the `vercel.json` proxy `rewrites` are the part to reconsider/replace per the chosen approach.

---

## 7. Approach options — DECISION REQUIRED

| | Approach | What it means | Keeps separate repo? | Native execution? | Effort | Backend reuse |
|---|---|---|---|---|---|---|
| A | **Reverse-proxy** *(committed, REJECTED)* | `vercel.json` rewrites `/intake`→agency page | ✅ | ❌ runs on agency | ✅ done | shared |
| B | **Attach domain to website-factory** | Point `intake-launchnow.cyberg7.com.sg` at the `cyberg7-agency` project; the dormant `LAUNCHNOW_INTAKE_HOST` middleware serves it natively | ❌ marketing must move into website-factory | ✅ truly native | low *(mostly pre-built)* | shared |
| **C** | **Native form page in THIS repo + shared backend API** ⭐ | Build a real `/intake` (and `/status`) page in intake-launchnow (small Vite/React bundle or self-contained HTML+JS matching the marketing design) that **POSTs to the shared `/api/intake`**. Page lives & runs on intake-launchnow; only data calls hit the backend. | ✅ | ✅ truly native | medium | shared (via CORS or a narrow `/api`-only proxy) |
| D | **Convert intake-launchnow to a Next.js app** | Rebuild this repo as Next.js, import the funnel pages, proxy or port the API | ✅ | ✅ | high | shared or duplicated |

**Recommendation: Approach C.** It is the only option that satisfies **all** stated constraints at once
— keep intake-launchnow as its own repo, `/intake` genuinely owned by this site, and reuse the existing
backend without rebuilding it. Sub-decision for C: how the native page reaches the backend without CORS
pain — either (1) enable CORS on `/api/intake` for the intake-launchnow origin, or (2) add a **narrow**
`vercel.json` rewrite for **`/api/*` only** (the *page* stays native; only JSON calls proxy). Confirm
which with the owner — option (2) is invisible and minimal, but still introduces an agency dependency for
data calls.

> If the owner is willing to drop the "separate repo" requirement, **Approach B is the fastest** because
> `src/proxy.ts` already handles the intake-launchnow host (it would just need the root `/` to serve the
> marketing homepage instead of the form).

---

## 8. Environment variables (NAMES only — never commit values)

> 🔐 All secret **values** live in **Vercel project settings / Doppler**, never in the repo.
> In the recommended **Approach C, intake-launchnow needs NO backend secrets** (the backend stays in
> website-factory). intake-launchnow only carries **public** client IDs already embedded in the HTML.

**Backend (`website-factory`) — server-side secrets:**
`N8N_FACTORY_WEBHOOK_URL`, `ORCHESTRATOR_SECRET`, `INTAKE_SHARED_SECRET`, `AIRTABLE_PAT`,
`AIRTABLE_BASE_ID`, `STRIPE_SECRET_KEY`, `STRIPE_DEPOSIT_CENTS`, `STRIPE_PREMIUM_CENTS`,
`EMAIL_FROM`, `OPERATOR_EMAIL`, `AGENCY_BASE_URL`, (Resend API key).

**Analytics (server-side):** `GA4_MEASUREMENT_ID`, `GA4_MP_API_SECRET`. **(public)** `NEXT_PUBLIC_GTM_ID`.

**Public client-side IDs (safe to embed, already in the static HTML):**
Google Ads `AW-18194187704`, Meta Pixel `1961722217792735`.

> ⚠️ The **value** of `N8N_FACTORY_WEBHOOK_URL` is a secret (it's the orchestrator webhook URL) — treat
> it as a secret, never print it. The contact-form webhook is "public by design." Never echo any secret
> value into chat, code, or commits.

---

## 9. Prerequisites & gates (ordering matters)

1. **🟥 Airtable prereq (do this FIRST, before merging the backend):** the Airtable **`jobs`** table
   must have a **`tenant` single-select** column (options `agency`, `launchnow`; blank → `agency`).
   *Owner confirmed this is present.* If it's missing, `/api/intake` **fails closed (503) for ALL
   tenants — including the live agency site.**
2. **Gate 1 — backend merge:** merge `website-factory` `LaunchNow-HTML-connect` → agency `main`.
   This deploys the tenant-aware `/api/intake` + the `/launchnow/*` pages and flips
   `agency.cyberg7.com.sg/launchnow/intake` from 404 → 200. **Owner reserved this merge.**
   *Caveats:* agency `main` has diverged (run `git fetch` first); the website-factory tree has **3
   pre-existing uncommitted files** (`src/lib/brands.ts` + 2 tests) — handle deliberately, don't blind-commit.
3. **Gate 2 — frontend:** build the native `/intake` per the chosen approach, then deploy intake-launchnow.
   **Do Gate 1 before any approach that depends on the backend being live**, or `/intake` will 404 in the gap.

---

## 10. Start-here runbook

For the engineer (or Claude Code agent) picking this up:

1. **Read this file**, then read the [§4 backend files](#4-files--where-things-live) from branch
   `LaunchNow-HTML-connect`:
   ```bash
   cd "C:\Users\cyber\Downloads\Claude MCP\website-factory"
   git show LaunchNow-HTML-connect:src/components/sections/IntakeForm.tsx
   git show LaunchNow-HTML-connect:src/app/api/intake/route.ts
   git show LaunchNow-HTML-connect:src/lib/brands.ts
   ```
2. **Answer the 4 verification questions** from [§4](#4-files--where-things-live) (form deps/couplings,
   CORS on `/api/intake`, status API path, Stripe `checkoutUrl` for launchnow).
3. **Confirm the architecture decision** ([§7](#7-approach-options--decision-required)) with the owner —
   default to **Approach C**.
4. **Confirm the Airtable `tenant` column** exists ([§9](#9-prerequisites--gates-ordering-matters)).
5. **Gate 1:** owner merges the backend branch (or authorizes you to). Verify with the [§11 checks](#11-verification-commands).
6. **Gate 2:** implement the chosen approach in this repo on `Connect-intake-form` (or a fresh branch):
   - Approach C: add `/intake` + `/status` pages (LaunchNow-branded, matching the marketing design),
     point the form at the shared backend, replace the broad proxy `rewrites` with a narrow `/api`-only
     rewrite *or* rely on CORS. Keep the relative-`/intake` CTA edits.
7. **Deploy & verify** end-to-end ([§11](#11-verification-commands)). Then merge to `main`.

---

## 11. Verification commands

```bash
# Gate 1 — is the backend funnel live? (expect 200 after the merge; currently 404)
curl -s -o /dev/null -w "%{http_code}\n" https://agency.cyberg7.com.sg/launchnow/intake
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://agency.cyberg7.com.sg/api/intake   # 400 = route exists

# This repo — git state
cd "C:\Users\cyber\Downloads\intake-launchnow"
git log --oneline -2 Connect-intake-form
git diff --stat main..Connect-intake-form

# End-to-end (after go-live): the form page must be served BY intake-launchnow (Approach C/B),
# not 3xx/proxied to an agency URL. Then submit a test intake and confirm a <slug>-preview.cyberg7.com.sg
# preview is produced and the status page advances.
```

**Definition of done:** clicking "Get Started" on `intake-launchnow.cyberg7.com.sg` lands on
`intake-launchnow.cyberg7.com.sg/intake` (a page owned by this site), the form submits to the shared
backend, a preview site is generated, and the status page tracks it — with **no dependency on the
agency `/launchnow/intake` page** for rendering the form.

---

## 12. Open decisions to flag to the owner

- **Stripe timing:** if `/api/intake` returns a `checkoutUrl` for `tenant=launchnow`, the flow becomes
  *Get Started → form → Stripe deposit → preview.* Decide **deposit-first vs. free-preview-first**
  (`IntakeForm.tsx` redirects to `checkoutUrl` if present). This is a product call.
- **Backend reach for the native page (Approach C):** CORS on `/api/intake` vs. a narrow `/api`-only
  `vercel.json` rewrite. (The broad page/_next/launchnow rewrites should be removed.)
- **Stale tags (low priority):** `terms.html` and `pdpa.html` still have `canonical`/`og:url` pointing at
  the old `launchnow.cyberg7.com.sg` host, and `terms.html` has a `//terms` double-slash. Update to
  `intake-launchnow.cyberg7.com.sg`.

---

## 13. Security notes (carry forward)

- Never print or commit secret **values**; reference env vars by **name** only. Never commit `.env`.
- `N8N_FACTORY_WEBHOOK_URL`'s value is a secret. Public client IDs (Google Ads / Meta Pixel) are fine.
- Never force-push to `main`; create new commits (don't `--amend` after a failed hook); don't skip hooks.
- Set analytics/secret env in **Vercel/Doppler**, not in chat or code.
```
