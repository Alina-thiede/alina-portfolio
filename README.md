![](assets/alina-thiede-header.png)

# Alina Thiede

Junior data engineer · Microsoft Fabric and Power BI · Vienna

[CV](#TODO-cv-link) · [LinkedIn](#TODO-linkedin) · [Email](#TODO-email)

---

I build multi-user data applications on Microsoft Fabric — declared schemas, Entra-authenticated APIs, row-level security in SQL — and the agent tooling that lets people query and write that data in plain English.

Stack
- Interfaces · `React 19 · TypeScript · Vite · Claude Code plugins (Node CLI)`
- Storage · `Microsoft Fabric SQL (MSSQL) · declarative entity schemas (Rayfin)`
- Serving · `Auto-generated Data API (DAB) · T-SQL summaries`
- Platform and security · `Microsoft Entra ID · SQL row-level security · Azure CLI token auth`

## Contents
- [Work hour tracker — flagship](#work-hour-tracker--flagship)
- [Contact](#contact)

---

> **A note on what you can click.** This work was built for a working consultancy, so the running system sits behind company sign-in and holds real client hours. What is public instead: a full technical write-up, screenshots, and a short demo recording.

## Work hour tracker — flagship
*Completed: August 2026*

**Problem** · A small consultancy logged billable hours in personal spreadsheets, so "how many hours did we put on this client this month, and what did they cost?" could only be answered by collecting files by hand.
**What I built** · A multi-user time-tracking app on Microsoft Fabric — React frontend, Rayfin-declared entities compiled into a Data API and a Fabric SQL database, Entra ID sign-in — plus a Claude Code plugin that reads and writes the same database in natural language, scoped per person by SQL row-level security.
**Outcome** · Hours, rates and monthly plans live in one queryable Fabric SQL database instead of per-person spreadsheets; logging and reporting work either through the web app or conversationally from a terminal, and both paths write rows that are indistinguishable from each other. Historical earnings became immutable — every entry freezes the rate that applied when it was logged, so a raise never rewrites last year's money. *(measured: released as plugin v1.0.1 and verified end to end; row-level security active on 4 of 5 tables)*
**Recommendation** · Treat the database policy, not the app, as the privacy boundary — it filters enrolled logins only, so a Fabric workspace admin still reads everything; enrol every colleague before promising confidentiality, and move the plugin repo from the personal account into the organisation account so distribution survives an owner leaving.
**Stack** · `React 19 · TypeScript · Vite · Tailwind 4 · Rayfin 1.33 on Microsoft Fabric · Fabric SQL (MSSQL) · Entra ID · Node CLI (mssql) · Claude Code plugin`

**[60-second demo](#TODO-demo-video)** · **[Full write-up →](projects/work-hour-tracker.md)** · **[Code excerpts →](#TODO-code-excerpts)**
*Live app is private — it holds colleague hours under company sign-in.*

<!-- TODO: two more featured projects (a different capability each) + a supporting-work table
     | Project | What it does | Stack |
     Portfolio rules: exactly 3 featured, <=5 supporting, strongest first, never chronological. -->

---

## Contact

[CV](#TODO-cv-link) · [LinkedIn](#TODO-linkedin) · [Email](#TODO-email)
