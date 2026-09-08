## Project: AKEWTutor — Backend Test Documentation: Shared Config — Integration (Persistence)

**Sibling of:** [9-1. Backend Test Documentation: Shared Config (Auth, Notifications, Policies, Announcements) + Cross-Cutting Foundations] (Unit + Integration (HTTP contract) tiers live there). **Links back to:** [04. Database & Data Model §4.2 (RefreshToken), §4.4 (Indexes & Constraints)], [8-1. Function-Level Spec: Shared Config — `refreshAccessToken`, `logout`, `logoutAll`], [00-agent-rules.md's Audit-log test convention].
**Created by:** Phase 5.4 of `09-redesign-implementation-plan.md`, per Review §6.2 item 4.

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### Why this file exists, and what it does not duplicate

`9-1-shared-config.md`'s Unit tier (§9.8, `refreshToken.ts` and `login / refreshAccessToken / logout / logoutAll` subsections) already proves — against a **mocked** Prisma client — that `refreshAccessToken` reads the presented token, decides between the rotate-path and the reuse-detected-path, and calls the right shape of update/create. It cannot prove that the rotation is actually atomic against a real database, that the `RefreshToken(tokenHash)` uniqueness the schema promises is a real constraint, that a genuinely concurrent pair of requests racing the same token can't both "win," or that the `User → RefreshToken` cascade-delete behavior `04-database-and-data-model.md §4.4` documents as this schema's one deliberate exception actually fires. A mock told to return "already revoked" for the reuse case only proves the code branches correctly on that input — it was never actually raced against a real row.

**In scope (per Review §6.2 item 4 — narrower than the other four modules' persistence scope, and deliberately not expanded here):** refresh-token **rotation** and **reuse detection**, run against a real database, including the concurrent-request race the review calls out by name ("reuse-detection logic is exactly the kind of thing that looks correct against mocks and fails against real transaction ordering"). Per `00-agent-rules.md`'s Audit-log test convention (§"Integration (persistence) tier" bullet), this file is also where the Unit tier's `LOGIN_FAILED_THRESHOLD` audit-call assertion (`9-1-shared-config.md §9.8`) gets its durability escalation — the same treatment already given to `GUARDIAN_REMOVED`/`ACCOUNT_SUSPENDED` in `9-2-accounts-guardianship-persistence.md` and `REFUND_APPROVED`/`REFUND_REJECTED` in `9-7-payments-earnings-persistence.md`, and the specific pairing (repeated failed logins → Integration (persistence)) the review's own §6.4 table names.

**Deliberately not in this file** — see §9.25 below:
- `login`'s token-*issuance* path (first login, no prior family) — the Unit tier's mocked assertion ("a `RefreshToken` row is created with a hashed value and a fresh `familyId`") is a single, unconditional `create` call with no constraint contention and no branch to get wrong; a real-DB pass doesn't strengthen it. It is, however, the necessary *setup* step for every case below, so it happens implicitly in every case's Setup column — just not asserted on for its own sake a second time.
- `logout` / `logoutAll` — both are a single `updateMany` with no rotation chain, no unique-constraint interaction, and no concurrency risk (per `04-database-and-data-model.md`'s note that already-issued access tokens on other devices simply expire naturally; there is no real-DB race to prove here that the Unit tier's mocked assertion doesn't already cover).
- `src/utils/refreshToken.ts` (`generateRefreshToken`, `hashRefreshToken`) — pure functions with no database interaction at all; the Unit tier already tests these against real `crypto`, not a mock, so there is nothing this tier adds.

---

### Test environment convention (binding on every test file in this document)

1. **Real database, no silent fallback**, per `00-agent-rules.md` Rule 7.
2. **Isolation between tests** (transactional rollback or truncate-and-reseed) — no case in this file may depend on `RefreshToken`/`User` rows left behind by another test.
3. **Real Prisma client, real service functions.** Every case below calls the actual exported `auth.service.ts` functions — never a re-implementation of the rotation/reuse logic — with the real Prisma client injected/imported as production code does.
4. **Fixture usage.** Every `User` seeded in this file comes from `shared-config.factory.ts`'s `buildUser()`; per `00-test-fixtures.md §1.1`, every `RefreshToken.userId` used below is a real, just-created `User.id`, never a factory default UUID with no backing row.
5. **No prescribed locking mechanism.** Where a case below asserts "exactly one call rotates successfully," the test asserts the **outcome** against real concurrent DB access — it does not assert *which* mechanism (a conditional `UPDATE ... WHERE revokedAt IS NULL` returning zero rows, a `SELECT ... FOR UPDATE`, or a serializable transaction) the implementation uses to achieve that outcome, consistent with how `9-3-matching-cohorts-persistence.md §9.14`'s equivalent note frames the same choice for cohort-capacity races.

---

### 9.22 Test File Map (persistence)

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/services/auth.service.ts | tests/integration/auth.service.persistence.test.ts | Integration (persistence) | ☐ |

Only one row: per the scope note above, `refreshAccessToken`'s rotation/reuse path is the sole real-concurrency, real-constraint surface in this module that the Unit tier's mocks can't fully validate. This is consistent with `00-agent-rules.md` Rule 4 — every source file still needs a Test File Map row even when, as here, a persistence tier turns out to need only one.

---

### 9.23 Test Case Detail — auth.service.persistence.test.ts

FRs: FR-SP-003. NFRs: NFR-013, NFR-014, NFR-015. Traces to `04-database-and-data-model.md §4.2` (RefreshToken) and §4.4 (RefreshToken indexes/constraints), `8-1-shared-config.md`'s `refreshAccessToken` side-effects note.

#### refreshAccessToken — real rotation, durably correct

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Rotation is real and atomic | seed a real `User`; call `login(identifier, password)` against the real DB to obtain a real, active `RefreshToken` row (`revokedAt: null`) and its raw token | call `refreshAccessToken(rawToken)` | a fresh query shows the **presented** row now has `revokedAt` set and `replacedByTokenId` pointing at a real, different `RefreshToken.id`; a second fresh query on that new id shows an active (`revokedAt: null`) row with the identical `familyId` as the presented one — both read from the database, not the function's return value |
| The `tokenHash` unique constraint is real | seed a real, active `RefreshToken` row via `login` as above | attempt a direct Prisma insert of a second `RefreshToken` row reusing the identical `tokenHash` | rejected by the real unique constraint (`P2002`) — schema-level confirmation of `04-database-and-data-model.md §4.4`'s `RefreshToken(tokenHash)` constraint, independent of any service function |
| **Concurrent refresh race on the same still-valid token — only one caller rotates it** | seed a real, active `RefreshToken` row via `login` | issue two concurrent `refreshAccessToken(sameRawToken)` calls (`Promise.all`, two separate DB connections, so the calls are genuinely interleaved, not serialized by a shared connection) | exactly one call resolves a new `{ accessToken, refreshToken }` pair; the other throws `ApiError(401, "Session expired — please log in again")`. A fresh query confirms the original row has `revokedAt` set **exactly once** (not toggled twice) and points to **exactly one** real child `RefreshToken` row, never two — this is the review's named risk made concrete: a non-atomic implementation (read-then-write instead of a single guarded conditional update) could let both concurrent calls observe `revokedAt: null` before either writes, producing two "winning" children sharing one `familyId`, which is exactly the kind of double-issuance a mock could never surface since it always returns whatever the test told it to |

#### refreshAccessToken — real reuse detection

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Reuse of an already-rotated token revokes every real row in the family | seed a real, active `RefreshToken` row via `login`; call `refreshAccessToken` once for real to rotate it (per the first case above), producing a real child row | call `refreshAccessToken` again with the **original** (now-revoked) raw token | throws the identical `ApiError(401, "Session expired — please log in again")`; a fresh query for every `RefreshToken` row sharing that `familyId` shows **all** of them — the original and the rotated child alike — now have `revokedAt` set, confirmed by an actual `findMany({ where: { familyId } })`, not inferred from the presented row alone |
| Reuse detection does not touch a different user's family | seed two real `User`s, each with their own real, active `RefreshToken` (distinct `familyId`s) via `login`; rotate user A's token once for real | present user A's now-revoked original token to `refreshAccessToken` again (triggering reuse detection) | a fresh query confirms every `RefreshToken` row in user B's `familyId` remains untouched (`revokedAt: null`) — the family-wide revocation genuinely scopes by `familyId`, not by some broader match that could over-revoke |
| A three-generation-deep chain is revoked in full on reuse of the oldest token | seed a real, active token via `login`; rotate it twice more for real in sequence, producing a genuine three-row chain (`grandparent → parent → child`), all sharing one `familyId` | present the **grandparent's** (oldest, longest-revoked) raw token to `refreshAccessToken` | throws the same `ApiError(401, ...)`; a fresh `findMany({ where: { familyId } })` shows all three real rows — including the currently-active `child` — now have `revokedAt` set. This is the case a shallow mock is least likely to get right: a mock that only knows how to revoke "the presented row plus one other" would pass a two-row reuse test while silently leaving a longer real chain's active leaf token valid |

#### User deletion — real cascade behavior

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Deleting a User real-cascades its RefreshToken rows | seed a real `User` with two real, active `RefreshToken` rows (from two separate real `login` calls, simulating two devices) | delete the `User` row directly via Prisma | both `RefreshToken` rows are gone on a fresh query — confirming `04-database-and-data-model.md §4.4`'s documented `onDelete: Cascade` exception (the only cascading relation in the entire schema) is a real, enforced database behavior, not just a documentation note nobody has verified against an actual migration |

#### login's `LOGIN_FAILED_THRESHOLD` audit entry — durable persistence

| Case | Setup | Action | Expected result |
|---|---|---|---|
| **The threshold-crossing audit entry is durably persisted, not just "called"** | seed a real `User`; perform (N − 1) real failed `login(identifier, wrongPassword)` calls against the real DB, where N is the same threshold `rateLimiter.middleware.ts` uses for `login` per NFR-013 (matching `9-1-shared-config.md §9.8`'s Unit-tier case) | call `login(identifier, wrongPassword)` one further time (the Nth) | a fresh query against the real `AuditLog` store returns an entry with `actor: <the matched user's id>, action: 'LOGIN_FAILED_THRESHOLD', target: <same user id>`, and a real, persisted `timestamp` — escalating the Phase 4 Unit-tier "`record()` was called" assertion to a durability proof, per `00-agent-rules.md`'s Audit-log test convention and consistent with the identical treatment given to `GUARDIAN_REMOVED`/`ACCOUNT_SUSPENDED` (`9-2-accounts-guardianship-persistence.md`) and `REFUND_APPROVED`/`REFUND_REJECTED` (`9-7-payments-earnings-persistence.md`) |
| The (N − 1)th failed attempt writes no audit entry | seed a real `User`; perform (N − 2) real failed `login` calls | call `login(identifier, wrongPassword)` once more (the (N − 1)th, one short of the threshold) | a fresh query against the real `AuditLog` store returns **no** entry for this user/action — confirming against real data (not a mock told what "not yet" looks like) that the entry fires only at the threshold, not on every failure below it, which is the same distinction `9-1-shared-config.md §9.18`'s Coverage Honesty Check already flags as easy to get wrong at the Unit tier and worth re-confirming here against a real accumulation of real rows rather than a single mocked call count |

---

### 9.24 Coverage Honesty Check (persistence addendum — per PR Steward, at review time)

- [ ] Every case in §9.23 above was run against an actually-reachable real test database at review time, not skipped/pending due to environment unavailability.
- [ ] The concurrent-refresh race case was observed to genuinely interleave at the database level at least once across repeated local runs, not merely "passed once" — a race test that only ever happens to serialize favorably is not exercising the guard it claims to.
- [ ] The three-generation reuse-detection case queried **every** row in the family (not just the presented row and its immediate replacement) before asserting all were revoked.
- [ ] The cascade-delete case confirmed **both** seeded `RefreshToken` rows were gone, not just one — a test that seeded and checked only a single token could pass even if the cascade only fired for the "current" row and left older, already-revoked rows in the same family orphaned.
- [ ] The `LOGIN_FAILED_THRESHOLD` audit-persistence case was verified against whatever concrete storage the implementation chose for `AuditLog`, matching the same verification already done in `9-2-accounts-guardianship-persistence.md` and `9-7-payments-earnings-persistence.md` — no case here re-invents a different verification method than those two established.
- [ ] The "(N − 1)th attempt writes nothing" case's absence-assertion queried the real `AuditLog` store (not the mocked spy from the Unit tier) and confirmed zero rows, not merely that the function didn't throw.

---

### 9.25 Out of Scope for Automated Testing (and why)

- **`login`'s token-issuance path, `logout`, `logoutAll`, and `src/utils/refreshToken.ts`.** As noted above, none of these carry the real-constraint or real-concurrency risk this tier exists to catch; their Unit-tier coverage (mocked for the first three, against real `crypto` for the last) is not meaningfully strengthened by a real-DB pass. Flagged explicitly here, rather than silently omitted from the Test File Map, per this doc set's "document why, don't just skip" convention (Review §2 item 3).
- **Rate-limiting behavior itself** (the `login`/`resendVerification`/`requestPasswordReset` per-endpoint limits). This was Phase 0's contradiction-resolution scope, not Phase 5's — see `9-1-shared-config.md §9.19`'s decision record. This file only escalates the *audit-logging consequence* of repeated failed logins to a durability proof; it does not re-test the rate limiter itself, which remains an in-memory-store concern already covered by `rateLimiter.middleware.test.ts`.
- **Sustained multi-client load / token-storm testing** (hundreds of concurrent refresh attempts across many users). The two-way race case above proves the rotation guard is real, not merely mock-shaped, but it is not a substitute for a dedicated load/performance testing pass (Review §3.8, deferred as a low-priority, separately-tracked item) — consistent with the same exclusion `9-3-matching-cohorts-persistence.md §9.19` and `9-7-payments-earnings-persistence.md` already make for their own concurrency cases.
- **`createdByIp` / `userAgent` field population.** `04-database-and-data-model.md §4.2` documents these as "best-effort, for audit only — never used as a security boundary"; since no case in this file (or the Unit tier) treats them as a security-relevant assertion, a persistence-level check of their presence would test logging hygiene, not a behavior this spec makes any guarantee about.

---

**Back to:** [9-1. Backend Test Documentation: Shared Config] · **Next:** proceed to → Phase 5.5 (`9-6-gamification-engagement-persistence.md`)
