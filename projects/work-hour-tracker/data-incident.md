# The incident — eight days of two databases

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)

A migration that was verified three separate ways, and still went wrong — because the thing that broke was not the thing that was verified.

---

## 1. What happened

The app was moved from one Microsoft Fabric workspace to another: new region, new capacity, new SQL database.

`rayfin up` into a new workspace creates the item, the database and every table — **empty**. No supported Fabric mechanism brings the rows along; deployment pipelines and git integration carry item metadata only, and the backend items have no readable definition over the items API. So the rows had to be read over SQL, written over SQL, and then the result had to be *proved*. That tooling was written, run, and verified three ways (§3).

The migration itself was fine.

What was not noticed: one global config file — the one the command-line tooling reads to find its database — still pointed at the **old** workspace.

So for eight days:

```
   the APP  ─────────────────────────────►  NEW database   (production)
   the CLI and both report scripts  ─────►  OLD database
```

Both sides accumulated real rows. Hours logged in the browser landed in one; hours logged from the terminal, and the monthly plans written by the planning script, landed in the other.

**Neither side was a subset of the other.**

Worse: for those eight days the old workspace was believed to be a frozen rollback copy, and was under a written rule saying it must not be written to. It was being written to the entire time, by the person who wrote the rule.

---

## 2. Why the safety net was not one

The migration tooling had produced snapshots at every step, and the documentation described those snapshots as the cold backup.

They were not. **A snapshot stored column signatures, row counts, SHA-256 hashes per table and a control pair — no rows.**

That is a perfectly good integrity check. It could prove that two databases differed, and it did. It could not put anything back. By the time the divergence was found, the real hours existed in exactly one restorable place: the databases themselves.

> **An integrity check is not a backup.** They answer different questions — *"are these the same?"* versus *"can I get this back?"* — and a document that calls one the other is worse than a document that mentions neither, because it stops anyone from going to look for the real one.

A separate tool now writes the actual rows to a file with a hash beside it, and the destructive scripts refuse to run unless a matching export exists whose per-table hashes equal the live database. That check catches both *"I forgot to export"* and *"the export is older than the last few writes"*.

---

## 3. How it was proved, before and after

The verifier runs three proofs between two snapshots. The reason there are three matters more than the mechanism:

| Proof | What it compares |
|---|---|
| **Data** | every row, rendered to text *by SQL Server* in a form that does not depend on declared column width, sorted so no engine's row order can matter, SHA-256 per table |
| **Schema** | every column's declared type, length, precision and nullability — the half proof 1 deliberately ignores, which is what keeps proof 1 free of false alarms |
| **Control pair** | two numbers per person, computed the way a human reads them, **written down before the first write** and recomputed afterwards |

**Three, because a copier and a verifier that share a canonical form can agree perfectly about the wrong data.** The control pair is computed by a different query, over different columns, and was recorded in advance — so it cannot be talked into agreeing.

---

## 4. The reconciliation

The obvious move — run the copier again, old to new — would have made things worse. Two reasons, and both are why this needed a different tool rather than a second run:

**Content duplicates under different ids.** Four monthly-plan rows existed on *both* sides with identical content and different primary keys: that month's plan had been written once by the app into the new database, and once by the planning script into the old one. A copier that matches on primary key sees four rows the target lacks and inserts them — **doubling that month's planned hours**.

**Drift under a shared id.** One plan row existed on both sides with the *same* id and different content: 40 hours on one side, 0 on the other. A copier that skips primary keys the target already holds is blind to it.

So the merge tool asks a narrower question, per table: *is this row's content already there?*

```js
/**
 * Tables whose rows carry a natural identity beyond the primary key, used to
 * recognise a content duplicate that was written twice under two different ids.
 */
const NATURAL_KEY = {
  ProjectMonthlyPlans: ['user_email', 'month', 'project_id'],
};
```

One person plans one project once per month. A candidate row whose triple is already present is a duplicate **whatever its id says**, and is skipped. Everything else matches by primary key as usual.

Rows present on both sides with the same id and different content are **reported always and written only behind an explicit flag** — because *"the old database is newer"* was true for that one row and is not a rule: a row edited in the app after the divergence would be newer in production. The diff gets read by a person before anything is written.

Direction is hardcoded in the source. There is no flag that makes production the source.

**Result: 8 rows existed only in the old database. All 8 were merged into production, the result was proved with all three checks, and the old workspace was then emptied and re-seeded with invented demo data** — which is the dataset every screenshot in this write-up was taken from. Production is now the only place real hours live, and the old workspace is a sandbox whose every row is fabricated.

---

## 5. The two rules that came out of it

### Anything that resolves a resource by name must print what it picked

Both workspaces contain a SQL database with the *same display name*. Any discovery by name can land on either one. That is not a bug in Fabric — it is what names are.

`whoami` now prints the workspace and the database, and it is the first thing to run before trusting any number or issuing any write. The setup command refuses to run without an explicit workspace: without one it scans every workspace the caller can see and takes the first item with a matching name.

The same rule was applied to the demo tooling: the wipe and seed scripts are **hardcoded** to the sandbox, and passing a workspace argument is *rejected* rather than ignored. Passing the production id says so by name.

### The write freeze is the whole mitigation, and it will not hold

Two real time entries were logged in the app *while the migration was running*. Nothing was lost — the copier reads live and is idempotent, so it picked both up and re-running cost nothing — but the frozen baseline went stale twice and had to be re-taken with the delta accounted for row by row.

If a freeze is the plan: stop the writes first, and re-snapshot immediately before the final comparison. A freeze that is only announced is not a freeze.

---

## 6. What it actually cost

An evening of reconciliation, and eight days during which any number either tool reported was some fraction of the truth — including a monthly review generated in the middle of it.

Nothing was lost, and that is partly because the copier happened to be idempotent and the divergence happened to be small. It was recoverable by good luck as much as by good design. The tooling that exists now is what turns that into a repeatable answer instead of a story.

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)
