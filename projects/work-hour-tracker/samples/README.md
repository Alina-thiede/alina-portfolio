# Sample output

[← Work hour tracker](../../work-hour-tracker.md) · [The agent layer](../agentic-layer.md) · [Portfolio](../../../README.md)

These are the actual documents the app's tooling produces, for one person and one month. Nothing here was written by hand: each file is the output of a script that reads the database, and each is idempotent — the same call produces a byte-identical file.

> **Every figure is invented.** These were generated against the demo database, where every time entry note begins with `[DEMO]` — a prefix you can see running down the middle of the billing statement and the timesheet. The rates (90, 100, 120 €/h) are round numbers no real engagement uses. **Two edits were made before publishing:** the tax advisor's name was replaced with "die Steuerberatung", and one link to a file that lives only in the private repository was removed. Nothing else was touched.

---

## The billing side

| File | What it is |
|---|---|
| **[abrechnung-demo-2026-08.md](abrechnung-demo-2026-08.md)** | The monthly billing statement, seven sections. German. |
| **[timesheet-demo-2026-08.csv](timesheet-demo-2026-08.csv)** | One row per time entry — the raw data behind the statement. |

**Read section 7 first.** It is the part that matters, and the part most generated documents do not have: eleven plausibility checks, each stated and each ticked (no entry over 16 h, no weekend bookings, nothing on an Austrian public holiday, no overlapping periods, and a reconciliation proving that the timesheet, the task level, the project level and the grand total all agree to **0,00 € rounding difference**). Then seven numbered assumptions — including the honest one, **A1**: *the schema has no task entity, so the entry description is used as the task*. And then the open points, which say what should change in the data model rather than papering over it.

That is the discipline the skill enforces on the model driving it: **calculate nothing by hand.** Every number in the document comes from the script's JSON output. Anything missing is written in as an open point, never estimated.

Section 3 is where the [three-stage rate model](../data-model.md#2-money-lives-in-three-places-on-purpose) becomes visible: three projects, three different frozen rates, and the statement bills each hour at the rate that applied when it was logged rather than at today's rate.

## The statutory side

| File | What it is |
|---|---|
| **[zeitaufzeichnung-demo-2026-08.md](zeitaufzeichnung-demo-2026-08.md)** | The Austrian statutory monthly working-time record. German. |
| **[zeitaufzeichnung-demo-2026-08.csv](zeitaufzeichnung-demo-2026-08.csv)** | The same table as data. |

A *Saldenaufzeichnung* under **§ 26 Abs 3 AZG**: Austrian law lets an employer record only the **duration** worked each day rather than start and end times, provided the balance is kept. So there are no clock times in it — that absence is the legal form, not missing data.

The arithmetic it has to get right: a daily target derived from the contracted week, every Monday-to-Friday counted whether or not it was worked, holiday and sick days credited as *fulfilled* so a lawful absence cannot push the month negative, and a closing balance that goes to payroll. Here: 21 working days × 4,8 h = **100,80 h target** against **88,00 h actual**, balance **−12,80 h**.

The notes underneath it do the thing a compliance document should — they flag that one day exceeded six hours, which under **§ 11 AZG** obliges a 30-minute break that a balance record does not capture, so the hours entered must be understood as excluding it. And the closing note says a growing negative balance *belongs discussed, and not quietly deducted from salary*.

**Both formats render from one model**, so the Markdown and the CSV cannot disagree. The CSV is German-Excel compatible: UTF-8 with BOM, semicolons, decimal commas, CRLF.

## The planning side

| File | What it is |
|---|---|
| **[planungsreview-demo-2026-08.md](planungsreview-demo-2026-08.md)** | The review of the month that ended. English. |
| **[monatsplan-demo-2026-09.md](monatsplan-demo-2026-09.md)** | The plan for the month that follows, as written to the database. |

These two are the documents behind [the planning screenshots](../agentic-layer.md#month-planning--review-then-questions-then-plan). The order is fixed and deliberate — **review, then questions, then plan** — because the review is what makes the questions answerable.

The split of labour is the interesting part. The script computes every number: working days, deviations, fulfilment rates, trends, and its own warnings (`chronic-miss` for a project under 60 % twice running, `unbooked-workdays` for empty weekdays). The narrative — *what was actually achieved* — is written from the notes on the entries, and that half is the model's job. Section 3 of the review is that half: it reads eleven entries and concludes the feature went from working to finished, which is not something a query can say.

The plan document records two things it was told to flag rather than quietly apply, both of which argue against the choice that was made. An agent that silently adjusts your numbers is worse than one that refuses.

---

## Why there is no PDF

The tooling generates Markdown and CSV only — never HTML, never PDF. One renderer per document, so there is no second generator to fall out of step with the first. If a print version is ever needed, the Markdown gets converted at that moment.

PDF copies of two of these files do exist locally. They are deliberately **not** published here, because shipping them beside a paragraph explaining why the project does not produce PDFs would be a contradiction a reader could see.

[← Work hour tracker](../../work-hour-tracker.md) · [The agent layer](../agentic-layer.md) · [Portfolio](../../../README.md)
