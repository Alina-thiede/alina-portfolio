# Month review 2026-08 — Alina Thiede

> Source: work-hour-tracker time entries, `agentic_demo` workspace.
> **All figures are demo data** — every entry note carries a `[DEMO]` prefix.
> Generated 2026-09-08 from `scripts/monatsplanung.mjs --month 2026-08`.

## 1. At a glance

| | |
|---|---:|
| Planned | 100.00 h |
| Actual | 88.00 h |
| Delta | −12.00 h |
| Fulfilment | **88 %** |
| Entries | 22 |
| Earnings | 9,810.00 € |
| Booked days | 15 of 21 workdays |
| Ø per booked day | 5.87 h |
| Ø per workday | 4.19 h |

## 2. Plan vs. actual per project

| Project | Planned | Actual | Delta | Fulfilment |
|---|---:|---:|---:|---:|
| Aerzte App | 50.00 | 55.00 | +5.00 | 110 % |
| Work-hour-tracker | 30.00 | 24.00 | −6.00 | 80 % |
| Fabric Demo | 20.00 | 9.00 | −11.00 | 45 % |
| **Total** | **100.00** | **88.00** | **−12.00** | **88 %** |

No unplanned project, no planned project left untouched — every plan line was
worked on, and nothing was worked on without a plan line.

## 3. What was achieved

**Aerzte App — 55 h across 11 entries.** The bulk of the month, and it reads as
a feature going from working to finished: the specialty search filter, the
lakehouse query behind the results list, pagination and empty states, then a
second pass over the same surface for the things that decide whether it ships —
an accessibility pass on the search form, error handling on the detail page,
caching for repeated lookups, and a mobile layout. Two entries stand slightly
apart: cleaning up the specialty reference data, and reviewing the data agent's
answers against the source tables — checking that the thing tells the truth, not
just that it responds. The month closes with the query model written up for
handover, which is why the +5 h overrun is not drift: it includes the part that
lets someone else take it over.

**Work-hour-tracker — 24 h across 7 entries.** Smaller and more scattered, and
almost entirely about the parts of the tool that had already been noticed as
weak: the monthly planning save path, per-person project rates, rate resolution
in the monthly statement script, edit-in-place for time entries, the month
navigation. Plus a row-level security policy review and a deployment pipeline
cleanup. Two 2-hour slots and two larger blocks — the larger blocks (per-person
rates, pipeline cleanup) are where the substance sits; the 2-hour slots read as
fitting work into gaps.

**Fabric Demo — 9 h across 4 entries.** All four entries fall in the last nine
days of the month (19.08 onwards): a lakehouse overview notebook, the demo
walkthrough, a report layout for the summary page, and a refreshed sample
dataset. That is coherent demo preparation and none of it is wasted — but it
only started once the other two projects had taken their share.

## 4. More than planned

**Aerzte App, +5 h (110 %).** The only project above plan, and the second month
running — July was 64/60 h (107 %). The overrun is small and the content
explains it: the handover write-up and the accessibility/error-handling pass are
finishing work, the kind that is systematically under-estimated because it is
invisible until the feature is nearly done. This is not a project running away;
it is a plan line that has been 5–10 % too low twice.

## 5. Less than planned

**Fabric Demo, −11 h (45 %).** The largest single deviation of the month, and
the second consecutive month below 60 % — July was 11/20 h (55 %). The script
flags this as `chronic-miss`, and the label is right: 20 h has now been planned
twice and delivered at roughly half twice. That is a planning pattern, not a
discipline problem. The entry dates show the mechanism — the project only gets
touched once everything else has been served, so whatever is left over is what
it gets.

**Work-hour-tracker, −6 h (80 %).** Less dramatic, and the shape is different:
the work happened throughout the month rather than being crowded to the end. The
shortfall here looks like ordinary absorption by Aerzte App rather than a
structural under-delivery.

**Six workdays without any booking:** 03.08 (Mon), 07.08 (Fri), 14.08 (Fri),
21.08 (Fri), 28.08 (Fri), 31.08 (Mon). **All four Fridays in August are among
them** — that is not scatter, that is a pattern. Whether those are deliberate
days off, vacation, or simply time that was never recorded does not follow from
the data, and it changes the next plan a lot: 22 September workdays minus four
Fridays is 18 effective days, not 22. This is the open question of the month.

## 6. Trend vs. previous month

| Project | 2026-07 | 2026-08 | Delta |
|---|---:|---:|---:|
| Aerzte App | 64.00 | 55.00 | −9.00 |
| Work-hour-tracker | 38.00 | 24.00 | −14.00 |
| Fabric Demo | 11.00 | 9.00 | −2.00 |
| **Total** | **113.00** | **88.00** | **−25.00** |

| Month | Plan | Actual | Fulfilment |
|---|---:|---:|---:|
| 2026-07 | 120.00 | 113.00 | 94.17 % |
| 2026-08 | 100.00 | 88.00 | 88.00 % |

Every project is down, and both the plan and the actual fell — the plan by 20 h,
the actual by 25 h. So the drop is not only a smaller month being planned; the
gap to the plan also widened, from 5.83 % to 12 %. Two data points are not a
trend line, but they point the same way, and the next plan is the place to stop
extrapolating it by accident.

## 7. Conclusions for the next plan

1. **Fabric Demo has been planned at twice its real size, twice.** 55 % then
   45 %. Consequence: either plan it at the ~10 h it actually receives, or keep
   20 h and give it a fixed slot that is not available to the other two
   projects. Planning 20 h again without a mechanism produces a third 45 % month.
2. **Fridays carry no bookings at all.** Consequence: plan against ~18 effective
   days for September, not 22. At August's demonstrated 5.87 h per booked day
   that is ~106 h — planning 120 h across a calendar that only has 18 working
   days in it is how the last two months were set up to miss.
3. **Aerzte App is the project that absorbs whatever is left over.** It beat its
   plan in both months while the other two missed theirs. Consequence: plan it at
   the level it actually runs at (55–60 h) instead of 50 h. Under-planning the
   project that always wins the competition for hours is what makes the other
   plan lines look like failures.
4. **The total is drifting down and the gap is widening.** Consequence: set the
   September total deliberately — as a capacity decision with a number behind it
   — rather than letting it land wherever the sum of the project lines falls.
