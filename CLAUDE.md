# CLAUDE.md — Roofing Forecast

## What this is

Single-page cash-flow forecasting app for Roofing (South East) Ltd (RSE) — "the
Forecast app". One self-contained `index.html` (~670 KB, vanilla HTML/JS, no build
step) talking to Supabase over PostgREST, deployed on GitHub Pages. QuickBooks Online
is integrated via Supabase Edge Functions. Stuart is the live user. British English
throughout.

## Current status (June 2026)

The June 2026 security hardening is complete: real Supabase Auth replaced the cosmetic
PIN, RLS locks every table to logged-in users, and QBO tokens are now server-side only.
Live version is `v2.75-Jun-02`. One item is deferred: leaked-password protection is a
Pro-plan feature and the project is on the Free plan, so it cannot be enabled (lowest-
severity advisor warning).

## Repository & deployment

- **GitHub repo (source of truth):** `github.com/stuartmarshall-star/Roofing-Forecast`,
  branch `main`. GitHub Pages serves the repo root; pushing to `main` redeploys in ~1 min.
- **Files in the repo:** `index.html` (the whole app), `eula.html`, `privacy.html`,
  this `CLAUDE.md`.
- **Deploy = commit + push `index.html` to `main`** (or upload via the GitHub web UI).
- **Drift warning:** a richer local working copy lives outside the repo (Stuart's
  OneDrive: `…/5. R&D/Forecast App/1. Claude Code/roofing-forecast/`, which also holds
  `docs/` and `supabase/`). It has drifted from production before (was v2.67 while live
  was v2.74). **Always treat the GitHub repo as source of truth** — clone/pull and diff
  before deploying.

## Supabase

- **Project ref:** `asaedxtvehsjmrkdpegj`
- **Tables:** `jobs`, `liabilities`, `credits`, `settings`, `unallocated_costs`,
  `qbo_credentials`.
- **Edge functions:** `qbo-auth` (starts OAuth), `qbo-callback` (OAuth redirect →
  stores tokens; public/no-JWT so Intuit can reach it), `qbo-sync` (pulls QBO data and
  applies it to jobs; also serves a `{check:true}` connection probe), `AI-function`.
  `qbo-auth`, `qbo-sync` and `AI-function` require a valid JWT; `qbo-callback` is public.
- **Anon/publishable key + project URL are in `index.html`** — expected and safe;
  security rests on Auth + RLS, not on hiding the key.
- **SQL changes:** version-controlled migrations only, applied deliberately. **Always
  flag SQL changes to Stuart before running.** Migrations live in the local working copy
  under `supabase/migrations/` (e.g. `0002_secure_rls.sql`).

## Auth & security model (current, post-June-2026)

- **Supabase Auth**, single shared login (`code@forecast.com`) for Stuart + accountant.
  `supabase-js` is loaded from CDN; `onAuthStateChange` drives app boot — nothing loads
  until a real session exists (not a CSS toggle).
- **Data calls run as the `authenticated` role:** `sbHeaders()` sends
  `apikey: <publishable key>` + `Authorization: Bearer <user session token>` (via
  `sbAuthToken()`). Realtime and edge-function calls use the same token.
- **RLS (migration `0002_secure_rls`):** the old `Public access USING(true)` policies
  are gone; every table has an `authenticated`-only policy; `unallocated_costs` has RLS
  enabled; `qbo_credentials` has RLS on with **no policies** (service-role only).
- **QBO tokens live in `qbo_credentials`** (single canonical row id
  `00000000-0000-0000-0000-000000000001`), readable only by the service role inside the
  edge functions. The client never reads tokens — it asks `qbo-sync {check:true}` for
  connection status.
- **Deferred:** leaked-password protection (HaveIBeenPwned) is Pro-plan-only; project is
  on Free. Mitigate with a strong, unique shared password.

## QuickBooks sync (how it behaves)

- `qbo-sync` does a full pull of invoices, bills, credit memos, vendor credits and
  customers; matches QBO customers to jobs by E-number (job name starts with `E####`);
  and writes billing/cost actuals back onto `jobs.actuals`.
- It is **destructive on actuals** for months QBO has data for, and must **not** wipe
  months with no QBO data. Future-dated invoices are filtered out. See Section 2.
