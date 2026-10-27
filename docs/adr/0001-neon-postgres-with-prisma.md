# ADR-0001: Neon Postgres with Prisma 7 and the Neon driver adapter

- **Status:** Accepted (2026-10-27)
- **Date:** 2026-10-27
- **Relates to:** TAD §2, §12, §14; PRD NFR-COST-01, NFR-PERF-02/03, NFR-REL-01

## Context

The brief fixes PostgreSQL on Neon through Prisma, on free tiers. The data is relational and date-heavy: share intervals, rule revisions, per-day entries, and several uniqueness rules that must hold under concurrent requests (the 15-member cap, one open share interval per target and group).

Facts checked on 2026-10-27:

- Neon Free: 100 CU-hours per project per month, scale-to-zero after 5 minutes of inactivity (cannot be disabled), 5 GB public network transfer, 1 GB storage, 10 branches, 6-hour restore window.
- Prisma 7 connects to Neon through a driver adapter. `@prisma/client` and `@prisma/adapter-neon` are at 7.10.1. The npm `latest` tag of the `prisma` CLI points to **8.0.0-rc.27**, a release candidate.
- Prisma's Neon guide recommends the pooled (`-pooler`) URL for the application and the direct URL for the CLI.

## Decision

- Use Postgres on **Neon Free**, accessed with **Prisma ORM 7.10.1**. Pin `prisma`, `@prisma/client` and `@prisma/adapter-neon` to the same exact version; never install `prisma@latest`.
- The app connects with `PrismaNeon` using `DATABASE_URL` (pooled). The CLI (`prisma migrate`) uses `DIRECT_URL` through `prisma.config.ts`.
- Use **Prisma Migrate**, committed as SQL. Where Prisma's schema language cannot express a constraint (partial unique indexes, exclusion constraints, `CHECK`s, case-insensitive uniqueness) edit the generated SQL.
- Run on Vercel's **Node.js runtime**, with the Vercel function region and the Neon region in the same geography.
- Use interactive transactions for multi-step writes. Enforce the group cap by locking the group row (`SELECT … FOR UPDATE`) and counting active memberships inside the join transaction.
- Store local dates in `DATE` columns (ADR-0005). Convert to and from `YYYY-MM-DD` strings only in the repository layer.
- Use a Neon dev branch for local work and branches for previews; take the single manual snapshot before risky migrations.

## Alternatives considered

| Option | Why not |
|---|---|
| Drizzle ORM | Fine technically, but not in the brief's stack. |
| Raw `@neondatabase/serverless` SQL | Loses typed models and migrations; more code to get wrong. |
| Prisma Accelerate or Prisma Postgres | An extra service with its own quotas; unnecessary at this scale. |
| Prisma 8 release candidate | Not stable; `@prisma/client` and the adapter are not released at 8. |
| SQLite/Turso, Supabase | Different stack from the brief; no benefit here. |

## Consequences

- **+** Typed data access, migrations, and database branching for previews.
- **+** Real transactions and constraints keep the cap and interval invariants correct under concurrency.
- **−** Some constraints live in hand-edited SQL and need tests.
- **−** The first request after 5 idle minutes pays a database wake-up (budgeted in NFR-PERF-03).
- **−** **Neon compute hours are the tightest free-tier limit.** Expected use is about 30–41% of the allowance and grows with the number of visits per day (TAD §12.3). This is why the polling degradation ladder exists.
- **−** Prisma's own Neon page shows an older adapter version and is ambiguous about where the datasource URL belongs in Prisma 7; spike S1 settles this before feature work.
- **−** Exact pins mean upgrades are deliberate, manual steps.
