# Security — one rule, enforced twice

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)

**The rule, in one sentence:** you may read your own rows, plus everyone's if you are the manager; you may write only your own rows, manager included. *A manager may look, never touch.*

Enforcing that sentence took two independent systems, because there are two doors into the data.

---

## 1. Why two layers

| Door | Who uses it | Guarded by |
|---|---|---|
| The Data API | the app, in a browser | compiled policies — **layer 1** |
| Direct SQL | the `work-hours` CLI, `az`, anything with a connection | row-level security — **layer 2** |

Layer 1 alone is not enough, because [the agent path](agentic-layer.md) goes underneath it. Layer 2 alone is not enough, because the app never touches SQL as the end user — its data backend connects as one identity for everybody.

Two systems enforcing one sentence is a drift problem waiting to happen. The failure mode of a security control is *silence*: nothing breaks, nothing logs, nothing alerts, and one day somebody reads a row they should not have. So neither layer owns the rule.

```
                  rayfin/data/access.json
                  { managerEmails: [...],
                    serviceLogins:  [...] }        ← THE source of truth
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
   access.ts  ownerOnly()          wht.mjs rls --apply
              ownerOrManager()             │
              │                            ▼
     compiled into API policies    sec.WorkHourAccess predicates
              │                    + sec.Enrollment allowlist
              ▼                            ▼
   ┌────────────────────────┐   ┌───────────────────────────┐
   │ guards THE APP         │   │ guards DIRECT SQL         │
   └────────────────────────┘   └───────────────────────────┘
                  │                        │
                  └──── `rls --status` compares the two
                        and complains when they have drifted ────┘
```

A plain JSON file is the one format a TypeScript policy and a Node script can both read without either one parsing the other's source. Changing who is a manager therefore takes two commands — one to redeploy the app's policies, one to rewrite the database's predicates — and a third that tells you whether you remembered both.

---

## 2. Layer 1 — the compiled policy

An entity is a class, and its access rules are decorators on it:

```ts
@entity()
// Read: your own rates — plus everyone's, if you are the manager.
@authenticated('read', {
  policy: (claims, item) => ownerOrManager(claims, item.user_email),
})
// Write: strictly your own. Nobody sets anyone else's rate.
@authenticated(['create', 'update', 'delete'], {
  policy: (claims, item) => ownerOnly(claims, item.user_email),
})
export class ProjectRate {
  @uuid()    id!: string;
  @decimal() hourlyRate!: number;

  @uuid() project_id!: string;
  @one(() => Project) project?: Project;

  @text({ max: 320, optional: true }) user_email?: string;
}
```
*Abridged from `rayfin/data/ProjectRate.ts`. `rayfin up` turns this into an MSSQL table, REST endpoints, and a compiled policy on each of them.*

The rule those two decorators point at is written **once**, for every entity:

```ts
export const MANAGER_EMAILS: string[] = accessConfig.managerEmails;

/** "This row is mine." — the whole rule for create, update and delete. */
export function ownerOnly(claims: ClaimsDsl, ownerEmail: FieldRef): PolicyExpression {
  return claims.email.eq(ownerEmail);
}

/** "This row is mine, OR I am a manager." — the read rule. */
export function ownerOrManager(claims: ClaimsDsl, ownerEmail: FieldRef): PolicyExpression {
  return MANAGER_EMAILS.reduce<PolicyExpression>(
    (expression, managerEmail) => expression.or(claims.email.eq(managerEmail)),
    ownerOnly(claims, ownerEmail),
  );
}
```
*`rayfin/data/access.ts`. Functions rather than copy-pasted expressions, because one entity left on the old rule is an invisible hole.*

**The constraint that shaped everything here:** a policy compiles to a predicate comparing the caller's claims against columns **on the same row**. It cannot look anything up — no subquery, no join, no *"is this person in the managers table?"*. And the two built-in roles cannot be extended, because custom claims are static and app-wide, so nothing can hand one person a `manager` claim and not another.

That leaves two possible shapes, and this took the simpler one: the manager's address is baked into the compiled policy. The trade-off is honest — adding a manager is a code change plus a redeploy, not an edit in the app. For one manager who sees everybody, that is the right trade.

