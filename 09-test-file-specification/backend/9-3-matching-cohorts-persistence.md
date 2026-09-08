## Project: AKEWTutor — Backend Test Documentation: Matching & Cohorts — Integration (Persistence)

**Sibling of:** [9-3. Backend Test Documentation: Matching & Cohorts] (Unit + Integration (HTTP contract) tiers live there). **Links back to:** [04. Database & Data Model §4.2 (MatchRequest, TutorExclusion, Cohort, CohortMembership)], [04 §4.4 (Indexes & Constraints)], [8-3. Function-Level Spec: Matching & Cohorts].
**Created by:** Phase 5.1 of `09-redesign-implementation-plan.md`, per Review §6.2 item 1 (highest-priority module for this tier).

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### Why this file exists, and what it does not duplicate

`9-3-matching-cohorts.md`'s Unit tier already exercises every branch of `matching.service.ts`, `cohort.service.ts`, and `adminMatching.service.ts` against a **mocked** Prisma client — including, since Phase 4, the *shape* of the seat-capacity and admin-claim conditional writes (§9.5's `formOrJoinCohort` last-seat case, §9.7's `manuallyAssignTutor` concurrent-claim case). Those tests prove the code *asks* the database the right conditional question. They cannot prove the database actually *enforces* the answer, because the mock's response is whatever the test told it to return.

This file exists to close exactly that gap, per Review §3.3 and §6.2: every case below runs against a real Postgres test database (Testcontainers or a dedicated test schema — implementation's choice, not prescribed here), using the real Prisma client, real foreign keys, and real unique constraints. Nothing here is mocked except genuine third parties, of which this module has none.

**In scope (per Review §6.2 item 1):** cohort creation, membership assignment, and capacity/claim enforcement against real DB constraints (FKs, unique constraints, and the conditional-write guards Phase 4 already specified at the mock level).

**Deliberately not in this file** — see §9.19 below for the full reasoning:
- `formatSwitch.service.ts`'s persistence behavior (its refund handoff is `payments-earnings`' concern, covered when that module's persistence tier is drafted in Phase 5.2).
- Genuine multi-client, sustained load testing beyond a two-way concurrent race.

---

### Test environment convention (binding on every test file in this document)

1. **Real database, no silent fallback.** Per `00-agent-rules.md` Rule 7: *"Any test marked 'Integration (persistence)' must run against the real test database — it may not silently fall back to a mock if the test DB is unreachable; it must fail loudly."* A test file in this document that cannot reach the configured test database must throw/fail its setup step, not skip or degrade to a mocked client.
2. **Isolation between tests.** Each test resets the tables it touches before or after itself (transactional rollback wrapper or truncate-and-reseed) — no test in this file may depend on row state left behind by another test in the same file or a different one.
3. **Real Prisma client, real service functions.** Tests call the actual exported `matching.service.ts` / `cohort.service.ts` / `adminMatching.service.ts` functions — never a re-implementation or a partial stub of them — with the real Prisma client injected/imported as production code does.
4. **Cross-module fixture usage.** `matching-cohorts.factory.ts` builds this module's own entities, but every one of them FKs into `accounts-guardianship` (`StudentProfile`, `TutorProfile`) and/or `shared-config` (`User`) rows. Test setup in this file seeds real antecedent rows using `accounts-guardianship.factory.ts`'s `buildUser`/`buildStudentProfile`/`buildTutorProfile` and `buildSubject`, then feeds their real generated ids into `matching-cohorts.factory.ts`'s factories — per `00-test-fixtures.md §1.1`, every FK override in this file is a real, just-created row's id, never a factory default UUID.
5. **No prescribed locking mechanism.** Where a case below asserts "capacity is never exceeded" or "only one caller wins," the test asserts the **outcome** against real concurrent DB access. It does not assert which mechanism (row-level lock, guarded conditional `UPDATE`, or serializable transaction) the implementation uses to achieve that outcome — that remains an implementation choice, consistent with how `9-3-matching-cohorts.md §9.5` already frames the guard-clause shape assertion as tier-appropriate to Unit, not persistence.

---

### 9.14 Test File Map (persistence)

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/services/matching.service.ts | tests/integration/matching.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/cohort.service.ts | tests/integration/cohort.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/adminMatching.service.ts | tests/integration/adminMatching.service.persistence.test.ts | Integration (persistence) | ☐ |

`tests/integration/` is a new, parallel directory alongside `tests/services/`, per Review §6.2 — it does not mirror `src/` 1:1 the way the Unit tier's `tests/` tree does, since a persistence test's organizing principle is "which service's real-DB behavior," not "which source file," and several of the cases below span more than one service's write path in a single test (e.g. `tutorExitContinuity`'s handoff into `adminMatching.service.ts`).

