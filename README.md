# Nectar — Sponsor Portal

Single-page HTML app that lets borrowers log in via magic link and generate account statements for their accounts.

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
  2. **`view-picker`** — table of accounts returned by `get_sponsor_deals()`, in the order the backend returns them (open first, closed after — the frontend does not re-sort)
  3. **`view-statement`** — statement page for the selected account
- `supabase.auth.onAuthStateChange` is the single source of truth — `SIGNED_IN` routes to the picker, `SIGNED_OUT` to login.

## Admin impersonation

Emails listed in the `MASTER_EMAILS` constant (top of the `<script>` block) see an extra "View as sponsor:" dropdown above the picker. Selecting a sponsor:

- Calls `get_sponsor_deals_as(p_email)` instead of `get_sponsor_deals()` — returns **all** statuses (Open / Closed / Workout), not just Open
- Calls `get_sponsor_statement_as(p_deal_id, p_email)` instead of `get_sponsor_statement(p_deal_id)`
- Shows a sticky amber banner — "Viewing as: <email> (admin mode)" with a "Return to my view" button. The banner is hidden by `@media print` so it does not appear on the PDF
- Populates from `get_sponsor_emails()` (returns `[{email, deal_count}]`)

All three `_as` RPCs are server-gated to master admins — calling them as a non-master returns nothing, so the frontend list is purely a UX convenience. Non-master users never see the dropdown or banner.

## Statement layout