- A first full sync takes ~50 s. "0 updated / N skipped" means the data already matched
  — healthy, not an error.

## Working on the single-file app (practical notes)

- `index.html` contains some very long/minified lines; reading the whole file exceeds
  context limits and even some line-range reads fail on the long lines. Use Grep with
  context, targeted line-range reads that avoid those lines, or scripted (node)
  transforms for bulk edits.
- Verify changes by serving the file locally (a static server) and driving it against
  the live Supabase backend (log in, confirm load/save) before deploying.

## Conventions

- **British English.**
- **Proportionate engineering** — one small business, one main user. Simplest thing
  that works and is maintainable; no over-engineering.
- **"Production-ready" means a draft for review, not a guarantee** — flag assumptions
  and anything you couldn't verify; trace root causes rather than guessing.

## Scope & boundaries

- **This repo is the Forecast app only.** The Measure tool (separate client-side HTML)
  and RoofPay / Subby Portal (CIS self-billing) are **separate projects — do not fold
  them in here.**
- The MACS&Co measure→BOM engine is **not in scope** (another team's code).
- A possible future React/Vite + Supabase migration is noted in the local working copy
  at `docs/MIGRATION-TO-VITE.md` — no rush; the single-file app is fine until the
  feature set justifies the move.

## Purpose — what this app is for

A project-pipeline-driven forward cash view for an SME roofing contractor (RSE).
Unlike a treasury/accounting tool that forecasts at company level from bank actuals,
this works bottom-up: contract → billing schedule → payment-terms lag → retention →
cash due, month by month. The whole point is cash *timing* — when money actually
lands and leaves — not period totals. Most of the subtle logic exists to model that
timing correctly. RSE is the live user; it runs on GitHub Pages + Supabase.

When proposing changes, optimise for: timing accuracy, the user trusting the closing
balance, and not silently losing or double-counting income across month boundaries.
"Better" means the forecast more faithfully reflects real cash movement — not more
features.

## Protected logic — what each guarded area is and why

- No breaking changes to `buildSchedule` — the core engine that turns a contract +
  duration + payment terms into the month-by-month billing/cost/retention schedule.
  Every downstream number depends on it. Test against a known job before committing.
- Actuals locking / "used value" logic — once a real figure is recorded for a month,
  it overrides the forecast for that month (actual if present, else forecast). This
  is deliberate: it lets the forecast converge on reality as the month closes. Don't
  reintroduce forecast figures over recorded actuals.
- Carry-forward (the "incl. £X b/fwd" note) — when a closing balance is recorded
  (costs paid, no income), billing income due that month on lagged terms hasn't landed
  yet, so it must roll into the next month rather than vanish. Don't "simplify" it away.
- QB sync is destructive on actuals — it overwrites actuals[month] fields. Never wipe
  months with no QB data; future-dated QB invoices are filtered out and should surface
  as overrides, not be lost.

## Wider context (guardrail, not a build instruction)

This is one of three RSE apps that may share a Supabase backbone later (the others:
the Measure + Engine main app, and RoofPay/Subby Portal — both out of scope here and
not integrating now). Do NOT build toward integration or add cross-app features
unless explicitly asked. The only ask: where it's cheap to do so, keep the data model
clean and avoid hard-coding RSE-only assumptions that would foreclose a future
multi-tenant or shared-data direction. Simplest-thing-that-works still wins.

Known blocker for any future shared/multi-tenant direction: Supabase RLS policies are
currently using (true) with check (true) (effectively open), and the PIN is
client-side/cosmetic. The anon key + URL are in the client, so today anyone loading
the page has full read/write to all tables. Fine for single-user RSE use; must be
fixed before any data is shared beyond Stuart/accountant.

> **[Annotation — updated June 2026]** The "Known blocker" paragraph above describes the
> state *before* the June 2026 security work and is now **resolved**; it is retained
> verbatim for historical context only. As of `v2.75`: the cosmetic PIN was replaced
> with Supabase Auth, RLS restricts every table to the `authenticated` role (no more
> `using (true)` public policies), and QBO tokens were moved to the service-role-only
> `qbo_credentials` table. The anon key + URL remaining in the client is expected and
> safe under that model. See "Auth & security model" above for the current state.