---

### 9.15 Test Case Detail — matching.service.persistence.test.ts

FRs: FR-MA-002, FR-MA-006 (as inherited by `selectTutor`'s cohort-creation side effect). Traces to `04-database-and-data-model.md §4.2` (MatchRequest, TutorExclusion, Cohort, CohortMembership) and §4.4 (TutorExclusion's `(studentId, tutorId)` unique composite).

#### selectTutor — real cohort + membership creation

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful selection persists a real Cohort and CohortMembership | seed a real `StudentProfile`, a real `VERIFIED` `TutorProfile`, a real `Subject`; no existing `MatchRequest`/`Cohort`/`TutorExclusion` row for the pair | call `selectTutor(studentId, 'STUDENT', undefined, tutorId)` against the real DB | a `Cohort` row exists in the test DB with `tutorId`/`subjectId` matching the seeded rows and `format: ONE_TO_ONE`; a matching `CohortMembership` row exists with `cohortId` pointing at that Cohort and `studentId` matching the seeded student — both read back via a fresh Prisma query, not the function's return value alone |
| Excluded tutor is rejected before any write occurs | seed the same trio, plus a real `TutorExclusion(studentId, tutorId)` row | call `selectTutor(studentId, 'STUDENT', undefined, tutorId)` | throws `ApiError(400, ...)`; a fresh query confirms **no** `Cohort` or `CohortMembership` row was created for this attempt — the rejection happens before any write, not as a write-then-rollback |
| Double-submit race: two concurrent selectTutor calls for the same student, different tutors | seed one `StudentProfile` and two distinct `VERIFIED` `TutorProfile` rows, no existing match for the student | issue both calls concurrently (`Promise.all`) against the real DB | exactly one call resolves successfully; the other throws `ApiError(409, "You already have a pending or active match")` — confirmed by querying the real DB afterward and asserting **exactly one** `Cohort`/`CohortMembership` pair exists for this student, never zero and never two, regardless of which call the database happened to serialize first |

#### FK integrity — real constraint enforcement, not assumed

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Non-existent tutorId is rejected by a real foreign-key constraint, translated to a clean error | seed a real `StudentProfile` and `Subject`; do **not** create any `TutorProfile` for the id passed | call `selectTutor(studentId, 'STUDENT', undefined, randomUUID())` | the call throws an `ApiError` (not an unhandled `PrismaClientKnownRequestError`, and not a raw Postgres FK-violation message reaching the caller) — this is the one case in this file that exists specifically to catch a mock-based false positive: a Unit test that mocks Prisma to "just work" can never fail this way, but an implementation that doesn't wrap its Prisma call in error translation will surface a raw database error here, which this test is designed to catch |

#### TutorExclusion — real unique constraint

| Case | Setup | Action | Expected result |
|---|---|---|---|
| The `(studentId, tutorId)` unique composite is real, not merely documented | seed a real `StudentProfile` and `TutorProfile`; insert one `TutorExclusion(studentId, tutorId)` row directly via Prisma | attempt a second direct Prisma insert of `TutorExclusion` with the identical `(studentId, tutorId)` pair | the second insert is rejected by the real database's unique constraint (Prisma's `P2002`) — this test targets the schema itself, independent of any service function, confirming `04-database-and-data-model.md §4.4`'s documented constraint actually exists in the schema and isn't just a documentation-only intention |
| `rejectCohort`'s exclusion write does not crash if an exclusion for the pair already exists | seed the pair plus a pre-existing `TutorExclusion(studentId, tutorId)` row (e.g. left over from an earlier, unrelated rejection cycle); seed a `ONE_TO_ONE` `Cohort` pending approval for the same student/tutor | call `rejectCohort(cohortId, adminId, "reason")` | the call completes without an unhandled `P2002` reaching the caller as a 500 — either the write is skipped as a no-op because the exclusion already holds, or the service catches the constraint violation and treats it as already-satisfied; either way the cohort-rejection side effects (fresh `MatchRequest`, cohort state change) still complete, confirmed by querying the real DB afterward |

---

### 9.16 Test Case Detail — cohort.service.persistence.test.ts

