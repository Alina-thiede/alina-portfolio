# Technical details — What I'd tell the next person

*Outcome and recommendations in full, what the build found, the design decisions behind it, how it was checked, and what is still missing.*

[← back to the story](../work-hour-tracker.md#what-id-tell-the-next-person) · [Portfolio](../../README.md)

---

## 1. Outcome

- One Fabric SQL database holds projects, time entries, per-person rates and monthly plans behind Entra sign-in — reporting is a query, not a collection exercise.
- Two write paths, one data shape: the web app, and a Claude Code plugin ("log 3h on Acme today") driven by a Node CLI. Both stamp the same four owner columns, so rows written from the terminal are indistinguishable from rows written in the browser.
- Earnings are point-in-time correct: each time entry carries a frozen copy of the rate that applied when it was logged.
- Per-person data isolation is enforced in the database itself, not only in the app.

---

## 2. Recommendations

I am confident about the first two. The third is a guess about how this company grows, and the fourth is the one I keep not doing.

- **State the privacy boundary exactly, then close it.** The row-level policy is an *allowlist* over enrolled logins — an unenrolled login (workspace admin or owner) is not filtered at all. Enrol every colleague, and run `rls --status` before telling anyone their rows are private.
- **Give the app its own service principal.** The whole allowlist exists because the app's data backend connects to SQL as the *workspace owner* — a real person — so a deny-by-default policy would catch it and blank the app for everyone. With a service principal, that person can be enrolled like anybody else and the policy can close. *(The policy itself is now complete: 25 predicates across all 5 owner-scoped tables — see [security](security.md).)*
- **Move the plugin repo to the organisation account.** It lives under a personal account today; the transfer keeps history, survives an owner leaving, and is a precondition for org-wide distribution.
- **Before adding analytics, add tests.** The next feature should not be the first thing this codebase relies on CI for.

---

## 3. Results in detail

- **Role-based authorisation was a dead end on this platform, and finding that out early saved building it twice.** *(measured)* Rayfin exposes exactly two roles, `anonymous` and `authenticated`; `claims.role` is defined only in `rayfin.yml`, which is static and app-wide; and API policies compare claims against columns on the same row, so "is this person a manager?" cannot be a subquery. Access control had to move down into SQL.
- **The policy had to be an allowlist, not a denylist.** *(measured)* Filtering every login would have caught the app's own database identity and blanked the app for every user simultaneously. It therefore filters only logins enrolled in `sec.Enrollment` — which is exactly why an unenrolled admin is still unfiltered. That limitation is a consequence of the design, not an oversight.
- **`SUSER_SNAME()` on a skill connection returns the real Entra UPN**, matching stored `user_email` values *(measured)* — the fact that makes SQL-side row filtering viable, and the reason the skill needs no per-person configuration.
- **Three defects surfaced only under real installation, not review** *(measured)*: a setup probe that called `process.exit` instead of throwing, so `setup` told the user to run `setup`; a `^11.0.1` dependency spec silently pinned to an exact version because Windows `cmd.exe` eats `^`; and a home directory containing a space arriving as two arguments under `shell: true` (Node DEP0190). The middle one cost me most of a day, and I was angrier about it than a caret deserves.
- **Plugin config cannot live beside the plugin.** *(measured)* Claude Code replaces the plugin cache directory wholesale on every version bump, and `$CLAUDE_PLUGIN_DATA` is not exported to skill-invoked bash — so the `~/.work-hours/` branch is what actually runs, not a fallback.

---

## 4. Design decisions & trade-offs

| Decision | Rejected alternative | Why |
|---|---|---|
| Row-level security in SQL | Role-based API policies | The platform has no per-user role claim and its policies cannot subquery — the API layer physically cannot express "managers see everyone" |
| Allowlist policy over enrolled logins | Deny-by-default for all logins | Deny-by-default catches the app's own service identity and blanks the app for everyone; the cost is that unenrolled admins stay unfiltered, and that limit is documented rather than hidden |
| Frozen rate copied onto each entry | Join to the current rate at read time | A raise must not rewrite invoiced history; the duplication is the point |
| A shared `Projects` table with the rate split out into `ProjectRates` | One rate column on the project | A project — its name and colour — is shared by nature; a rate is personal. With one column, sharing a project shared the money too, and two people could not work the same engagement at different rates |
| Sharing as a join table (`ProjectShares`) | `shared_with: "a@x;b@x"` on the project row | Not taste — a policy can only compare a claim against a column *on the same row*: no `contains`, no subquery. A delimited list is physically unenforceable at the backend. **The data model was chosen by what the security layer can express** |
| `edit-entry` / `edit-project` in place | Delete and re-add | Delete-and-re-add silently discards the id, untouched notes and the frozen rate |
| Config in `~/.work-hours/` | Config beside the plugin | The plugin directory is deleted and re-copied on every version bump |
| Direct SQL for the agent path | Reuse the app's Data API | Fabric sign-in is browser-only — no device-code or service-principal flow exists for a terminal tool |

The data-model rows are explained with code in [the data model](data-model.md); the security rows in [security](security.md).

---

## 5. Quality & testing

Verified by hand end to end, from a clean install through to writes landing in the live database. The safety work sits in the tool design rather than in a test suite:

- `setup --check` walks the whole chain — az login → config → driver → token → connectivity → row ownership — and reports where it breaks.
- `--dry-run` on every write prints before → after without touching the database.
- `query` accepts a single `SELECT` only, rejects DML/DDL, and runs inside a transaction that always rolls back. It is a guard-rail against accidents, not a security boundary.
- Writes report the row count they actually affected and fail loudly on zero, so "it worked" is never inferred.
- `WHT_USER_EMAIL` is an optional guard-rail: if it disagrees with the connected identity, the CLI refuses to run rather than show the wrong person's data.

**Gap, stated plainly:** there is no automated test suite and no CI. That is the first item in section 6.

---

## 6. Limitations & next steps

- **No automated tests or CI.** Every defect so far was found by installing and using the thing. First priority.
- **Project *names* are readable by any signed-in user.** A policy cannot check membership, so the app filters the project list and the backend does not. Hours, rates and plans are not exposed — but names are, and calling that "filtered in the UI" would be dishonest.
- **Manager visibility is a literal email list** compiled into the policy. Fine for one team, wrong past a handful of people.
- **Unenrolled logins are not filtered.** Workspace admins and owners read everything, by design of the allowlist.
- **Onboarding is manual**: Node, the Azure CLI, and a one-off database grant per person.
- **This is an OLTP application, not a pipeline.** The natural next build is the analytics layer — a scheduled aggregate of hours and earnings into a Fabric lakehouse, with freshness and quality checks.

[← back to the story](../work-hour-tracker.md#what-id-tell-the-next-person) · [Portfolio](../../README.md)
