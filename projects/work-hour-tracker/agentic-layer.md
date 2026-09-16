# Technical details — Logging hours by just saying so

*The agent layer: three Claude Code skills, the command-line tool underneath them, and the rules that keep them honest.*

[← back to the story](../work-hour-tracker.md#logging-hours-by-just-saying-so) · [Portfolio](../../README.md)

---

## 1. Why the skills go underneath the app

The app is one way into this data. This is the other: three [Claude Code](https://claude.com/claude-code) skills that read and write the same Fabric SQL database from a terminal conversation, and for two of them produce a finished document at the end of it.

**Why they go underneath the app rather than through it.** The deployed Data API only supports interactive browser sign-in: there is no device-code flow and no service principal a command-line tool could use. So the skills sit *beneath* it and talk to Fabric SQL directly, through one 2 020-line Node CLI:

```
   Person, in Claude Code                 ┌── az login (as themselves)
        │  "log 3h on Fabric Demo today"  │   short-lived Entra token,
        ▼                                 │   no stored password
   the work-hours CLI ─────────────────────┘
        │  identity = SUSER_SNAME() from the CONNECTION, never from config
        ▼
   Fabric SQL ── filtered by row-level security
```

Two consequences, both deliberate. **Nobody can become someone else by editing a text file** — the caller's identity comes from the authenticated database connection, so colleagues share one checkout and each sees only their own hours, with no per-person setup. And because this path bypasses the app's policy engine by construction, [the database needed its own copy of the access rule](security.md).

---

## 2. The three skills at work

Every screenshot below is a real run against the demo database.

### `work-hours` — read and write

The general-purpose one. Projects, time entries and monthly plans: list, add, edit, delete, summarise by project, day, week or month, plus a fenced read-only SQL escape hatch for anything the fixed commands do not cover.

![Logging two entries in plain language and reading them back](media/skill-work-hours-01-log-entry.png)

Two things in that run matter more than the write itself.

**It reads back what it wrote**, rather than reporting success from an exit code — and the read-back is where the three-stage rate model becomes visible: Fabric Demo froze at **90 €/h**, the *personal* `ProjectRates` value, not the 80 €/h suggested on the project row. ([Why there are three rates.](data-model.md#2-money-lives-in-three-places-on-purpose))

**It reported its own side effects.** It flagged that it had added the `[DEMO]` prefix without being asked and said why, and it noticed the sandbox had drifted from its documented seed — 52 entries and 9 plan rows against a documented 50 and 6 — and named the two commands that restore it. Neither was requested. An agent with database write access that only tells you what you asked about is an agent you cannot audit.

### `monthly-statement` — the billing and statutory documents

Turns a month of time entries into four files: the billing statement, the Austrian statutory working-time record, and both as data.

**It asks before it runs, and prices every option.** A wrong VAT rate makes the whole document unusable, so it is a question rather than a default — with each answer's consequence worked out before you choose:

![The skill asking which VAT treatment to apply, each option costed out](media/skill-statement-01-vat-question.png)

**Then it checks its own work against an independent query:**

![The finished statement, cross-checked and reconciled](media/skill-statement-02-result.png)

The number that matters there is not the €9,810. It is the line beneath it — the document's totals re-derived by a **different command** (`wht.mjs summary --month`) and compared: 22 entries, 88.00 h, identical, rounding difference €0.00, all 11 internal checks green.

That is the rule the skill enforces on the model driving it: **calculate nothing by hand.** Every number in the document comes from the script's JSON output; anything missing is written in as an open point, never estimated.

#### What it produced

| Document | What to look at |
|---|---|
| **[`abrechnung-demo-2026-08.md`](samples/abrechnung-demo-2026-08.md)** | The billing statement. **Section 7** is the reason to open it: eleven plausibility checks, each stated and ticked — nothing on an Austrian public holiday, no weekend bookings, no day over 12 h, and a reconciliation proving timesheet = task = project = total at **0,00 € rounding difference**. Then seven numbered assumptions, including the honest one — *the schema has no task entity, so the description is used as the task* |
| **[`zeitaufzeichnung-demo-2026-08.md`](samples/zeitaufzeichnung-demo-2026-08.md)** | The statutory record — a *Saldenaufzeichnung* under **§ 26 Abs 3 AZG**, which permits recording only the duration worked per day. 21 working days × 4,8 h = **100,80 h target** against **88,00 h actual**, balance **−12,80 h** to payroll. Its notes flag that one day exceeded six hours, obliging a § 11 break that a balance record does not capture |
| **[`timesheet-demo-2026-08.csv`](samples/timesheet-demo-2026-08.csv)** · **[`zeitaufzeichnung-demo-2026-08.csv`](samples/zeitaufzeichnung-demo-2026-08.csv)** | The same content as data. German-Excel compatible — UTF-8 with BOM, semicolons, decimal commas, CRLF |

Both formats of the working-time record render from **one** model, so they cannot disagree. **Markdown and CSV only — never HTML or PDF**: one renderer per document means there is no second generator to drift out of step with the first.

### `month-planning` — review, then questions, then plan

Three steps in a fixed order, and the order is the design. **The questions come after the review, because the review is what makes them answerable** — asking first is asking into the dark.

The script computes every number: working days, deviations, fulfilment rates, trends, and its own warnings. The narrative half — *what was actually achieved* — is written from the notes on the entries, and that half is the model's job.

Then four questions, and **every option carries the number behind it**:

![Capacity: how many hours to plan, each option derived from a different reading of the data](media/skill-planning-01-capacity.png)

![An anomaly the script found on its own — four Fridays with no bookings — turned into a question](media/skill-planning-04-fridays.png)

That second one is the interesting one. Nobody asked it to look for empty weekdays; the script flags `unbooked-workdays` by itself, and the answer changes the arithmetic of the entire plan — four non-working Fridays means September has **18 effective days, not 22**, and every line gets sized against 18.

<details>
<summary>The other two questions — a chronically missed project, and where the focus should go</summary>

![A project under 60% of plan two months running, and four ways to respond](media/skill-planning-02-chronic-miss.png)

![Where the focus should go, with each project's two-month history attached](media/skill-planning-03-focus.png)

</details>

**Writing the plan back**, with a seatbelt: `--total` is the expected sum, and if the distribution does not add up to it, **nothing is written at all**. Afterwards the rows are read back out of the database and compared.

![The finished plan, written and read back, both warnings closed](media/skill-planning-05-result.png)

It also did two things it was not asked to do, and *announced* both rather than performing them quietly: it pushed back on the focus choice, pointing out that the chosen project had won every contest for leftover hours in both prior months; and it recorded the evidence for how the hours should be shaped — two six-hour blocks produced real work in August while four separate two-hour slots did not, so the plan assumes two full days a week rather than 2.8 hours spread daily. **An agent that quietly adjusts your numbers is worse than one that refuses.**

#### What it produced

| Document | What to look at |
|---|---|
| **[`planungsreview-demo-2026-08.md`](samples/planungsreview-demo-2026-08.md)** | The review. **Section 3** — *what was achieved* — is the half written from the entry notes rather than computed, and it is where the split between script and model is easiest to see: it reads eleven entries and concludes a feature went from working to finished, which is not something a query can say |
| **[`monatsplan-demo-2026-09.md`](samples/monatsplan-demo-2026-09.md)** | The plan that came out of it and was written to the database — including the two objections above, recorded as flags rather than applied as edits |

All six documents, with their framing: **[samples/ →](samples/)**

---

## 3. Identity belongs to the connection, not to the configuration

The first version read *"who am I"* from a config file. That does not fail loudly when it is wrong. It quietly shows you somebody else's earnings, formatted perfectly.

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

The configured value was not deleted. It was **demoted to a guard rail**. If it is set and disagrees with the connection, the tool refuses to run at all rather than showing anything. Normally it is left blank and the tool detects you.

`SUSER_SNAME()` returns the caller's real Entra UPN, which is exactly what the `user_email` columns hold, so the CLI and [the database's row-level security](security.md) agree on what an identity *is* with no mapping table in between.

**The general rule: derive identity from the authenticated channel, never from configuration.** Configuration is a suggestion; a connection is a fact. There is no field anyone can fill in wrong, and no per-person setup step to forget.

The same instinct produced `whoami`, which prints the resolved identity, the workspace, the database and how many rows you own. It is the first thing to run whenever a number looks wrong.

---

## 4. The command surface

```
SETUP   setup · setup --check · whoami · grant-user · rls --status/--apply/--drop
READ    projects · hours · summary --by project|day|week|month · plans
        query "<SELECT …>"                      ← fenced, see below
WRITE   add-project · add-entry · set-plan (upsert) · delete-entry
EDIT    edit-entry --id · edit-project
PERIODS --today --yesterday --week --last-week --month YYYY-MM --from/--to
```

**Edit is always preferred to delete-and-re-add.** Re-adding an entry loses the id, the notes, and — worst — the historically frozen rate, silently repricing old work at today's rate.

`setup --check` walks the whole chain (az login → config → driver → token → connectivity → row ownership) and reports where it breaks. Every write supports `--dry-run`, prints before → after, reports the row count it actually affected, and fails loudly on zero — so *"it worked"* is never inferred.

---

## 5. The escape hatch, and how it is fenced

`query "<SELECT …>"` lets a conversation ask something the fixed commands do not cover. It is the most dangerous surface in the tool, so it is fenced three ways.

The first version checked `/^\s*select\b/i`, which only inspects how the string *starts*. The driver executes batches, so `SELECT 1; DELETE FROM TimeEntries` sailed straight through.

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

**And it is documented as what it is:** a guard rail against accidents, *not* a security boundary. A genuine boundary would be a read-only database principal.

---

## 6. What the skills instruct the model to do

A skill is not only a wrapper around a CLI; it is a set of standing instructions, and these three are mostly rules about what the model may *not* do.

**Calculate nothing by hand.** Every number in a generated document must come from the script's JSON output. Anything missing is written into the document as an open point, never estimated. So the statement skill re-derives its own totals with a second, independent command before reporting, and the [published statement](samples/abrechnung-demo-2026-08.md) carries that reconciliation in section 7.

**Review before questions, questions before writing.** The planning skill fixes the order because the review is what makes the questions answerable.

**Flag, do not adjust.** Where the data argues against the choice a person made, that goes into the document as a recorded objection, not a quiet correction to the numbers.

**Confirm the target before writing.** More than one workspace holds a database with the same display name, so `whoami` comes before anything that writes.

**Ask when a parameter would change the document materially.** A wrong VAT rate makes the whole statement unusable, so it is a question with each option's consequence costed out — not a default that quietly applies.

---

## 7. What I would do differently

- **A read-only database principal for `query`.** Three guards honestly labelled *"not a security boundary"* are still not a security boundary.
- **The tool should print its target before every write, not only when asked.** `whoami` exists and is documented as the first thing to run — but documenting a check is not the same as running it.
- **Skills that share a data source should share a test fixture.** Three skills whose totals must agree, and nothing automated that proves they do. The cross-check in the statement is performed by the model, per instruction — it should be performed by a test.

[← back to the story](../work-hour-tracker.md#logging-hours-by-just-saying-so) · [Sample output](samples/) · [Portfolio](../../README.md)