FRs: FR-MA-006, FR-MA-009, FR-MA-011, FR-MA-016. Traces to `04-database-and-data-model.md §4.2` (Cohort, CohortMembership) and the partial-unique note on `CohortMembership(cohortId, studentId) where status != ENDED` (§4.4 — application-enforced, not a hard DB constraint; see the capacity cases below for why this file still tests it under real concurrency).

#### formOrJoinCohort — real cohort/membership lifecycle

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates a real Cohort when no compatible FORMING cohort exists | seed a real `StudentProfile`, `TutorProfile`, `Subject`; a real `MatchRequest(format: ONE_TO_FIVE, path: PATH_C)` for the student; no existing `Cohort` row matching subject/format | call `formOrJoinCohort(matchRequestId)` | a fresh `Cohort` row exists in the DB with `status: FORMING` and `groupFormationWindowExpiresAt` set to a real 48-hour-out timestamp; a `CohortMembership` row exists linking the student to it |
| Joins an existing compatible FORMING cohort, does not create a duplicate | seed a real `Cohort(status: FORMING, targetGroupSize: 5)` with 2 existing real `CohortMembership` rows, plus a new student's `MatchRequest` compatible with it | call `formOrJoinCohort(newMatchRequestId)` | no new `Cohort` row is created (row count for matching subject/format/tutor stays at 1); a third `CohortMembership` row is added to the same, pre-existing `Cohort.id` |
| **Last-seat concurrent race — capacity is never exceeded under real simultaneous access** | seed a real `Cohort(status: FORMING, targetGroupSize: 3)` with exactly 2 active `CohortMembership` rows (one seat remaining); seed two distinct students each with a compatible `MatchRequest` | issue both students' `formOrJoinCohort` calls concurrently (`Promise.all`, two separate DB connections/Prisma client instances so the calls are genuinely interleaved, not serialized by a shared connection) | exactly one call resolves successfully; the other throws `ApiError(409, "This cohort is no longer accepting members — please search again")` (or the implementer's equivalent conflict message, matching `9-3-matching-cohorts.md §9.5`'s Unit-tier case). A fresh query against the real DB afterward confirms the `Cohort` has **exactly 3** active `CohortMembership` rows — never 4 — regardless of database-level scheduling order. This is the genuine-concurrency case that `9-3-matching-cohorts.md §9.13` explicitly named as out of scope for the Unit/Vitest-mocked tier; it is in scope here because this tier runs against a real database connection pool, not sequential mocked calls |
| Never sets a pricing field, confirmed against real rows | seed as in the first case above | call `formOrJoinCohort(matchRequestId)` | query the created `Cohort` row directly — no `pricePerStudentPerHour`/`amount`-shaped column exists on `Cohort` at all (schema-level confirmation of the Unit tier's mock-based assertion at `9-3-matching-cohorts.md §9.5`) |

#### tutorExitContinuity — real multi-row spawn and FK integrity

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Spawns exactly one real MatchRequest row per affected membership | seed a real, active `ONE_TO_FIVE` `Cohort` with 5 real active `CohortMembership` rows across 5 real `StudentProfile` rows | call `tutorExitContinuity(cohortId, 'DROPOUT')` | querying `MatchRequest` afterward for rows with `path: PATH_C, status: PENDING_ADMIN_ASSIGNMENT` and a `studentId` among the 5 affected students returns **exactly 5** rows, each with a real, valid `subjectId`/`format` FK inherited from the ended cohort's context — not a count inferred from mock call arguments, an actual row count |
| Old Cohort and its memberships are real ENDED rows, not deleted | same setup | call `tutorExitContinuity(cohortId, 'SUSPENDED')` | the original `Cohort` row still exists (per `04-database-and-data-model.md`'s no-cascade-delete note) with `status: ENDED, endedReason: TUTOR_SUSPENDED`; all 5 original `CohortMembership` rows still exist with `status: ENDED, endReason: DROPPED_BY_ADMIN` — confirming the "carried forward, never deleted" design decision holds against the real schema's restrict-delete constraints, not just application-level intent |

---

### 9.17 Test Case Detail — adminMatching.service.persistence.test.ts

FRs: FR-MA-013–015, FR-MA-017. Traces to `04-database-and-data-model.md §4.2` (Cohort, CohortMembership, MatchRequest).

#### manuallyAssignTutor / manuallyAssembleGroup — real claim race

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Two admins racing the same single MatchRequest — real conditional-write guard, not a mocked one | seed a real `MatchRequest(status: PENDING_ADMIN_ASSIGNMENT)` and a real `VERIFIED` `TutorProfile` eligible for it | issue two concurrent `manuallyAssignTutor([matchRequestId], tutorId, adminId)` calls (two different `adminId`s, two separate DB connections) via `Promise.all` | exactly one call resolves with a `CohortAssignmentDTO`; the other throws `ApiError(409, "One or more of these requests have already been assigned")`. Querying the real DB afterward confirms **exactly one** `Cohort` was created from this `MatchRequest`, and the `MatchRequest.status` moved away from `PENDING_ADMIN_ASSIGNMENT` exactly once |
| Partial-array claim conflict rolls back atomically — no partial Cohort from the losing call | seed 5 real `MatchRequest` rows, all `PENDING_ADMIN_ASSIGNMENT`, all eligible for one tutor | first, call `manuallyAssignTutor([the 5 ids], tutorId, adminId)` and let it complete; then, using 4 of the same 5 (now-claimed) ids plus one never-before-seen fresh eligible `MatchRequest`, call `manuallyAssembleGroup([those 5 ids], tutorId2, adminId2)` | the second call throws `ApiError(409, ...)` for the entire call; querying the real DB confirms **no** new `Cohort` or `CohortMembership` row exists from the second, failed attempt — not even for the one `MatchRequest` in that call that was genuinely still available. This is a real-transaction assertion: a mocked Unit test can assert "the function threw," but only a real DB query can confirm the failed attempt left zero partial writes behind |

---

### 9.18 Coverage Honesty Check (persistence addendum — per PR Steward, at review time)

- [ ] Every case in §9.15–§9.17 above was run against an actually-reachable real test database at review time, not skipped/pending due to environment unavailability — a `Written before code?` checkbox alone does not satisfy this; the reviewer confirms the CI run that exercised these specific test files.
- [ ] The two concurrent-race cases (`formOrJoinCohort`'s last-seat race, `manuallyAssignTutor`'s claim race) were each observed to actually interleave at the database level at least once across repeated local runs — not merely "passed once," since a race condition test that only ever happens to serialize favorably is not exercising the guard it claims to.
- [ ] The FK-translation case (`selectTutor` with a non-existent `tutorId`) was verified against the *actual* error shape the implementation returns, not assumed to match `ApiError`'s documented shape without inspection.
- [ ] `9-3-matching-cohorts.md §9.13`'s out-of-scope note was updated to reflect that the two-way concurrent races are now covered here (cross-reference, not duplicate coverage claims).

---

### 9.19 Out of Scope for Automated Testing (and why)

- **`formatSwitch.service.ts`'s persistence behavior.** `requestSwitch`'s cohort-membership-ending and new-`MatchRequest`-creation writes do belong in this tier eventually, but its refund handoff (`refund.service.createPendingRefund`) is the half of that function with the highest real-money risk, and that logic is owned and persistence-tested by `payments-earnings` (Review §6.2 item 2, Phase 5.2). Splitting `formatSwitch`'s persistence coverage across two phases whose sequencing isn't yet fixed by the locked plan risks either duplicate or missing coverage; deferred to a follow-up sub-phase once 5.2 is drafted and the seam between the two is concrete, rather than guessed at here.
- **`selectTutor`'s double-submit race (§9.15 above) is covered here as a genuine addition beyond Review §6.2's literal item-1 wording**, because it is the same class of capacity-limited race Phase 4 already flagged for `formOrJoinCohort` and `manuallyAssignTutor` — Path A's "one active match per student" invariant is exactly as real-concurrency-sensitive as Path C's seat count. Flagged here explicitly (rather than silently added) since it was not itemized in the original Phase 4 Unit-tier case list for this function.
- **Sustained multi-client load testing** (dozens of concurrent joins against a single near-full cohort, sustained webhook-storm-style concurrency) remains genuinely out of scope for this Vitest-based suite, consistent with `9-3-matching-cohorts.md §9.13`'s existing note — the two-way races above prove the guard clause is real, not merely mock-shaped, but they are not a substitute for a dedicated load/performance testing pass (Review §3.8, deferred as a low-priority, separately-tracked item).
- **Real notification delivery, job/interval scheduling.** Same reasoning and same exclusions as `9-3-matching-cohorts.md §9.13` — unchanged by this file's addition.

---

**Back to:** [9-3. Backend Test Documentation: Matching & Cohorts] · **Next:** proceed to → Phase 5.2 (`9-7-payments-earnings-persistence.md`)
