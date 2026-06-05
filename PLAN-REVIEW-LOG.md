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
