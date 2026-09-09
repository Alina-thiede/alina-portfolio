# The agent layer — design detail

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)

> **The walkthrough is on the main page.** [Claude Code skills](../work-hour-tracker.md#claude-code-skills) shows all three skills running, with screenshots and the documents they produced. This page is the part that did not fit there: the identity rule, the command surface, the read-only fence, and the instructions the skills give the model.

---

## 1. Identity belongs to the connection, not to the configuration

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

`SUSER_SNAME()` returns the caller's real Entra UPN, which is exactly what the `user_email` columns hold — so the CLI and [the database's row-level security](security.md) agree on what an identity *is* with no mapping table in between.

**The general rule: derive identity from the authenticated channel, never from configuration.** Configuration is a suggestion; a connection is a fact. There is no field anyone can fill in wrong, and no per-person setup step to forget.

The same instinct produced `whoami`, which prints the resolved identity, the workspace, the database and how many rows you own — the first thing to run whenever a number looks wrong, and the command that would have caught [the incident](data-incident.md) on day one instead of day eight.

---

## 2. The command surface

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

## 3. The escape hatch, and how it is fenced

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

## 4. What the skills instruct the model to do

A skill is not only a wrapper around a CLI — it is a set of standing instructions, and these three are mostly rules about what the model may *not* do.

**Calculate nothing by hand.** Every number in a generated document must come from the script's JSON output. Anything missing is written into the document as an open point, never estimated. A report where the model did the arithmetic is a report nobody can check — so the statement skill re-derives its own totals with a second, independent command before reporting, and the [published statement](samples/abrechnung-demo-2026-08.md) carries that reconciliation in section 7.

**Review before questions, questions before writing.** The planning skill fixes the order because the review is what makes the questions answerable. Asking first is asking into the dark.

**Flag, do not adjust.** Where the data argues against the choice a person made, that goes into the document as a recorded objection, not a quiet correction to the numbers.

**Confirm the target before writing.** Both workspaces hold a database with the same display name, so `whoami` comes before anything that writes. That rule was bought at the price of [eight days of divergence](data-incident.md).

**Ask when a parameter would change the document materially.** A wrong VAT rate makes the whole statement unusable, so it is a question with each option's consequence costed out — not a default that quietly applies.

---

## 5. What I would do differently

- **A read-only database principal for `query`.** Three guards honestly labelled *"not a security boundary"* are still not a security boundary.
- **The tool should print its target before every write, not only when asked.** `whoami` exists and is documented as the first thing to run. Eight days of divergence say that documenting it is not the same as doing it.
- **Skills that share a data source should share a test fixture.** Three skills whose totals must agree, and nothing automated that proves they do. The cross-check in the statement is performed by the model, per instruction — it should be performed by a test.

[← Work hour tracker](../work-hour-tracker.md) · [Sample output](samples/) · [Portfolio](../../README.md)
