# Plan Trajectory — project notes

Single-file dashboard (`index.html`) tracking Nick & Lauren's 10-year debt payoff /
income growth / savings plan, starting July 1, 2026. No build step, no dependencies
beyond Chart.js (loaded from cdnjs). Everything — HTML, CSS, JS, and a live financial
model — lives in one file.

## What it does

- Recomputes a full 10-year financial projection **live in the browser** whenever any
  assumption changes (income, tax rate, childcare, loan terms, growth rates, targets).
- Shows a "ribbon" hero chart of net worth (savings minus debt) crossing from red
  (in debt) to green (saving), plus milestone cards (debt-free date, house fund date,
  year-10 net worth, current position).
- Lets the user log periodic "check-ins" (actual loan balance / bucket balances on a
  given date) that overlay as dots against the projected lines, so projection vs.
  reality is visible at a glance.
- Persists both the assumptions and the check-in log so the page remembers state
  between visits.
- Syncs that same data across devices via a private GitHub Gist (see "Sync" below).

## The financial model (core logic, all in the `recompute(a)` function)

Order of operations per plan-year:
1. **Income**: Nick starts at $103K/yr, +3%/yr. Lauren: fixed residency-year steps
   (years 1–3), jumps to an "attending" salary in year 4, then +3%/yr after. Both can
   be overridden per-year via the "Salary changes" table (see below) — projected
   values keep using the 3% rule; actual entries chain forward from whatever was
   last logged.
2. **Tax**: flat rate applied to gross (default 30%, user-set — deliberately *not*
   trying to model real marginal brackets; the person who owns this plan wants it
   simple and has explicitly said to leave it flat).
3. **401(k) deferral**: starts year 4, is Lauren-only, reduces net income by the
   deferral %, and feeds retirement growth alongside an employer match of the same
   nominal %.
4. **Expenses**: living costs (different $/mo for residency years 1–3 vs. after),
   plus childcare (kicks in once either spouse reaches a configurable age — currently
   32 — at a configurable $/mo), plus lifestyle spending (starts year 4 at a lower
   rate, steps up once the house fund hits its target).
