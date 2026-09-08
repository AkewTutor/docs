## Project: AKEWTutor — Backend Test Documentation: Accounts & Guardianship — Integration (Persistence)

**Sibling of:** [9-2. Backend Test Documentation: Accounts & Guardianship] (Unit + Integration (HTTP contract) tiers live there). **Links back to:** [04. Database & Data Model §4.2 (StudentProfile, ParentStudentRelationship)], [04 §4.4], [8-2. Function-Level Spec: Accounts & Guardianship].
**Created by:** Phase 5.3 of `09-redesign-implementation-plan.md`, per Review §6.2 item 3.

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### Why this file exists, and what it does not duplicate

`9-2-accounts-guardianship.md`'s Unit tier already proves, against a mocked Prisma client, that `handleSoleGuardianRemoval` sets `GUARDIAN_REQUIRED_HOLD` and makes no delete call, and that `auditLog.service.record` is called for `GUARDIAN_REMOVED`. It cannot prove that a real database round-trip actually leaves every one of the student's real, related rows untouched, that the `(parentId, studentId)` and `inviteToken` uniqueness guarantees the schema promises are real constraints, or that an audit entry a mock reported as "called" actually lands.

**In scope (per Review §6.2 item 3):** the guardian-removal → `GUARDIAN_REQUIRED_HOLD` transition, the sole-guardian edge case, and cascading effects on active cohorts — the last of which, on inspection of `04-database-and-data-model.md` and `8-2-accounts-guardianship.md`, maps most concretely to `adminPeople.service.ts`'s `suspendAccount` computing `affectedCohortIds` against a Tutor's real active `CohortMembership` rows (see the scope note below on why the guardian-removal path itself has no automatic cohort-level cascade to test).

**Deliberately not in this file:**
- `tutorProfile.service.ts`, `availability.service.ts`, `subject.service.ts`, `adminTutorVerification.service.ts` — none carry the concurrency or cross-entity-integrity risk this tier exists to catch; their Unit-tier mocked coverage is not meaningfully strengthened by a real-DB pass.
- The cross-feature call sites of `assertAccountStatusAllowsAccess` inside `matching-cohorts` and `class-delivery-library` — those are that feature's own persistence concern (`9-3-matching-cohorts-persistence.md`, and `class-delivery-library` has no persistence tier per the Phase 5 priority order); this file tests only that `assertAccountStatusAllowsAccess` itself reads the real, persisted `accountStatus` correctly (§9.23a below), not every caller of it.

---

### Test environment convention (binding on every test file in this document)

1. **Real database, no silent fallback**, per `00-agent-rules.md` Rule 7.
2. **Isolation between tests** (transactional rollback or truncate-and-reseed).
3. **Real Prisma client, real service functions.**
4. **Cross-module fixture usage.** `accounts-guardianship.factory.ts`'s own entities FK into `shared-config`'s `User`; `matching-cohorts.factory.ts`'s `Cohort`/`CohortMembership` are seeded here (real, cross-module) wherever a case needs a real active cohort to check cascade behavior against — per `00-test-fixtures.md §1.1`, every FK is a real row's id.
5. **Gap closed — named enforcement function now exists.** `assertAccountStatusAllowsAccess(studentId)` in `studentProfile.service.ts` is the single, named gate for `GUARDIAN_REQUIRED_HOLD` (see `04-database-and-data-model.md §4.2`, `8-2-accounts-guardianship.md`). §9.23a below adds the real-DB test for the function itself; the Unit-tier mocked call-site tests for its two callers (`matching.service.ts`'s three booking-entry functions, `session.service.ts`'s `assertSessionAccessAllowed`) live in `9-3-matching-cohorts.md` and `9-4-class-delivery-library.md` respectively, per Rule 7's "test the thing where it's owned" convention.

---

### 9.22 Test File Map (persistence)

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/services/guardianship.service.ts | tests/integration/guardianship.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/adminPeople.service.ts | tests/integration/adminPeople.service.persistence.test.ts | Integration (persistence) | ☐ |

---

### 9.23 Test Case Detail — guardianship.service.persistence.test.ts

FRs: FR-AC-002–008. Traces to `04-database-and-data-model.md §4.2` (StudentProfile, ParentStudentRelationship).

#### activateInvite — real uniqueness and real double-fire guard

