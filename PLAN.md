# Plan: Native LaunchNow `/intake` + `/status` funnel on intake-launchnow

_Locked via grill — by Claude + owner (aptintel.io). 2026-06-05. **Rev 2** (post Codex Round 1)._

## Goal
Build a native, LaunchNow-branded `/intake` form page (and a `/status` progress page) **inside the
`intake-launchnow` repo**, so clicking "Get Started" on `intake-launchnow.cyberg7.com.sg` lands on a page
**owned and served by this site** — visually identical to the marketing pages — that submits the brief
directly to the shared backend (`agency.cyberg7.com.sg/api/intake`, `tenant=launchnow`) and tracks
generation to a free preview at `<slug>-preview.cyberg7.com.sg`. No agency page-proxy. Payment is
post-preview and out of scope here.

## Gates & external dependencies (ordered — clear before the dependent step)
- **Gate 0 — backend contract in hand (BLOCKS form impl).** Before writing the form/submit/status wiring,
  obtain from `website-factory@LaunchNow-HTML-connect` and pin here: the exact `POST /api/intake` request
  schema + field names + example payload (incl. the `tenant` key name/value); the **status endpoint URL +
  response schema**; whether a `checkoutUrl` is returned for `launchnow`; the CORS posture. The page
  *shell/chrome* (steps 1, 5–7) may proceed before Gate 0; the *form fields + wire-up* (steps 2–4) may not.
- **Gate 1 — backend live (BLOCKS prod release).** Owner merges the tenant-aware funnel/API to agency
  `main` (`/launchnow/intake` 404→200; tenant-aware `/api/intake`). Prereq: Airtable `jobs.tenant`
  single-select exists, else `/api/intake` fails closed (503) for ALL tenants.
- **Backend acceptance criteria (owner sign-off before merge to `main`)** — NOT edited from this repo, but
  MUST hold for the frontend to function safely:
  (a) `/api/intake` sends CORS `Access-Control-Allow-Origin: https://intake-launchnow.cyberg7.com.sg` (+ preflight);
  (b) the job/status identifier is an **opaque, unguessable token** (not sequential/guessable);
  (c) the status endpoint returns only **minimal, display-safe** fields (no raw brief / PII / internal URLs);
  (d) **server-side rate limiting + per-tenant quotas** on `/api/intake` (free generation is abusable);
  (e) any returned `checkoutUrl` is an **allow-listed HTTPS** origin.

## Approach
1. **Page shell (vanilla, reuse this site's chrome).** Add `intake.html` + `status.html` (served at
   `/intake`, `/status` via `cleanUrls`) cloning `<head>`/fonts/CSS vars (dark `#0a0b10` / purple
   `#6c5ce7`, Outfit + Space Mono)/nav/footer from `contact.html`/`index.html`. No build step. *(pre-Gate-0 OK)*
2. **Intake form.** Model on `contact.html`: `.form-group/.form-input/.form-status`, intl-tel-input,
   `.shiny-cta`, off-screen honeypot. **Fields ported from the Gate-0 contract (not guessed).** Add a
   required **PDPA consent** checkbox + short disclosure (brief is processed by the backend to generate &
   publish a preview; links to `/pdpa` + `/privacy`) before submit. Payload always includes
   `tenant:"launchnow"`. *(needs Gate 0)*
3. **Submit handler (direct CORS).** honeypot → `checkValidity()` + phone + consent →
   `fetch('https://agency.cyberg7.com.sg/api/intake', {POST, JSON, tenant:"launchnow"})`. On success read
   the backend's **opaque status token** → go to `/status?t=<token>`. **`checkoutUrl` open-redirect guard:**
   only redirect if it parses as `https:` AND host ∈ allow-list (`checkout.stripe.com`, agency host); else
   ignore + show support fallback. *(needs Gate 0)*
4. **Status page.** Reads the opaque token; polls the status endpoint on a defined policy: start ~2s,
   **exponential backoff + jitter** (cap ~15s), **pause while `document.hidden`**, honor `Retry-After`,
   hard **max duration** (~10 min) → retry + support fallback. Render only display-safe fields. When ready,
   surface `<slug>-preview.cyberg7.com.sg` (button + auto-advance). `noindex`. *(needs Gate 0)*
