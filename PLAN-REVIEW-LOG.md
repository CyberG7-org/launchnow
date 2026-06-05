# Plan Review Log: Native LaunchNow /intake + /status funnel
Act 1 (grill) complete — plan locked with the user (2026-06-05). MAX_ROUNDS=5.

## Act 1 — decisions locked (you ↔ Claude)
- **Q1 Page tech** → Vanilla HTML+JS, reuse this site's chrome. Rationale: `/intake` must match
  `intake-launchnow.cyberg7.com.sg` (which *is* this repo); reusing its own CSS/nav/footer + the
  `contact.html` form pattern guarantees the match with no build step. React/Next would re-implement the
  design and risk drift.
- **Q2 Backend reach** → Direct browser `fetch` to `agency.cyberg7.com.sg/api/intake` (+ status) with
  CORS + honeypot, mirroring the existing contact form. `vercel.json` stays free of any agency reference;
  backend adds our origin to its CORS allow-list (Gate 1). Rejected: the broad page/_next/launchnow proxy
  rewrites on `Connect-intake-form`.
- **Q3 Pay timing** → Free preview first. Submit → generate → `/status` → preview; pay-to-publish after.
  Submit handler still defensively follows a `checkoutUrl` if the backend returns one.
- **Q4 Build scope** → `/intake` + `/status` only. Payment + the generated preview site are the backend's
  domain; no Stripe code in this repo.

Folded defaults (recommended, low-controversy): always send `tenant:"launchnow"`; honeypot per
contact.html; mirror gtag + Meta Pixel with a lead event on submit; develop on `claude/busy-curie-pjEii`;
drop the proxy + keep the relative-`/intake` CTA edits; opportunistic terms/pdpa SEO-tag cleanup.

## Act 2 — Codex review (Claude ↔ Codex)
NOT RUN. The `codex` CLI is not installed/authenticated in this environment (Act-2 prerequisite check
failed). The cross-model adversarial review cannot run until `codex` is installed
(`npm i -g @openai/codex@latest`) AND authenticated (`codex login`, which requires the owner's OpenAI/
ChatGPT account — cannot be done from this remote container). Surfaced to the user; no fake convergence.

## Round 1 — Codex (VERDICT: REVISE)
**Findings**

- [PLAN.md:20](</home/user/intake-launchnow/PLAN.md:20>) leaves the form field set blocked on `IntakeForm.tsx`, while [PLAN.md:38](</home/user/intake-launchnow/PLAN.md:38>) still describes implementation/deploy verification; this can ship a frontend that serializes the wrong payload or misses required backend validation. Fix: add a hard Gate 0 requiring copied backend request/response schemas, field names, examples, and status contract before any `/intake` implementation.

- [PLAN.md:51](</home/user/intake-launchnow/PLAN.md:51>) treats honeypot + CORS as abuse mitigation for free generation, but CORS does not stop direct scripted POSTs and honeypots are trivial to bypass. Fix: require backend rate limiting, tenant-side quotas, Turnstile or equivalent bot challenge, and server-side tenant authorization before enabling free preview generation.

- [PLAN.md:24](</home/user/intake-launchnow/PLAN.md:24>) and [PLAN.md:26](</home/user/intake-launchnow/PLAN.md:26>) expose `jobId` in `/status?job=...` without specifying entropy, authorization, or response redaction; guessable IDs could leak customer brief data or preview URLs. Fix: use an opaque unguessable status token/status URL from the backend and ensure the status endpoint returns only minimal display-safe fields.

- [PLAN.md:24](</home/user/intake-launchnow/PLAN.md:24>) says to follow any returned `checkoutUrl`; that is an open redirect/phishing footgun if the backend response is malformed or compromised. Fix: only redirect to an allow-listed HTTPS origin/path, otherwise ignore and show support fallback.

- [PLAN.md:26](</home/user/intake-launchnow/PLAN.md:26>) says “polls the backend status endpoint on an interval” but gives no interval, backoff, max duration, visibility handling, or `Retry-After` behavior; this can create needless backend load and poor UX during outages. Fix: specify exponential backoff with jitter, pause on hidden tabs, honor `Retry-After`, and cap polling with a clear recovery path.

- [PLAN.md:32](</home/user/intake-launchnow/PLAN.md:32>) says to mirror Meta Pixel `1961722217792735`, but `rg` finds no `fbq`/pixel currently in the repo; adding it is not mirroring, it is a new tracker with privacy and consent implications. Fix: require owner confirmation, update privacy/PDPA disclosures, and keep analytics events free of PII.

- [PLAN.md:46](</home/user/intake-launchnow/PLAN.md:46>) equates the new intake POST with the existing contact webhook, but [contact.html:1932](</home/user/intake-launchnow/contact.html:1932>) only sends basic contact fields while intake likely sends richer business/content data to a different processor. Fix: add explicit PDPA consent/disclosure for brief generation, backend processing, retention, and preview publication before submit.

