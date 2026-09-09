# The agent layer — three skills on one CLI

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)

The app answers *"log three hours on Fabric Demo"* in a browser. The agent layer answers it in a terminal conversation, from inside the editor where the work is already happening — and answers the two questions a browser is bad at: *"write me the month's billing statement"* and *"what did I actually get done, against what I planned?"*

> Every screenshot on this page was taken against the **demo database**, whose every row is invented and whose every time-entry note begins with `[DEMO]`.

---

## 1. Why it goes underneath the API, not through it

The deployed Data API only supports interactive browser sign-in. There is no non-interactive flow a command-line tool could use — no device code, no service principal.

So the CLI goes **beneath** it, straight to Fabric SQL:

```
   Person, in Claude Code                 ┌── az login (as themselves)
        │  "log 3h on Fabric Demo today"  │   short-lived Entra token,
        ▼                                 │   no stored password
   the work-hours CLI  ────────────────────┘
        scripts/wht.mjs — 1 790 lines, mssql driver
        │
        │  identity = SUSER_SNAME() from the CONNECTION, never from config
        ▼
   Fabric SQL  ── filtered by row-level security (layer 2)
```

That is exactly why the database needed [its own copy of the access rule](security.md). Bypassing the policy engine is a design decision with a consequence, and the consequence is a second enforcement layer.

**No stored password anywhere.** Each person runs `az login` as themselves, the token is minted per run and expires. Colleagues share one checkout and each sees only their own hours.

---

## 2. Identity belongs to the connection, not to the configuration

The first version read *"who am I"* from a config file. That does not fail loudly when it is wrong — it quietly shows you somebody else's earnings, formatted perfectly.

```js
/** The caller's email, as the database sees it. The one source of "who am I". */
async function me() {
  if (cachedMe) return cachedMe;
  const rows = await query('SELECT SUSER_SNAME() AS email');
  const email = rows[0]?.email;
  if (!email) die('The database did not report a login name for this connection.');

  // A stale configured email used to mean silently reading the wrong person's
  // rows. Now it is a hard stop that names both sides.
  if (CONFIGURED_EMAIL && CONFIGURED_EMAIL.toLowerCase() !== email.toLowerCase()) {
    die('Identity mismatch — the connection wins. Remove WHT_USER_EMAIL from .env.');
  }
  return (cachedMe = email);
}
```
*Abridged from `wht.mjs`.*

The configured value was not deleted — it was **demoted to a guard rail**. If it is set and disagrees with the connection, the tool refuses to run at all rather than showing anything. Normally it is left blank and the tool detects you.

**The general rule: derive identity from the authenticated channel, never from configuration.** Configuration is a suggestion; a connection is a fact. There is no field anyone can fill in wrong and no per-person setup step to forget.

The same instinct produced `whoami`, which prints the resolved identity, the workspace, the database and how many rows you own — the first thing to run whenever a number looks wrong, and the command that would have caught [the incident](data-incident.md) on day one instead of day eight.

---

## 3. `work-hours` — read and write

```
SETUP   setup · setup --check · whoami · grant-user · rls --status/--apply/--drop
READ    projects · hours · summary --by project|day|week|month · plans
        query "<SELECT …>"                      ← fenced, see below
WRITE   add-project · add-entry · set-plan (upsert) · delete-entry
EDIT    edit-entry --id · edit-project
PERIODS --today --yesterday --week --last-week --month YYYY-MM --from/--to
```

**Edit is always preferred to delete-and-re-add.** Re-adding an entry loses the id, the notes, and — worst — the historically frozen rate, silently repricing old work at today's rate.

![Logging two entries and reading them back](media/skill-work-hours-01-log-entry.png)

Two things in that run are worth more than the write itself.

**It reads back what it wrote,** rather than reporting success from an exit code — and the read-back is where the three-stage money chain becomes visible: Fabric Demo froze at **90 €/h**, the *personal* `ProjectRates` value, not the 80 €/h suggested on the project row. That is the model working, demonstrated rather than asserted.

**It reported its own side effects.** It flagged that it had added the `[DEMO]` prefix without being asked and explained why, and it noticed that the sandbox had drifted from its documented seed — 52 time entries and 9 plan rows against a documented 50 and 6 — and named the two commands that restore it. Neither was requested. An agent with database write access that only tells you what you asked about is an agent you cannot audit.

### The escape hatch, and how it is fenced

`query "<SELECT …>"` lets a conversation ask something the fixed commands do not cover. It is the most dangerous surface in the tool, so it is fenced three ways.

The first version checked `/^\s*select\b/i` — which only inspects how the string *starts*. The driver executes batches, so `SELECT 1; DELETE FROM TimeEntries` sailed straight through.

```js
const BANNED_SQL = /\b(insert|update|delete|drop|alter|truncate|merge|create|grant|revoke|exec|execute|backup|restore|shutdown)\b|\b(sp_|xp_)\w*/i;

function assertReadOnly(text) {
  const stripped = text
    .replace(/\/\*[\s\S]*?\*\//g, ' ')  // block comments
    .replace(/--[^\n]*/g, ' ')          // line comments
    .replace(/'(?:''|[^'])*'/g, "''")   // string literals
    .trim().replace(/;+\s*$/, '');      // one harmless trailing semicolon

  if (stripped.includes(';'))              die('single statement only — remove the ";".');
  if (!/^(select|with)\b/i.test(stripped)) die('read-only — SELECT or WITH … SELECT only.');
  const hit = stripped.match(BANNED_SQL);
  if (hit)                                 die(`read-only — "${hit[0]}" is not allowed.`);
  return stripped;
}
```

