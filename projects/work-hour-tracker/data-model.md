# Technical details — A raise shouldn't rewrite last year

*The data model: six tables, and why the money lives in three places.*

[← back to the story](../work-hour-tracker.md#a-raise-shouldnt-rewrite-last-year) · [Portfolio](../../README.md)

---

## 1. Six tables

Every data table carries the same four owner columns — `user_email`, `user_name`, `user_first_name`, `user_last_name` — written together on each insert.

| Table | Grain | Key columns |
|---|---|---|
| `Projects` | one row per project, **shared** — its `user_email` is a *created by* label, not an owner | `id`, `name`, `hourlyRate` *(suggested)*, `color` |
| `TimeEntries` | one row per person per project per logged block | `id`, `date`, `hours`, `hourlyRate` *(frozen copy)*, `notes`, `project_id` |
| `ProjectRates` | what one person charges on one project | `id`, `hourlyRate`, `project_id` |
| `ProjectShares` | one project made visible to one other person | `id`, `project_id`, `shared_with` *(the invitee)*, `user_email` *(the sharer)* |
| `ProjectMonthlyPlans` | one row per person per project per month | `id`, `month` (`YYYY-MM`), `plannedHours`, `project_id` |
| `UserProfiles` | one row per person | source of truth for their real name; doubles as the team roster |

None of those grains is a database constraint — the platform offers no `unique` option — so each is enforced in application code. Saving your rate, for example, is an upsert rather than an insert, because nothing in the database stops a second `ProjectRates` row for the same person and project.

---

## 2. Money lives in three places, on purpose

| Where | What it means |
|---|---|
| `Projects.hourlyRate` | a *suggested* rate — the default until you set your own |
| `ProjectRates.hourlyRate` | what one person actually charges on one project |
| `TimeEntries.hourlyRate` | the rate **frozen** onto the entry at the moment it was logged |

The third one is the reason for the chapter title. A time entry does not look its rate up when it is read; it carries a copy of the rate that applied when it was written:

```ts
@entity()
export class TimeEntry {
  @uuid()    id!: string;
  @date()    date!: Date;
  @decimal({ optional: true }) hours?: number;

  // The rate that applied WHEN THIS ENTRY WAS LOGGED, in €/hour.
  // Looking it up live was decided against: giving yourself a raise
  // today would silently rewrite what you earned last year.
  @decimal({ optional: true }) hourlyRate?: number;

  @text({ max: 1000, optional: true }) notes?: string;

  @uuid() project_id!: string;
  @one(() => Project) project?: Project;

  @text({ max: 320, optional: true }) user_email?: string;
}
```
*Abridged from `rayfin/data/TimeEntry.ts` — the access decorators ([security](security.md#2-layer-1--the-compiled-policy)), the three display-name columns and three legacy columns are left out.*

Copying the rate at write time freezes history, and as a side effect the dashboard needs no join at all. The cost is that every write path must stamp it. Three places a rate can live means three chances to disagree, so everything that reads or writes one goes through a single file:

```ts
/** What the signed-in user charges on this project today — used to stamp new entries. */
export function effectiveRate(project: RatedProject): number {
  return project.myRate ?? project.suggestedRate ?? 0;
}

/** What an existing entry actually earned per hour — used for every total. */
export function entryRate(entry: {
  hourlyRate?: number | null;
  project?: { hourlyRate?: number | null } | null;
}): number {
  const stamped = Number(entry.hourlyRate ?? NaN);
  if (Number.isFinite(stamped)) return stamped;
  return Number(entry.project?.hourlyRate ?? 0) || 0;
}
```
*From `src/lib/rates.ts`, doc comments shortened.*

Two details in there carry weight:

- **The fallback chain in `effectiveRate()` is what makes a shared project work on day one.** No `ProjectRates` row of your own means you inherit the project's suggestion rather than earning nothing.
- **The fallback in `entryRate()` is what made the change safe without a data migration.** Entries written before the frozen column existed fall back to the project's rate — which, back when every project belonged to exactly one person, *was* that person's rate. The command-line tool reads earnings the same way, as `COALESCE(t.hourlyRate, p.hourlyRate)`.

The only event that re-stamps a frozen rate is moving an entry to a different project, because the old copy is then the wrong project's money:

```ts
...(projectChanged && target ? { hourlyRate: effectiveRate(target) } : {}),
```
*From the edit form in `src/pages/TimeEntries.tsx`.*

**Rejected:** joining to the current rate at read time. A raise must not rewrite invoiced history; the duplication is the point.

---

## 3. A project is shared, a rate is personal

The first version had one rate column on the project. That made sharing a project share the money too, and two people could not work the same engagement at different rates. So the rate moved out into `ProjectRates`, one row per person per project, readable only by its owner and the manager.

Sharing itself is a join table rather than a list of addresses on the project. That is not taste. A policy can only compare the caller's claims against a column **on the same row** — no `contains`, no subquery, no join — so `shared_with: "a@x;b@x"` could never be checked by the backend. One row per person turns it into a question the backend *can* answer:

```ts
@entity()
// Read: I shared it, it was shared with me, or I am the manager.
@authenticated('read', {
  policy: (claims, item) =>
    ownerOrManager(claims, item.user_email).or(claims.email.eq(item.shared_with)),
})
// Write: only the person who granted the share.
@authenticated(['create', 'update', 'delete'], {
  policy: (claims, item) => ownerOnly(claims, item.user_email),
})
export class ProjectShare {
  @uuid() id!: string;
  @uuid() project_id!: string;
  @text({ max: 320 }) shared_with!: string;                 // the colleague
  @text({ max: 320, optional: true }) user_email?: string;  // the sharer
}
```
*Abridged from `rayfin/data/ProjectShare.ts`.*

A share needs two different people to touch the same row: the sharer must *create* it, naming somebody else, and the invitee must *read* it to discover the project exists. Read matches either address; write matches only the sharer. So you can hand a colleague access and take it away again, and nobody can grant themselves access to your project by writing a row that says you did.

The colleague's address is typed in by hand rather than picked from a list — forced, because an employee can only read their *own* profile row, so there is no directory to offer. A typo therefore shares with nobody rather than with the wrong person, which is the safe direction to fail.

**The data model was chosen by what the security layer can express.** What that same limit costs — project *names* stay readable to every signed-in user — is covered in [security](security.md#4-the-two-exceptions-stated-rather-than-hidden).

---

## 4. One identity, stamped on every row

**`user_email` is the only real identity.** The name columns are display text — blank on older rows, and two people can derive the same name. Every read filters, groups and joins on the email.

The real name comes from `UserProfiles`, not from the sign-in: the platform's session exposes only `{ id, email, role }`, with no name claim. The profile is loaded once at sign-in and cached, so stamping stays a plain synchronous call.

Two helpers in `src/lib/user.ts` keep the owner columns honest:

- **`ownerColumns(email)`** is spread into all seven `.create()` calls in the app, so no write site can remember one owner column and forget the others.
- **`ownedBy(email)`** is its read-side twin, and every read that means *"mine"* must use it. The read policy is *"mine, or I am the manager"*, so for the manager an unfiltered read returns the whole team — and their own dashboard would quietly total everyone's hours and present them as theirs. No error, no crash, just wrong numbers that look plausible.

---

## 5. Columns that cannot be removed

Almost every column added after the first deploy is optional, because MSSQL refuses to add a `NOT NULL` column to a table that already has rows.

Some columns are no longer used at all and are still there: the legacy `Projects.hourlyRate`, and three clock-time columns on `TimeEntries` from when the app asked for a start and end time instead of hours. Dropping a column is a destructive migration, `rayfin up` refuses those, and one refused drop blocks every other change in the same deploy.

The practical consequence: on this platform, every column you add is, in effect, permanent. The schema carries its own history.

[← back to the story](../work-hour-tracker.md#a-raise-shouldnt-rewrite-last-year) · [Portfolio](../../README.md)