5. **Debt paydown**: **two separate loan tranches** (Nick's and Lauren's — do not
   collapse these back into one combined balance; that was an earlier version and
   it's wrong now). Each tranche accrues its own interest rate annually. All
   available surplus pays down debt **avalanche-style** — whichever tranche has the
   higher interest rate gets paid first. A one-time "windfall" (bonus, tax refund,
   etc.) can be injected into a specific plan-year and **routed to a chosen bucket**
   (extra debt / emergency / house / retirement) via `windfallTarget`. "debt" joins
   that year's paydown pool; the others are applied straight to the bucket after the
   waterfall, bypassing the debt-first rule (so you can, e.g., fund retirement while
   still in debt). The one-time amount is deliberately excluded from
   `lastRetirementContribution` so it doesn't inflate the beyond-year-10 pace.
   `recompute` also tallies `totalInterest` (sum of interest accrued across both
   tranches) and accepts an optional `extraAnnual` (used by the "what moves the
   needle" levers to re-run the model with more cash toward debt).
6. **Savings waterfall**, only once *both* loan tranches hit zero: emergency fund
   (target $60K default) → house fund (target $160K default) → whatever's left goes
   to retirement as a cash-flow contribution. Emergency and house both grow at a
   HYSA-style rate (default 3%); retirement grows faster (default 7%) and also
   receives the deferral+match contribution every year regardless of the waterfall
   state.
7. **Beyond year 10**: a separate extrapolation (`extendRetirement` /
   `findFIAge`) holds retirement contributions flat at the year-10 pace and
   compounds forward to show projected balance at ages 60/65, plus an age at which
   the balance crosses 25× a configurable "desired retirement spending" figure (a
   rough FI heuristic, explicitly *not* meant to be precise — no sequence-of-returns
   modeling).

## Known modeling deviations from the original source plan

The original plan (a PDF the user supplied at the start of this project) used a
fixed, non-surplus-based debt payment schedule for residency years 1–3 that actually
*exceeded* that period's cash surplus — an inconsistency in the source, not
something to replicate. This dashboard instead applies one consistent rule for all
10 years: full surplus goes to debt once debt exists. Years 1–3 numbers will not
match the original PDF exactly as a result, and that's intentional, not a bug.

Interest is compounded **annually**, not monthly — there's a small (~1%) deviation
from the original PDF's year-4 loan balance because of this. Acceptable for a
tracking tool; would need a monthly-step simulation to close that gap exactly.

## UI structure

The page is split into **three tabs** via a sticky top nav (`.tabs`): **Dashboard**,
**Log**, **Plan**. Tabs are not separate DOM trees — every top-level element carries a
`data-tab="dashboard|log|plan"` attribute and CSS hides the non-active ones based on
`body[data-active=...]` (set by `switchTab`). So DOM order is unchanged; adding a
section just means tagging it. Charts live on Dashboard; `switchTab` calls
`chart.resize()` when returning there in case they were rebuilt (e.g. theme toggle)
while hidden. Assignment:
- **Dashboard**: hero, milestones, Where we stand, Your move this year, both charts,
  Cost of your debt, Year-by-year, Your road (timeline), Beyond year 10
- **Log**: Your logs (form + history)
- **Plan**: Quick adjust, Salary changes, All assumptions, Sync

Within each tab, top to bottom in the file:

- Theme toggle (dark/light, defaults to dark — this was a deliberate ask, don't flip
  the default back to light)
- Hero: animated net-worth number + ribbon chart
- Milestone cards
- **Where we stand today** — reality-anchored panel driven by the latest logs:
  debt-paydown + emergency + house progress bars, an ahead/behind verdict vs. the
  plan, and a debt-free countdown. Uses `latestKnown(field)` (most recent log that
  actually has that field) so a partial log doesn't reset anything to zero.
- **Your move this year** — this year's surplus ÷ 12 as a "save ~$X/mo" number; the
  bridge to the couple's separate daily-budgeting app.
- **Salary changes** (collapsed `<details>`) — per-year actual income override
- **Quick adjust** — 2 sliders (childcare $, tax rate) + a manual one-time windfall
  block (amount, year, target bucket, optional note). Childcare start age moved to
  All assumptions.
- **All assumptions** (collapsible `<details>`) — every other input
- **Your logs** (was "check-ins") — form + table. Blank number fields save as
  `null` (no data point), NOT zero. Optional free-text note per log renders as an
  italic sub-row (the shared quarterly journal).
- Debt payoff chart, **Cost of your debt** (total interest + sensitivity levers),
  saved-buckets chart (crosshair hover plugin — see below)
- Year-by-year table
- **Your road** — milestone timeline (plan start, Lauren attending, childcare,
  debt-free, house funded, FI)
- Beyond-year-10 milestones
- **Sync across devices** (collapsed `<details>`, moved to the bottom)

## Sync (GitHub Gist)

Data sync across devices works by pushing/pulling a JSON blob
(`{ updatedAt, assumptions, checkins }`) to/from a private Gist via the GitHub REST
API, authenticated with a personal access token scoped to `gist` only. Token + Gist
ID are entered once per device and stored in that device's `localStorage`
(`plan-gh-token`, `plan-gh-gist`) — they are **not** part of the synced data itself,
for obvious reasons.

**This is last-write-wins, not a merge.** If two devices save within moments of each
other, whichever pushes last overwrites the other's changes silently. No conflict UI
exists. Don't build one unless asked — it's a deliberate simplicity tradeoff, not an
oversight.

Local `localStorage` is always the fast/offline layer; Gist sync is a layer on top
that pushes ~1.2s after any change (`queueSync()` debounce) and pulls once on page
load if credentials are present.

## Visual design

Deliberately not the generic AI-dashboard look. Palette is a warm "financial ledger"
theme — parchment/paper tones in light mode, deep forest-green-black in dark mode —
with pine green / rust / gold / slate as the four semantic accent colors (debt =
rust, house/gold, retirement/pine, secondary = slate). Serif body font (Georgia
stack) for the ledger feel, sans-serif (system UI stack) for all interactive
chrome/labels/data. No external font loading — everything is system font stacks, to
avoid CDN/CSP issues since this runs as a bare static file.

Charts use Chart.js (CDN). All three main charts share a custom `crosshair` plugin
(registered once, applies globally) plus `interaction: { mode: 'index', intersect:
false }` so hovering anywhere near a year column shows the tooltip — this was a
deliberate fix after the default point-only hover felt broken with mostly-invisible
line points (`pointRadius: 0`).

## Things NOT modeled (explicitly out of scope, don't add without being asked)

- PSLF / loan forgiveness — explicitly ruled out by the plan owner, don't reintroduce
- Lawsuit settlement outcome
- Kids' costs beyond the single childcare line item
- Real marginal tax brackets (flat rate only, by request)
- Solo 401(k) for Nick (pending confirmation of BoundaryCare's tax filing status)

## If you're picking this up fresh

Read `assumptions` object and the `DEFAULTS` constant near the top of the `<script>`
block first — that's the entire data model surface. Then `recompute(a)` is the one
function that matters most; everything else (charts, tables, milestones) is just
rendering its output. `render()` is the orchestrator called after every input change.
