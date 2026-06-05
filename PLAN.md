# Plan: Native LaunchNow `/intake` + `/status` funnel on intake-launchnow

_Locked via grill — by Claude + owner (aptintel.io). 2026-06-05._

## Goal
Build a native, LaunchNow-branded `/intake` form page (and a `/status` progress page) **inside the
`intake-launchnow` repo**, so clicking "Get Started" on `intake-launchnow.cyberg7.com.sg` lands on a
page **owned and served by this site** — visually identical to the existing marketing pages — that
submits the brief directly to the shared website-factory backend
(`agency.cyberg7.com.sg/api/intake`, `tenant=launchnow`) and tracks generation to a free preview at
`<slug>-preview.cyberg7.com.sg`. No agency page-proxy. Payment is post-preview and out of scope here.

## Approach
1. **Page shell (vanilla, reuse this site's chrome).** Add `intake.html` + `status.html` (served at
   `/intake`, `/status` via existing `cleanUrls`) by cloning the `<head>`, Google Fonts (Outfit +
   Space Mono), CSS variables (dark `#0a0b10` / purple `#6c5ce7`), nav, and footer from
   `contact.html`/`index.html`. No build step, no new toolchain.
2. **Intake form.** Model on `contact.html`'s pattern: `.form-group/.form-input/.form-status` markup,
   intl-tel-input phone (SG default), `.shiny-cta` submit, off-screen honeypot (`company_website`).
   Final field set is ported from the backend `IntakeForm.tsx` (BLOCKED until website-factory access —
   see Risks). Payload **always includes `tenant: "launchnow"`**.
3. **Submit handler (direct CORS).** honeypot check → `checkValidity()` + phone validation →
   `fetch('https://agency.cyberg7.com.sg/api/intake', { POST, JSON, tenant:"launchnow" })`. On success,
   read the returned `jobId`/status URL → navigate to `/status?job=<jobId>`. Defensive: if the response
   carries a `checkoutUrl`, follow it (covers a future deposit-first switch); default expects a `jobId`.
4. **Status page.** `/status` reads the jobId, polls the backend status endpoint (CORS) on an interval,
   renders progress in the site's style, and when ready surfaces `<slug>-preview.cyberg7.com.sg`
   (button + auto-advance). Failure/timeout → retry + support-email fallback (mirrors contact UX).
   `noindex` (transient page).
5. **CTAs.** Port the relative-`/intake` CTA edits from `Connect-intake-form` (19 "Get Started" links →
   `/intake`). **Drop** that branch's `vercel.json` proxy `rewrites` entirely — under CORS, none needed.
6. **Analytics.** Mirror gtag (Google Ads `AW-18194187704`) + Meta Pixel (`1961722217792735`); fire a
   lead/conversion event on successful submit (like contact's `generate_lead`), optionally a second on
   preview-ready. Exact Google Ads conversion *label* = owner to supply.
7. **SEO/meta.** Proper `canonical`/`og` for `/intake` on the `intake-launchnow.cyberg7.com.sg` host.
   Opportunistic cleanup: fix stale `canonical`/`og:url` on `terms.html`/`pdpa.html` (point at the old
   `launchnow.cyberg7.com.sg`; `terms` also has a `//terms` double slash). Low priority, off critical path.
8. **Deploy & verify** (after Gate 1): Get Started → `/intake` (served by this site) → submit → job
   created (Airtable `tenant=launchnow`) → `/status` advances → `<slug>-preview.cyberg7.com.sg` renders.
   Then merge `claude/busy-curie-pjEii` → `main`.

## Key decisions & tradeoffs
- **Vanilla HTML+JS, not React/Next (Q1).** Page must match this exact site; reusing its own chrome
  guarantees the match with zero build step. Tradeoff: porting a complex React `IntakeForm.tsx` to
  vanilla is manual if it carries heavy dynamic logic (unknown until we can read it).
- **Direct browser→backend via CORS, not a proxy (Q2).** Mirrors the existing contact-form pattern and
  keeps `vercel.json` free of any agency reference. Tradeoff: backend must add
  `intake-launchnow.cyberg7.com.sg` to the `/api/intake` CORS allow-list (bundled into Gate 1); agency
  origin is visible in the browser (already true for the contact webhook).
- **Free preview first (Q3).** Lowest friction for paid traffic; pay-to-publish after the preview.
  Tradeoff: every submit triggers a free generation (compute/abuse vector) — mitigated by honeypot +
  CORS allow-list; meaningful rate-limiting must live backend-side.
- **Scope = /intake + /status only (Q4).** Payment + the generated preview are the backend's domain.
  No Stripe code in this repo.

## Risks / open questions
- **BLOCKED on website-factory access.** Can't read `IntakeForm.tsx`, `app/api/intake/route.ts`,
  `lib/brands.ts` yet (repo out of GitHub scope; add-repo tool absent this session). Must confirm before
  finishing the build: (a) exact form fields; (b) exact `/api/intake` request keys + the `tenant` field
  name/value; (c) the **status endpoint path + response shape** `/status` polls; (d) whether CORS is set
  or needs adding for our origin; (e) whether a `checkoutUrl` is returned for `launchnow`.
- **Gate 1 (owner-reserved): backend not live.** `agency.cyberg7.com.sg/launchnow/intake` = 404 today;
  tenant-aware `/api/intake` is on the unmerged `LaunchNow-HTML-connect` branch. Per the chosen sequence
  ("backend first"), `/intake` releases to prod only after Gate 1. Airtable `jobs.tenant` single-select
  must exist before that merge or `/api/intake` fails closed (503) for ALL tenants.
- **Google Ads conversion label** unknown — event hooks scaffolded; owner supplies the label.
- **Pre-existing SEO inconsistency:** site canonicals point variously at `cyberg7.com` and
  `launchnow.cyberg7.com.sg`, not `intake-launchnow.cyberg7.com.sg`. Broader than the funnel; flagged.

## Out of scope
- Stripe/checkout UI and the "pay to publish" step (backend/preview-side).
- The generated preview site at `<slug>-preview.cyberg7.com.sg` (backend-produced, host-agnostic).
- Any `website-factory` backend changes (CORS allow-list entry, tenant logic, status API) — coordinated
  with the owner under Gate 1, not edited from this repo.
- Merging the backend branch / Airtable schema (owner-reserved).
