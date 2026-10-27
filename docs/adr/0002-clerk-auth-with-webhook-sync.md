# ADR-0002: Clerk for authentication, with lazy user creation plus webhook sync

- **Status:** Accepted (2026-10-27)
- **Date:** 2026-10-27
- **Relates to:** TAD §4, §10; PRD FR-ACC-01/02/03/04, FR-GRP-11, NFR-REL-02, NFR-SEC-01, NFR-PRIV-01

## Context

The brief chooses Clerk (Google OAuth and email/password) with users synced to our database by a webhook. Three things complicate a webhook-only design:

- FR-ACC-02 requires the profile to exist the moment sign-up completes. A webhook can arrive after the user's first request.
- Clerk's webhook guide states that deliveries are eventually consistent, can arrive out of order or twice, and are retried on 4xx/5xx or no response.
- NFR-REL-02: a user deleted in Clerk must never break group pages for other members, even though full account deletion (FR-ACC-07) is v1.1.

Clerk 7.9.21 supports Next.js 16 (`proxy.ts`, `clerkMiddleware`, `await auth()`) and ships `verifyWebhook` in `@clerk/nextjs/webhooks`, so no extra webhook library is needed.

## Decision

1. **Clerk handles identity and sessions only.** Our `users` table holds the application profile.
2. **Lazy creation is authoritative.** `ensureUser(clerkId)` creates the row on the member's first authenticated request (an upsert that is safe under concurrency). The hot path is one query.
3. **The webhook keeps the row in sync.** Subscribe to `user.created`, `user.updated` and `user.deleted`. Verify every request with `verifyWebhook` and `CLERK_WEBHOOK_SIGNING_SECRET`. All handlers are idempotent; a DB failure returns 500 so Clerk retries.
4. **Profile fields are copied** (display name, image URL) so that **no read path ever calls Clerk for another member**. Email is not stored.
5. **Deletion is a tombstone, not an erase.** `user.deleted` marks the user deleted, anonymises the display name, frees the handle, sets their memberships to `REMOVED`, closes their open share intervals and clears cached streaks. Rows are kept so nothing can dangle. A later `user.updated` for a tombstoned row is ignored. Erasure and owner succession come with FR-ACC-07 in v1.1.
6. `proxy.ts` protects everything except sign-in, sign-up and the webhook route, as an optimistic check. Every server entry point re-verifies identity (`requireUser`) and membership (`requireGroupAccess`).

## Alternatives considered

| Option | Why not |
|---|---|
| Webhook-only sync | Races the first request; FR-ACC-02 would need retries or a waiting screen. |
| Lazy creation only | Never learns about deletions or profile changes. |
| Reading member profiles from Clerk at render time | Adds latency and rate-limit exposure to every group page, and breaks the moment a member is deleted. |
| Carrying our user id in session claims | Extra provider configuration and still needs the row to exist. |
| Auth.js, Better Auth or Neon Auth | A change from the brief; we would own password handling. |

## Consequences

- **+** Immediately usable after sign-up; group pages depend only on our database.
- **+** Deleted users cannot break anyone's page, in any role, including the owner (the group simply has no active owner until v1.1 succession).
- **−** Two write paths must stay consistent. Mitigated by one shared `upsertFromClerk`, idempotence, and the tombstone check.
- **−** Names and avatars can be stale until the webhook lands.
- **−** Local development cannot receive webhooks without a tunnel; `ensureUser` makes that unnecessary for day-to-day work.
- **−** Tombstoned rows stay in the database until v1.1 adds erasure.
- **−** Vendor dependency on Clerk for sign-in.
