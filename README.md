# Nectar — Sponsor Portal

Single-page HTML app that lets borrowers log in via magic link and generate account statements for their loans.

Mirrors the conventions of the `investor-data-room` repo: everything is inline in `index.html`, no build step, no dependencies beyond two CDN scripts (Supabase JS + Google Fonts). GitHub Pages serves `main` directly.

## Run locally

```sh
python3 -m http.server 8001
# open http://127.0.0.1:8001/
```

That's it — Supabase JS is loaded from `esm.sh` at runtime, so there is nothing to install.

Note: the magic-link redirect lands on `window.location.origin + window.location.pathname`, so when testing locally the link will come back to `http://127.0.0.1:8001/`. If you keep the same browser tab open the session will appear automatically.

## Deploy

```sh
git push origin main
```

GitHub Pages serves the site at https://nectarinvestments.github.io/sponsor-portal/ and rebuilds within 1–2 minutes of a push.

## How it works

- Public `SUPABASE_ANON_KEY` is hardcoded in `index.html` — the key has no privileges of its own; access is gated by Postgres RLS + the two `SECURITY DEFINER` RPCs (`get_sponsor_deals`, `get_sponsor_statement`), which check the authenticated email against the `sponsor_access` allowlist.
- Three view states, all toggled in-place by `setView()`:
  1. **`view-login`** — magic-link form
  2. **`view-picker`** — table of loans returned by `get_sponsor_deals()`
  3. **`view-statement`** — statement page for the selected loan
- `supabase.auth.onAuthStateChange` is the single source of truth — `SIGNED_IN` routes to the picker, `SIGNED_OUT` to login.

## Statement layout

Matches the existing PDF template the servicing team uses (Walker's spreadsheet):

- **Brand bar** — purple (`#5b3fb5`), "Nectar" wordmark left, `www.usenectar.com` right
- **Title row** — entity name, loan number, statement-date picker (defaults to today)
- **Two-column** — Payment Overview / Account Information
- **Activity** — recorded payments on or before the statement date
- **Payment Schedule** — origination row (`-advance`), one row per scheduled payment with the `Buyback Option` column, and a `$0.00 / NA` post-term row
- **Footer** — verbatim disclaimer from the existing template

### Buyback option math

`computeBuybackSchedule(advance, monthlyPmt, termMonths, lockoutMonths)` (in `index.html`) returns an array of length `termMonths` whose `i`-th entry is the buyback amount after payment `i+1`:

- If `monthlyPmt * termMonths < advance * 1.05` → the deal is IO+balloon. Returns `advance` for non-locked months and `0` at maturity.
- Otherwise → solves for the implied periodic rate by bisection, then amortizes.
- Months `1..lockoutMonths` are `null` (rendered as `NA`).
- `buyback_lockout_months` is per-deal (defaults to 6) and is returned by `get_sponsor_statement`.

### "Regular Monthly Payment"

Derived client-side as the mode of `sched_amt > 0` across the `deal_dpd` rows. This naturally excludes the balloon row on IO+balloon deals (where the balloon `sched_amt` appears only once).

## Print to PDF

The "Download PDF" button calls `window.print()`. The `@media print` block hides the app chrome (header, logout, back link, the button itself), sets Letter / 0.5in margins, and applies `page-break-inside: avoid` to rows so the Payment Schedule doesn't tear awkwardly across pages.

## Files

```
index.html   — entire app (HTML + CSS + JS, inline)
README.md    — this file
```