The same constraint is why sharing is a join table rather than a list column. **The data model was chosen by what the policy engine can express.**

---

## 3. Layer 2 — row-level security

The same sentence, as a SQL predicate:

```sql
CREATE OR ALTER FUNCTION sec.fn_read_access(@user_email nvarchar(320))
  RETURNS TABLE WITH SCHEMABINDING
AS RETURN
  SELECT 1 AS allowed
   WHERE NOT EXISTS (SELECT 1 FROM sec.Enrollment e WHERE e.login = SUSER_SNAME())
      OR @user_email = SUSER_SNAME()
      OR EXISTS (SELECT 1 FROM sec.Enrollment e
                  WHERE e.login = SUSER_SNAME() AND e.is_manager = 1);
```

`fn_write_access` is the same function **without the manager clause** — the mirror of `ownerOnly` on the other side.

`SUSER_SNAME()` returns the caller's real Entra UPN, which is exactly what the `user_email` columns hold, so both halves of the system agree on what an identity *is* with no mapping table in between.

### Five predicates per table, and what each one closes

A filter alone leaves four holes open. The policy attaches all five:

| Predicate | Without it |
|---|---|
| `FILTER` | you can see other people's rows |
| `BLOCK AFTER INSERT` | you can create a row owned by someone else |
| `BLOCK BEFORE UPDATE` | the manager can see a colleague's row — so he could edit it |
| `BLOCK AFTER UPDATE` | he could reassign it to himself, which `BEFORE UPDATE` alone does not catch |
| `BLOCK BEFORE DELETE` | seeing a row would otherwise let him delete it |

**Measured: 5 predicates × 5 tables = 25**, generated from a two-key JSON file. The guarded tables are `TimeEntries`, `ProjectRates`, `ProjectMonthlyPlans`, `ProjectShares` and `UserProfiles`.

`Projects` is deliberately **absent**. Everyone logs time against the same projects and may edit them, so its `user_email` is a *created by* label rather than an owner, and filtering on it would hide colleagues' shared projects.

---

## 4. The two exceptions, stated rather than hidden

Both are deliberate, both are documented in the source, and both belong in a write-up rather than in a footnote.

### Project names are readable by any signed-in user

A policy cannot check membership, so the app filters the project list and the backend does not. A direct API query returns every project *name*. Hours, rates, plans and earnings are not exposed. It is a leak of labels rather than of data — but it is a leak, and calling it "filtered in the UI" would be dishonest.

### Row-level security is an allowlist, not a denylist

The predicates filter only logins listed in `sec.Enrollment`, and let every unrecognised identity through untouched.

That looks backwards for a security control. The reason: **the app's data backend connects to SQL as the Fabric workspace owner** — measured on a live deployment, not assumed — so it authenticates as a real person rather than as a service principal. A filter that caught that identity would intersect with every other user's policy and blank the app for everybody at once.

Failing open on unknown logins means the blast radius of a wrong guess is *"someone is not protected yet"*, not *"the product is down"*. It also costs less than it looks: an unenrolled identity can only reach this database by being a Fabric workspace admin or member, and those arrive as `db_owner` — who can simply drop the policy anyway.

**This protects enrolled employees from each other. It is not a wall against a workspace administrator, and it is not sold as one.** Run `rls --status` before telling anyone what their privacy actually is.

The fix, and the reason it has not been applied yet: give the Fabric item its own service principal, then the workspace owner can be enrolled like everybody else.

---

## 5. What I would do differently

- **Ship the drift check on day one, not after the second layer exists.** `rls --status` was written last. It should have been written first, because until it existed there was no way to answer *"do these two agree?"* other than reading both by eye.
- **Never let an application run as a person.** Everything awkward in section 4 follows from that one fact.
- **Write the exceptions into the source, beside the code they excuse.** Both of the above live as long comments in the files that implement them. Six months later that is the difference between a documented trade-off and a bug nobody can explain.

[← Work hour tracker](../work-hour-tracker.md) · [Portfolio](../../README.md)