5. **CTAs.** Repoint by **exact-href inventory** — all **19** occurrences of
   `href="https://launchnow.cyberg7.com.sg/intake"` (verified: index 14; about/contact/pdpa/privacy/terms 1
   each) → `/intake`. Leave the intentional `/contact` ("Talk to Our Team") links. **Drop** the
   `Connect-intake-form` `vercel.json` proxy rewrites entirely. *(pre-Gate-0 OK)*
6. **Analytics (existing only; new trackers gated).** Mirror the **gtag** actually present (Google Ads
   `AW-18194187704`) + fire a lead event on submit. **Meta Pixel is NOT in the repo** (verified — no `fbq`);
   adding `1961722217792735` is a NEW tracker → owner go-ahead + PDPA/privacy disclosure + PII-free events.
   Google Ads conversion *label* = owner-supplied. *(pre-Gate-0 OK, minus label)*
7. **SEO/meta + canonical-host DECISION.** Correct `canonical`/`og` for `/intake` on the chosen host;
   `noindex` `/status`. The site's canonical host is inconsistent today: `index/about/contact/privacy` →
   `cyberg7.com`; `pdpa/terms` → `launchnow.cyberg7.com.sg` (and `terms` has a `//terms` double slash); the
   live host is `intake-launchnow.cyberg7.com.sg`. **Pick ONE canonical host** and either fix
   canonical/OG/JSON-LD consistently site-wide, or **explicitly defer all non-funnel SEO** and only set
   correct tags on `/intake`. *(pre-Gate-0 OK)*
8. **Deploy & verify** (after Gate 1): Get Started → `/intake` (served here) → consent+submit → job
   (Airtable `tenant=launchnow`) → `/status` advances under the polling policy → `<slug>-preview...`
   renders. Then merge `claude/busy-curie-pjEii` → `main`.

## Key decisions & tradeoffs
- **Vanilla HTML+JS, not React/Next (Q1).** Page must match this exact site; reusing its own chrome
  guarantees the match with zero build step. Tradeoff: porting a complex React `IntakeForm.tsx` to vanilla
  is manual if it carries heavy dynamic logic (resolved at Gate 0).
- **Direct browser→backend via CORS, not a proxy (Q2).** Mirrors the contact-form pattern; keeps
  `vercel.json` free of any agency reference. Tradeoff: backend must allow our origin (acceptance crit a).
- **Free preview first (Q3).** Lowest friction; pay-to-publish after. Tradeoff: every submit triggers a
  free generation — abuse/cost vector handled in Security (not by honeypot alone).
- **Scope = /intake + /status only (Q4).** Payment + the generated preview are the backend's domain.

## Security & privacy requirements
- **Free generation is abusable** — honeypot + CORS do NOT stop scripted POSTs. Keep the honeypot (cheap),
  but real protection is **backend rate limiting + per-tenant quotas** (acceptance crit d). If abuse/cost
  is a launch concern, add a **bot challenge (Cloudflare Turnstile / hCaptcha)** on `/intake` (owner call;
  minor friction).
- **Opaque/unguessable status token** + **minimal, display-safe** status response (acceptance crit b, c);
  the frontend renders no raw brief / PII / internal URLs.
- **Open-redirect guard** on any `checkoutUrl` (allow-list scheme + host).
- **PDPA consent** captured before submit; disclosure covers backend processing, retention, preview
  publication; analytics events carry **no PII**.

## Risks / open questions (owner)
- Gate 0 contract still unread (website-factory out of GitHub scope; add-repo tool absent this session).
- Canonical-host decision (step 7).
- Bot challenge (Turnstile) at launch, or rely on backend limits?
- Meta Pixel: add the new tracker (with PDPA update) or skip?
- Google Ads conversion label.

## Out of scope (this repo)
- Stripe/checkout UI + "pay to publish" (backend/preview-side).
- The generated preview site at `<slug>-preview.cyberg7.com.sg` (backend-produced, host-agnostic).
- Editing `website-factory` (CORS entry, tenant logic, status API, rate limiting) — these are **external
  dependencies with acceptance criteria** (see Gates), coordinated with the owner, not done from here.
- Merging the backend branch / Airtable schema (owner-reserved).
