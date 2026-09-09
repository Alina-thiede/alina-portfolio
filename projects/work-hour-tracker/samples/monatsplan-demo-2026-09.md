# Month plan 2026-09 — Alina Thiede

> **Demo data** — written to the `agentic_demo` sandbox workspace, not production.
> Based on [`planungsreview-demo-2026-08.md`](planungsreview-demo-2026-08.md). Decided 2026-09-08.

September 2026 has **30 days, 22 workdays, no public holidays.** Fridays are
treated as non-working by decision, which leaves **18 effective days**.
106 h across those is **5.89 h per effective day** — almost exactly the 5.87 h
per booked day August actually sustained. Spread across all 22 calendar
workdays the script reports 4.82 h/workday.

## Distribution

| Project | Planned | Already booked | Remaining | August actual | Change |
|---|---:|---:|---:|---:|---:|
| Work-hour-tracker | 50.00 | 0.00 | 50.00 | 24.00 | **+26.00** |
| Aerzte App | 46.00 | 0.00 | 46.00 | 55.00 | −9.00 |
| Fabric Demo | 10.00 | 0.00 | 10.00 | 9.00 | +1.00 |
| **Total** | **106.00** | **0.00** | **106.00** | **88.00** | **+18.00** |

Nothing is booked in September yet, so the remaining column equals the plan.

## Why each line

**Work-hour-tracker — 50.00 h.** Chosen focus, and the biggest change in this
plan. It becomes the largest line for the first time, ahead of Aerzte App.
August delivered 24 h and July 38 h, so 50 h is **12 h above its best month
ever and 26 h above last month.** That is a deliberate reversal, not an
extrapolation — see the note below.

**Aerzte App — 46.00 h.** Deliberately below the 55 h it actually ran at in
August and the 50 h it was planned at. The wind-down is plausible: August closed
with the query model written up for handover, and the second half of the month
was finishing work (accessibility, error handling, mobile layout) rather than
new surface. This line is the capacity that Work-hour-tracker is being given.

**Fabric Demo — 10.00 h.** Down from 20 h, planned at the level it actually
receives: 11 h in July, 9 h in August. This closes the `chronic-miss` finding by
fixing the plan rather than by demanding more discipline. 10 h should now come
out near 100 % instead of 45 %.

## What this plan requires to work

The review's data points the other way on two of these lines, so this is stated
once, as a mechanism rather than a caution:

- **Aerzte App has beaten its plan two months running** (107 % in July, 110 % in
  August) and has won every contest for leftover hours. Planning it 9 h *below*
  its August actual only holds if the handover really has ended the feature
  work. If it has not, Aerzte App will take the hours from Work-hour-tracker
  again — that is exactly the August mechanism, where Fabric Demo did not get
  touched until the 19th.
- **Work-hour-tracker at 50 h needs whole days, not gaps.** August's evidence is
  specific: its two 6-hour blocks (per-person project rates, deployment pipeline
  cleanup) produced substantial work, while four separate 2-hour slots did not
  add up to much. 50 h over 18 effective days is 2.8 h/day if it is spread —
  which is the shape that already failed. **Two full days a week booked to
  Work-hour-tracker** is the concrete form this plan assumes.
- **Fridays are out.** If a Friday gets worked anyway, the effective days go from
  18 back up and the plan gets easier, not harder — but the 106 h was sized for
  18, so nothing needs re-planning in that case.

## How the month is measured

- **Total:** 106.00 h. Anything from 95 h up is a hit at the 90 % line, which
  both previous months cleared (94 %, 88 %).
- **Fabric Demo:** the real test of this plan. At 10 h it should land near 100 %.
  A third month under 60 % — under 6 h — means the problem was never the number.
- **Work-hour-tracker:** the ambition. 50 h is a doubling; even 38 h would match
  its best month. Below ~30 h means the focus decision did not survive contact
  with Aerzte App.
- **Unbooked Fridays are expected** and are not a finding this month.

## Changes to the database

Written to `ProjectMonthlyPlans` in the **`agentic_demo`** workspace on
2026-09-08 via `monatsplanung.mjs --plan-month 2026-09 --total 106`:

| Project | Month | plannedHours |
|---|---|---:|
| Work-hour-tracker | 2026-09 | 50 |
| Aerzte App | 2026-09 | 46 |
| Fabric Demo | 2026-09 | 10 |

Three rows written and read back from the database; `mismatch` was empty and the
read-back total was 106.00 h. No project was created, no time entry was touched,
and production was not connected to at any point.

**Note on the demo dataset:** the seed in `tools/demo-data/` documents
`agentic_demo` as July + August only, with 6 plan rows. These three rows take it
to 9 and add a September month that the seed does not describe. Rebuild with
`tools/demo-data/wipe.mjs` + `seed.mjs` to get back to the documented state.