- [PLAN.md:30](</home/user/intake-launchnow/PLAN.md:30>) claims “19 ‘Get Started’ links,” but current links include varied CTA text and cross-page nav targets such as [index.html:2499](</home/user/intake-launchnow/index.html:2499>) and [index.html:3356](</home/user/intake-launchnow/index.html:3356>), while about nav still points to `/contact`; this is easy to partially update. Fix: replace by exact `href="https://launchnow.cyberg7.com.sg/intake"` inventory across all HTML files, not by visible text/count.

- [PLAN.md:35](</home/user/intake-launchnow/PLAN.md:35>) scopes SEO cleanup mostly to `/intake`, `terms`, and `pdpa`, but existing canonical/OG/JSON-LD URLs still point to `cyberg7.com` in core pages like [contact.html:27](</home/user/intake-launchnow/contact.html:27>). Fix: define the canonical host for the whole site and update canonical, OG, and structured-data URLs consistently or explicitly defer all non-funnel SEO.

- [PLAN.md:73](</home/user/intake-launchnow/PLAN.md:73>) puts backend CORS, tenant logic, and status API out of scope, but those are prerequisites for the proposed frontend to function and to be safe. Fix: move backend acceptance criteria into mandatory external dependencies with concrete owner sign-off before merge.

VERDICT: REVISE

### Claude's response (Round 1)
Incorporated 9/10 into PLAN.md rev 2; 1 partial (method adopted):
- Added **Gate 0** (contract-in-hand) blocking form impl [1]; reframed backend items as **external dependencies w/ acceptance criteria + owner sign-off** [10].
- Abuse: honeypot kept but flagged insufficient; real protection = backend rate-limit/quotas (acceptance crit d) + optional Turnstile [2].
- **Opaque status token** + minimal display-safe status response (acceptance crit b,c); FE renders no sensitive fields [3].
- **checkoutUrl open-redirect guard** (allow-list scheme+host, else ignore+fallback) [4].
- **Polling policy**: backoff+jitter, pause on document.hidden, honor Retry-After, max duration + recovery [5].
- **Meta Pixel**: VERIFIED absent (no `fbq` in repo) — reframed from "mirror" to a NEW tracker needing owner go-ahead + PDPA disclosure [6].
- **PDPA consent** checkbox + disclosure before submit [7].
- **SEO canonical-host**: made an explicit decide-or-defer [9].
- Partial [8]: the "19 links" count was independently verified accurate (index 14 + 5 single-CTA pages); adopted the **exact-href inventory** method anyway; `/contact` "Talk to Our Team" links are intentional, not missed repoints.

## Round 2 — Codex (VERDICT: APPROVED)
Rev 2 addresses the prior 10 findings materially:

- Backend schema/status contract is now a hard Gate 0 before form wiring.
- Free-generation abuse is correctly moved to backend rate limits and per-tenant quotas, with bot challenge called out as an owner decision.
- Status tracking now requires opaque unguessable tokens and minimal display-safe responses.
- `checkoutUrl` handling now has frontend and backend allow-list requirements.
- Polling now has backoff, jitter, visibility pause, `Retry-After`, and a max duration.
- Meta Pixel is correctly treated as a new tracker, not existing repo behavior.
- PDPA consent and PII-free analytics are now explicit.
- CTA replacement is now exact-href based; I verified there are 19 current matches.
- Canonical-host inconsistency is now a decision point instead of being hand-waved.
- Backend work is now listed as external acceptance criteria, not silently out of scope.

No genuinely new material blocker found. The remaining owner decisions are explicit gates/open questions rather than plan flaws.

VERDICT: APPROVED
## Resolution
Converged in 2 rounds (R1 REVISE -> R2 APPROVED). Plan cross-model approved. Implementation remains gated on Gate 0 (backend contract) + Gate 1 (backend live); owner final sign-off pending.

## Build — pre-Gate-0 shell (2026-06-05)
Implemented the backend-independent parts of the plan on claude/busy-curie-pjEii:
- CTA repoint: ported the 19 exact `/intake` hrefs from Connect-intake-form across index/about/contact/pdpa/privacy/terms (0 old-host refs remain). No vercel.json proxy added (CORS approach).
- intake.html (new): LaunchNow brief page cloned from contact.html chrome. PROVISIONAL fields (flagged — final set from Gate-0 IntakeForm.tsx), required PDPA consent, honeypot, gtag lead event. Submit handler: honeypot, native-required consent, checkoutUrl open-redirect allow-list, opaque-token redirect to /status — all gated behind BACKEND_READY=false until Gate 0/1.
- status.html (new): noindex progress page. Polling scaffold — exponential backoff+jitter, hidden-tab pause, Retry-After, 10-min max duration, display-safe rendering — gated behind BACKEND_READY=false.
- Verified: both pages serve HTTP 200 locally; tag balance clean (section 6/6, script 6/6 & 7/7, form 1/1 & 0/0); 0 leftover contact wiring in intake.
TODO at Gate 0/1: pin exact fields + payload keys + status endpoint URL/response; flip BACKEND_READY; trim status.html trailing marketing sections; owner decisions (canonical host, Meta Pixel, Turnstile, Ads conversion label).