Then, regardless of what the parser concluded:

```js
await pool.request().query(`BEGIN TRANSACTION;\n${safe};\nROLLBACK TRANSACTION;`);
```

Anything that slips the parser still cannot persist.

**And it is documented as what it is:** a guard rail against accidents, *not* a security boundary. A genuine boundary would be a read-only database principal. Writing that sentence into the source is the difference between a known limitation and a false sense of safety.

---

## 4. `monthly-statement` — the billing and statutory documents

Four files per person per month: the billing statement, the Austrian statutory *Zeitaufzeichnung* (§ 26 Abs 3 AZG) in the authoritative Markdown format, the same record as data, and a raw timesheet CSV.

**It asks before it runs, and every option carries its consequence.**

![The skill asking which VAT treatment to apply, with the resulting figures for each option](media/skill-statement-01-vat-question.png)

A wrong VAT rate makes the entire document unusable, so it is a question rather than a default — and each option is priced out before you choose, so the choice is informed rather than guessed. The same pattern applies to the contracted weekly hours that the statutory sheet balances against.

**Then it checks its own work.**

![The finished statement, cross-checked against an independent query](media/skill-statement-02-result.png)

The number that matters there is not the €9,810. It is the line beneath it: the document's totals were re-derived by a **different command** (`wht.mjs summary --month`) and compared — 22 entries, 88.00 h, identical, rounding difference €0.00, and all 11 internal checks green.

That is the skill's central instruction to the model: **calculate nothing by hand.** Every number in the report must come out of the script's JSON output. If something is missing it is written into the document as a TODO, never estimated. A report where the model did the arithmetic is a report nobody can check.

Both formats of the working-time record render from **one** model, so they cannot drift. Both CSVs are German-Excel compatible — UTF-8 with BOM, semicolons, decimal commas, CRLF. The script is idempotent: the same call produces byte-identical files.

**Markdown and CSV only — never HTML or PDF.** One renderer per document. If a print version is ever needed it gets converted from the Markdown at that moment, rather than a second generator existing permanently to fall out of step with the first.

**→ Read the documents themselves:** [`abrechnung-demo-2026-08.md`](samples/abrechnung-demo-2026-08.md) and [`zeitaufzeichnung-demo-2026-08.md`](samples/zeitaufzeichnung-demo-2026-08.md), with the CSVs, are in [samples/](samples/). Section 7 of the statement is the one to read — eleven checks and seven stated assumptions.

---

## 5. `month-planning` — review, then questions, then plan

Three steps in a fixed order. **The questions come after the review, because the review is what makes them answerable** — asking first is asking into the dark.

The script computes the facts: working days, deviations, fulfilment rates, trends, and its own warnings. The narrative half — *what was actually achieved* — comes from the notes on the entries and is the model's job, not the script's.

Then four questions, and **every option carries the number behind it**:

![Capacity: how many hours to plan for the coming month, each option derived from a different reading of the data](media/skill-planning-01-capacity.png)

![A project that has missed its plan two months running, and four ways to respond](media/skill-planning-02-chronic-miss.png)

![Where the focus should go, with each project's two-month history attached](media/skill-planning-03-focus.png)

![An anomaly the script found on its own — four Fridays with no bookings — turned into a question](media/skill-planning-04-fridays.png)

That fourth one is the interesting one. Nobody asked it to look for empty weekdays; the script flags `unbooked-workdays` on its own, and the answer changes the arithmetic of the entire plan — four non-working Fridays means September has **18 effective days, not 22**, and every line gets sized against 18.

### Writing the plan back

![The finished plan, written to the database and read back, with both warnings closed](media/skill-planning-05-result.png)

```bash
node scripts/monatsplanung.mjs --plan-month 2026-09 \
  --set "work-hour-tracker=50,aerzte-app=46,fabric-demo=10" --total 106
```

`--total` is the seatbelt: **if the distribution does not add up to it, nothing is written at all.** After writing, the rows are read back out of the database and compared — *"mismatch was empty, read-back total 106.00 h, exit 0"*.

And it did two things it was not asked to do, both of which it announced rather than performed quietly: it pushed back on the focus choice, pointing out that the chosen project had won every contest for leftover hours in both prior months and that planning it *below* its actual only holds if a handover really ended the work; and it recorded, in the plan document itself, the evidence for how the hours should be shaped — two six-hour blocks produced real work in August while four separate two-hour slots did not, so the plan assumes two full days a week rather than 2.8 hours spread daily.

**An agent that quietly adjusts your numbers is worse than one that refuses.** Both of those went into the document as flags, not edits.

**→ Read the documents themselves:** [`planungsreview-demo-2026-08.md`](samples/planungsreview-demo-2026-08.md) is the review, [`monatsplan-demo-2026-09.md`](samples/monatsplan-demo-2026-09.md) is the plan that came out of it. Section 3 of the review — *what was achieved* — is the half written from the entry notes rather than computed, and it is where the split between script and model is easiest to see.

---

## 6. What I would do differently

- **A read-only database principal for `query`.** Three guards honestly labelled *"not a security boundary"* are still not a security boundary.
- **The tool should print its target before every write, not only when asked.** `whoami` exists and is documented as the first thing to run. Eight days of divergence say that documenting it is not the same as doing it. → [the incident](data-incident.md)
- **Skills that share a data source should share a test fixture.** Three skills whose totals must agree, and nothing automated that proves they do. The cross-check in section 4 is done by the model, per instruction — it should be done by a test.

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)