| Case | Setup | Action | Expected result |
|---|---|---|---|
| The `inviteToken` unique constraint is real | seed a real `ParentStudentRelationship` with a known `inviteToken` | attempt a second direct Prisma insert of another `ParentStudentRelationship` reusing the identical `inviteToken` | rejected by the real unique constraint (`P2002`) — schema-level confirmation, independent of any service function |
| The `(parentId, studentId)` unique composite is real | seed a real `ParentProfile` and `StudentProfile` with one `ParentStudentRelationship` already linking them | attempt a second direct Prisma insert linking the identical pair again | rejected by the real unique constraint |
| **Two near-simultaneous activation attempts on the same valid token create exactly one real User/StudentProfile pair** | seed a real, unexpired `ParentStudentRelationship(status: INVITED)` | issue two concurrent `activateInvite(token, password)` calls (`Promise.all`, two DB connections) | exactly one call resolves `{ accessToken, studentId, relationshipStatus: 'ACTIVE' }`; the other throws (a 404/409, whichever the implementation chose per the Unit tier's case). A fresh query confirms **exactly one** real `User` row and **exactly one** real `StudentProfile` row exist for this relationship — never two — and `ParentStudentRelationship.status` is `ACTIVE` exactly once, not toggled twice. This is the real-DB promotion of the Phase 4 Unit-tier case (`9-2-accounts-guardianship.md §9.6`), which could only assert the guard's *shape* against a mock whose "already consumed" state was told to it in advance, not reached by the function itself |

#### revokeOrModifyRelationship / handleSoleGuardianRemoval — real state transition and real no-cascade proof

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sole-guardian removal persists `GUARDIAN_REQUIRED_HOLD` for real | seed a real `StudentProfile`, a real `ParentProfile`, exactly one real `ParentStudentRelationship(relationshipType: MANDATORY_GUARDIAN, status: ACTIVE)` linking them | call `revokeOrModifyRelationship(parentId, 'PARENT', relationshipId, { revoke: true })` | a fresh query shows `StudentProfile.accountStatus: GUARDIAN_REQUIRED_HOLD` and the `ParentStudentRelationship.status: REVOKED, revokedAt, revokedById` set — all read from the database, not the function's return value |
| **No real row is deleted or orphaned across every FK-linked entity** | same setup, plus seed a real active `CohortMembership` for the student, one real `XPLedgerEntry`, and one real `Streak` row | call `revokeOrModifyRelationship(...)` (sole-guardian path) | fresh queries confirm the `StudentProfile` row, the `CohortMembership` row (still `ACTIVE`, untouched), the `XPLedgerEntry` row, and the `Streak` row **all still exist**, byte-identical apart from `StudentProfile.accountStatus` itself — this is the real-DB backstop behind the Unit tier's mocked "no delete call was made" assertion (`9-2-accounts-guardianship.md §9.6`/§9.20): a mock can only prove the code *didn't call* delete; only a real read proves nothing was *actually* removed, whether by this function or by an unexpected cascade configured elsewhere in the schema |
| Non-existent relationship id is rejected cleanly | seed a real caller with no matching relationship at all | call `revokeOrModifyRelationship(parentId, 'PARENT', randomUUID(), { revoke: true })` | throws a clean `ApiError(403, ...)` or `ApiError(404, ...)` (per the Unit tier's IDOR-vs-not-found convention) — not an unhandled Prisma "record not found" error |
| **Sole-guardian removal's audit log entry is durably persisted** | same sole-guardian setup as above | call `revokeOrModifyRelationship(...)` | a fresh query against the real `AuditLog` store returns an entry with `actor: parentId, action: 'GUARDIAN_REMOVED', target: relationshipId`, and a real persisted `timestamp` — escalating the Phase 4 Unit-tier "record() was called" assertion (`9-2-accounts-guardianship.md §9.6`) to a durability proof, per `00-agent-rules.md`'s audit-log convention |

#### assertAccountStatusAllowsAccess — real gate, read against a real row (**gap closed**)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Blocks against a real, persisted `GUARDIAN_REQUIRED_HOLD` row | run the sole-guardian removal case above first, so `StudentProfile.accountStatus` is really `GUARDIAN_REQUIRED_HOLD` in the database (not mocked) | call `assertAccountStatusAllowsAccess(studentId)` directly | throws `ApiError(403, ...)` — proves the gate reads the real column value, not a value handed to it by a mock, closing the exact spec gap this document previously flagged in its scope note |
| Blocks against a real, persisted `PENDING_ACTIVATION` row | seed a real Grades 1–5 `StudentProfile` with no activated guardian yet (`accountStatus: PENDING_ACTIVATION` by schema default) | call `assertAccountStatusAllowsAccess(studentId)` | throws the identical `ApiError(403, ...)` |
| Resolves silently once the hold is lifted | starting from the first case, seed a fresh, real, `ACTIVE` `ParentStudentRelationship` for the same student and confirm `StudentProfile.accountStatus` reads back `ACTIVE` | call `assertAccountStatusAllowsAccess(studentId)` again | resolves with no error — a real round-trip proof that a lifted hold genuinely restores access, not just that the enum value changed in isolation |

---

### 9.24 Test Case Detail — adminPeople.service.persistence.test.ts

FRs: FR-AD-001, FR-AD-003. Traces to `04-database-and-data-model.md §4.2` (Cohort, CohortMembership) and `8-2-accounts-guardianship.md`'s `suspendAccount` side-effect note.

#### suspendAccount — real cascading-effect computation on active cohorts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| **`affectedCohortIds` reflects real, currently-active cohorts only** | seed a real `TutorProfile` with 3 real `Cohort` rows: two `ACTIVE` (with real `CohortMembership` rows), one already `ENDED` | call `suspendAccount(tutorId, adminId, "Policy violation", 'SUSPENDED')` | the resolved `affectedCohortIds` contains exactly the two `ACTIVE` cohort ids — the `ENDED` one is excluded — computed from a real query against real rows, not a mocked list the test supplied in advance |
| Suspension is real and durable | same setup | call `suspendAccount(...)` | a fresh query on the target `User`/`TutorProfile` row shows the restriction field actually persisted — read back independently of the function's return value |
| **The suspension audit entry is durably persisted** | same setup | call `suspendAccount(tutorId, adminId, "Policy violation", 'SUSPENDED')` | a fresh query against the real `AuditLog` store returns an entry with `actor: adminId, action: 'ACCOUNT_SUSPENDED', target: tutorId` — escalating the Phase 4 Unit-tier assertion (`9-2-accounts-guardianship.md §9.18`) to a durability proof, consistent with the same treatment given to `GUARDIAN_REMOVED` above and `REFUND_APPROVED`/`REFUND_REJECTED` in `9-7-payments-earnings-persistence.md` |
| Suspending a Tutor with zero active cohorts computes an empty set from real data, not an omitted query | seed a real `TutorProfile` with only `ENDED` cohorts | call `suspendAccount(tutorId, adminId, reason, 'SUSPENDED')` | `affectedCohortIds` is genuinely absent from the response (per the existing field-presence convention), confirmed against a real query that found zero active rows — not a code path that skipped the query entirely and merely assumed none exist |

---

### 9.25 Coverage Honesty Check (persistence addendum — per PR Steward, at review time)

- [ ] Every case in §9.23–§9.24 above was run against an actually-reachable real test database at review time.
- [ ] The `activateInvite` double-fire case was observed to genuinely interleave at the database level at least once across repeated local runs, not merely "passed once."
- [ ] The "no row deleted or orphaned" case queried **every** FK-linked entity listed (`CohortMembership`, `XPLedgerEntry`, `Streak`) independently — not inferred from one entity's survival that the others also survived.
- [ ] Both audit-persistence cases (`GUARDIAN_REMOVED`, `ACCOUNT_SUSPENDED`) were verified against whatever concrete storage the implementation chose for `AuditLog`, matching the same verification already done in `9-7-payments-earnings-persistence.md` and `9-3-matching-cohorts-persistence.md` — no case here re-invents a different verification method than those two established.
- [ ] The `affectedCohortIds` case's real `Cohort`/`CohortMembership` fixtures included at least one genuinely `ENDED` cohort in the setup, so the test could actually fail if the query stopped filtering by active status — a setup with only active cohorts would pass even against a regression that dropped the status filter entirely.

---

### 9.26 Out of Scope for Automated Testing (and why)

- **Access-pause enforcement at booking/class-access check time.** As noted above, `04-database-and-data-model.md` documents this as an application-layer guarantee, but no function across Docs 06/08 currently implements or names such a check. Testing it here would mean inventing a function this spec doesn't yet define — a design decision outside this phase's authority per the locked plan's "do not re-litigate structural decisions" rule. Flagged for Phase 7's human review checklist as a genuine spec gap: either a named enforcement point needs to be added to `08-function-level-specification/backend/8-4-class-delivery-library.md`'s session-join path (the most likely home) and then persistence-tested here in a follow-up, or the guarantee in Doc 04 needs to be softened to reflect where it's actually enforced today.
- **`inviteReminder.job.ts`'s day-7 reminder persistence** — unchanged from `9-2-accounts-guardianship.md §9.21`; the reminder-sent tracking column doesn't yet have a settled name in Doc 8-2, so there's no concrete field to assert against yet.
- **Real image upload for `profilePictureUrl`** — unchanged from `9-2-accounts-guardianship.md §9.21`.

---

**Back to:** [9-2. Backend Test Documentation: Accounts & Guardianship] · **Next:** proceed to → Phase 5.4 (`9-1-shared-config-persistence.md`)