Matches the existing PDF template the servicing team uses (Walker's spreadsheet):

- **Brand bar** — purple (`#5b3fb5`), "Nectar" wordmark left, `www.usenectar.com` right
- **Title row** — entity name, account number, prominent `account_status` badge, month-first statement-date field (defaults to today), and the `Data as of {as_of}` reporting date
- **Two-column** — Payment Overview / Account Information
- **Activity** — recorded payments and missed months on or before the statement date, newest-first
- **Payment Schedule** — origination row (`-advance`), one row per scheduled payment with `Status` and `Buyback Option` columns, and a `$0.00 / NA` post-term row. Runs the **full contractual schedule through maturity**, with future rows muted and tagged `Scheduled`. Rendered **newest-first**, matching the order the backend returns `dpd` in
- **Footer** — `DISCLAIMER_TEXT` plus the servicing contact line

### Dates

All user-facing dates render month-first (US order).

The statement-date field is a **plain text input, not `<input type="date">`** — this is deliberate. Native date inputs render in the *viewer's* browser locale (day-first outside the US) and that is not overridable by CSS or attributes. A text field is the only way to guarantee one unambiguous month-first date on screen and on the PDF. Do not "improve" it back to a native date picker.

`parseMDY()` parses what the user types. It is strict: `MM/DD/YYYY` only, and it rejects dates that would silently roll over (`02/31/2026` → Mar 3 if handed straight to `new Date`). On an unparseable value the field flags red and reverts to the last good date rather than re-rendering against something it did not understand; on a valid one it normalizes the display (`1/5/2026` → `01/05/2026`) and clamps future dates to today.

### Account status

`deal.account_status` / `d.account_status` arrive from the backend as ready-to-display strings ("Current", "Past Due - 405 days", "Paid in Full - Standard Buyback", "Charged Off (Loss)", "Closed - Workout"). `acctStatusClass()` buckets them into `good` / `bad` / `muted` / `neutral` for color only — the string itself is never parsed for meaning and is rendered verbatim. Shown as a badge at the top of the statement and as a column in the picker.

### Missed vs. received

Each `dpd` row carries `is_missed` and `received_date` from the backend; the frontend no longer derives missed status from `actual_amt`. `is_missed` is set only once the due month has closed with nothing received, so a month later covered by a catch-up payment comes back with `is_missed = false` **and** a `received_date` — that is intentional and is rendered as paid, not missed.

`isMissed()` layers one frontend-only rule on top: the statement-date picker can back-date a statement, and a month that closed *after* the selected date had not yet been missed as of that statement. The three rendered states are `Missed` (red badge), `Paid {date}` + subtle `Late` tag (`received_date > due_date`), and plain `Paid {date}`.

### Ordering

**Both tables render newest-first.**

`dpd` arrives newest-first and is never re-sorted. Order-dependent math (buyback index alignment, first-due / schedule end, the running reserve balance) needs oldest-first, which `renderStatement` derives as `dpd.slice().reverse()` — reversing the backend's guaranteed order rather than imposing a sort of its own. `renderSchedule` builds rows oldest-first so `buyback[i]` stays aligned with payment `i+1`, then reverses once before writing them out.

`renderActivity` is the exception to "preserve backend order": it interleaves two sources (payments from `get_sponsor_payments`, missed rows derived from `dpd`), and merging two lists still needs an explicit ordering, so it sorts descending. Its `events` array is display-only — nothing downstream reads it — so that ordering carries no arithmetic.

### Future rows

`dpd` carries the **full contractual schedule, including payments that have not come due yet**. Future rows are flagged `is_future` and are hollow: `actual_amt` 0, `received_date` null, `is_missed` never true, and `reserve_balance_eom` **null**.

They render with `.row-future` (muted) and a `Scheduled` tag, never a Missed or Paid treatment.

The null `reserve_balance_eom` is the sharp edge. The Current Reserve Balance walks the array keeping the last qualifying row, and `Number(null)` is `0` while `Number.isFinite(0)` is `true` — so a null future row read as a genuine zero balance and wiped out the last known figure. The loop now skips `reserve_balance_eom == null` **before** coercing, leaving the balance on the most recent past row. There is a mutation-tested assertion covering this; without the guard the balance collapses to `$0.00`.

Two other consequences of the array growing:

- **Buyback is keyed to `pmt_num`, not array position.** `buyback[n]` is the amount after payment `n+1`. Position and payment number coincided only while the array was a contiguous run from payment 1; `pmt_num` is the contract, so index by it (falling back to position when absent).
- **`totalSched` is now correct.** The IO+balloon interest recovery sums every scheduled payment minus advance. While the array was capped that sum was short by every remaining payment, understating interest and skewing buyback figures. It now covers the full term.

`missedCount` is unaffected — `is_missed` is never true on a future row.

`currentSched` deliberately does **not** skip future rows: the reporting ceiling often sits in the previous month, which flags the current month's payment as future even though it is exactly what is due now, and that figure is the "Total Due {month}" line.

### Maturity vs. schedule end

Two dates that are easy to conflate, and were:

- `deal.maturity_date` — the contractual maturity, authoritative from the backend. Use this for the **Maturity Date** row.
- the last `dpd` row — **still not maturity**, even now that the array runs the full term. The explicit field is the contract; the array is data.

Never derive maturity from the `dpd` array. `scheduleEnd` and `maturity` are separate variables in `renderStatement` for exactly this reason.

The "$0.00 / NA" post-term row is anchored to `scheduleEnd` and suppressed while the schedule is truncated (`scheduleEnd` earlier than `maturity`), so it renders on a full schedule and stays hidden on a capped one.

`deal.days_past_due` is also available on the payload. Currently unused — `account_status` already carries "Past Due - N days" as a display string.

### Fields taken from the backend as-is

Do not recompute these client-side:

- `term_months` — the contractual term. Counting `dpd` rows is off by one (61 vs 60).
- `method` on `get_sponsor_payments` — already mapped for display (`Cash` → `Wire`, `Bank Account` → `ACH`).
- `maturity_date` — the contractual maturity. The last `dpd` row is the reporting ceiling, not maturity.
- `account_status`, `is_missed`, `as_of`, and the `dpd` / `get_sponsor_payments` sort orders.

### Disclaimer

`DISCLAIMER_TEXT` is a single template string in a `<script>` block at the **top** of `index.html`, feeding both the on-screen footer and the PDF. The current text is compliance-approved (source: Brittany, portfolio management call) — **do not edit it without compliance sign-off**.

It is **HTML, injected verbatim rather than escaped** — it ships its own `<p>` tags and a `mailto:` anchor. This is safe because it is a static constant that never contains user input; escaping it would print the tags literally. If it is ever replaced, the replacement must also be trusted HTML.

Two things follow from the approved text that are easy to break:

- The servicing contact line lives **inside** `DISCLAIMER_TEXT`. The footer must not add one of its own or the statement shows it twice.
- The global `* { margin: 0 }` reset zeroes paragraph margins, so `.stmt-footer p + p` restores the spacing. Keep the styling in CSS rather than editing the approved text.

### Buyback option math

`computeBuybackSchedule(advance, monthlyPmt, termMonths, lockoutMonths)` (in `index.html`) returns an array of length `termMonths` whose `i`-th entry is the buyback amount after payment `i+1`:

- If `monthlyPmt * termMonths < advance * 1.05` → the deal is IO+balloon. Returns `advance × 1.01 + max(0, term - 2 - i) × charge` for unlocked months (floor = 1% prepayment premium plus interest charges through the penultimate month), and `$0` at maturity. `charge` defaults to `monthlyPmt`; the caller may pass a more precise per-month interest as an optional argument.
- Otherwise → solves for the implied periodic rate by bisection, then amortizes.
- Months `1..lockoutMonths` are `null` (rendered as `NA`).
- `buyback_lockout_months` is per-deal (defaults to 6) and is returned by `get_sponsor_statement`.

### Per-month interest precision (IO+balloon)

`sched_amt` is stored to 2 decimals, so multiplying the rounded value by `term-2-i` accumulates up to `term × $0.005` of error. To recover the contractual per-month interest, `renderStatement` sums all scheduled payments, subtracts `advance`, and snaps the resulting total interest to the nearest dollar when the gap is within `term × $0.01` (5× the cumulative rounding bound — tight enough that genuinely non-integer totals are not perturbed). The snapped total divided by `term` becomes the `charge` passed into the buyback function.

### "Regular Monthly Payment"

Derived client-side as the mode of `sched_amt > 0` across the `dpd` rows. This naturally excludes the balloon row on IO+balloon deals (where the balloon `sched_amt` appears only once).

## Print to PDF

The "Download PDF" button calls `window.print()`. The `@media print` block hides the app chrome (header, logout, back link, the button itself), sets Letter / 0.5in margins, and applies `page-break-inside: avoid` to rows so the Payment Schedule doesn't tear awkwardly across pages.

## Files

```
index.html   — entire app (HTML + CSS + JS, inline)
README.md    — this file
```
