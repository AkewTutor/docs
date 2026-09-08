## Project: AKEWTutor — Backend Test Documentation: Payments & Earnings — Integration (Persistence)

**Sibling of:** [9-7. Backend Test Documentation: Payments & Earnings] (Unit + Integration (HTTP contract) tiers live there). **Links back to:** [04. Database & Data Model §4.2 (PricingConfig, Payment, Refund, TutorEarning, Payout)], [04 §4.4 (Indexes & Constraints)], [8-7. Function-Level Spec: Payments & Earnings].
**Created by:** Phase 5.2 of `09-redesign-implementation-plan.md`, per Review §6.2 item 2.

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### Why this file exists, and what it does not duplicate

`9-7-payments-earnings.md`'s Unit tier already covers the proration formula, the reduced-makeup rate, the atomic-transaction *shape* of `createAndActivateConfig`, and — since Phase 4 — the audit-log *call* for `approveRefund`/`rejectRefund` (§9.12), all against a mocked Prisma client. As with matching-cohorts (see `9-3-matching-cohorts-persistence.md`), a mock proves the code asks the database the right question; it cannot prove three things that matter enormously for a money-handling module: that a webhook replayed twice really doesn't double-credit anything, that two admins racing to approve the same refund can't both win, and that an audit entry claimed by a mock actually lands durably.

**In scope (per Review §6.2 item 2):** payment status transitions, webhook idempotent processing against real rows, and payout calculation against real ledger rows — plus the refund-approval audit-log persistence check that `00-agent-rules.md`'s own audit-log convention explicitly assigns to this tier ("a separate, later test asserts the row actually lands in a real `AuditLog` table").

**Deliberately not in this file** — see §9.24 below for the full reasoning:
- `paymentPause.service.ts`'s persistence behavior (lower risk, and its `rescheduleSessionsDuringPause` side effects reach into `class-delivery-library`, whose own persistence tier isn't yet scheduled — see the cross-module note below).
- `promotion.service.ts` (simple CRUD, no concurrency or ledger risk that a Unit-tier mock can't already prove).

---

### Test environment convention (binding on every test file in this document)

1. **Real database, no silent fallback.** Per `00-agent-rules.md` Rule 7, a test in this file that cannot reach the configured test database fails loudly at setup — it never falls back to a mocked client.
2. **Isolation between tests.** Each test resets the tables it touches (transactional rollback wrapper or truncate-and-reseed); no cross-test row-state dependencies.
3. **Real Prisma client, real service functions**, called exactly as production code calls them.
4. **True third parties only are mocked.** `chapa.client.ts`'s outbound HTTP calls (`initiateCheckout`, `verifyWebhookSignature`'s underlying HMAC check may be exercised for real since it's pure computation, but the network call itself) remain mocked here, exactly as in the Unit tier — Chapa is a genuine external system, not an internal seam this tier exists to prove.
5. **Cross-module seam note — `session.service.generateSessionsForCohort`.** `handleChapaWebhook`'s `SUCCESS` path calls into `class-delivery-library`'s `session.service.ts` to generate the schedule. That module's own persistence tier is not in the Phase 5 priority order (Review §6.2 lists only matching-cohorts, payments-earnings, accounts-guardianship, shared-config, gamification-engagement). Rather than silently deciding this seam doesn't matter, this file mocks `generateSessionsForCohort` — the same treatment a true third party gets — and states plainly that **this specific cross-module call is not persistence-tested until `class-delivery-library` gets its own tier**; the call-count/call-args assertions on it below only confirm this module's own idempotency guard fires correctly, not that the downstream schedule is actually created correctly in a real DB. This boundary should be revisited once `class-delivery-library`'s tier exists.
6. **Cross-module fixture usage.** Antecedent rows (`User`, `StudentProfile`, `TutorProfile`, `Subject`, `MatchRequest`, `Cohort`, `CohortMembership`) are seeded via `accounts-guardianship.factory.ts` and `matching-cohorts.factory.ts` before this module's own factories (`payments-earnings.factory.ts`) are used — per `00-test-fixtures.md §1.1`, every FK is a real row's id.

---

### 9.23 Test File Map (persistence)

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/services/payment.service.ts | tests/integration/payment.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/pricing.service.ts | tests/integration/pricing.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/refund.service.ts | tests/integration/refund.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/earning.service.ts | tests/integration/earning.service.persistence.test.ts | Integration (persistence) | ☐ |
| src/services/payout.service.ts | tests/integration/payout.service.persistence.test.ts | Integration (persistence) | ☐ |

---

### 9.24 Test Case Detail — payment.service.persistence.test.ts

FRs: FR-PB-001–004. Traces to `04-database-and-data-model.md §4.2` (Payment, CohortMembership).

#### initiatePayment / handleChapaWebhook — real status transitions

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Real Payment row moves PENDING → SUCCESS | seed a real `CohortMembership(status: PENDING_PAYMENT)` and an active `PricingConfig`; call `initiatePayment` for real, capturing the real created `Payment.id`; mock only `chapa.client.verifyWebhookSignature` → `true` | call `handleChapaWebhook(rawBody, signature)` with `event: SUCCESS` referencing that real `Payment.id` | a fresh query shows `Payment.status: SUCCESS`; the real `CohortMembership.billingCycleAnchorDate` is now set to a real timestamp — both read back from the database, not from either call's return value |
| **Idempotent webhook replay — real row state proves no double-processing** | same as above, webhook already delivered once and the `Payment` is already `SUCCESS` in the real DB | call `handleChapaWebhook(rawBody, signature)` a second time with the byte-identical payload | `Payment.status` is still `SUCCESS` (a fresh query, not just "no error thrown"); `CohortMembership.billingCycleAnchorDate` is byte-identical to its value after the first call (not reset, not advanced); the mocked `session.service.generateSessionsForCohort` was invoked **exactly once** across both calls combined — this is the specific case Review §6.2 item 2 names, and the one a mocked-Prisma Unit test structurally cannot fully back, since a mock's "already SUCCESS" state is only ever what the test told it to be, never a state the function itself had to durably reach and then read back |
| FK integrity — a Payment can't reference a non-existent membership | — | attempt `initiatePayment(callerId, callerRole, randomUUID())` for a `cohortMembershipId` with no real row | throws a clean `ApiError(404, "...")`, not an unhandled Prisma FK error — confirms the real 404 lookup-before-write path, not merely a database-level rejection surfacing raw |

---

### 9.25 Test Case Detail — pricing.service.persistence.test.ts

FRs: FR-AD-009. Traces to `04-database-and-data-model.md §4.2` (PricingConfig — "partial unique... enforced at the application layer") and §4.4.

#### createAndActivateConfig — real atomic swap under concurrency

| Case | Setup | Action | Expected result |
|---|---|---|---|
| The deactivate-then-activate swap is a real single transaction | seed a real active `PricingConfig(format: ONE_TO_ONE)` | call `createAndActivateConfig('ONE_TO_ONE', input, adminId)` | a fresh query for `PricingConfig` rows where `format: ONE_TO_ONE, isActive: true` returns **exactly one** row — the newly created one; the previous row is now `isActive: false` in the same query |
| **Two admins racing to activate a new config for the same format — never two active rows, never zero** | seed a real active `PricingConfig(format: ONE_TO_THREE)` | issue two concurrent `createAndActivateConfig('ONE_TO_THREE', inputA, adminA)` / `(..., inputB, adminB)` calls (`Promise.all`, two separate DB connections) | exactly one call's new config ends up `isActive: true`; querying the real DB afterward for `format: ONE_TO_THREE, isActive: true` returns **exactly one** row, never two and never zero, regardless of which admin's write the database serialized first. This is the persistence-tier promotion of the exact risk the Unit tier's `$transaction`-call assertion (`9-7-payments-earnings.md §9.9`) can only prove the *shape* of, not the *outcome* of under real concurrent access — precisely the "looks correct against mocks, fails against real transaction ordering" pattern Review §6.2 item 4 calls out for `shared-config`'s token rotation, applying identically here |

---

### 9.26 Test Case Detail — refund.service.persistence.test.ts

FRs: FR-PB-007, FR-AD-012. Traces to `04-database-and-data-model.md §4.2` (Refund) and `00-agent-rules.md`'s audit-log convention.

#### approveRefund / rejectRefund — real state transitions and real claim race

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Approval persists a real, terminal state | seed a real `PENDING` `Refund` row meeting policy conditions | call `approveRefund(refundId, adminId)` | a fresh query shows `Refund.status: APPROVED, approvedById: adminId, approvedAt` set — read from the database, not the function's return value |
| **Two admins racing to action the same PENDING refund — only one wins** | seed a real `PENDING` `Refund` row | issue `approveRefund(refundId, adminA)` and `rejectRefund(refundId, adminB, "reason")` concurrently (`Promise.all`, two connections) | exactly one call succeeds; the other throws `ApiError(409, "This refund has already been actioned")`. A fresh query confirms the `Refund` row landed in exactly one terminal state (`APPROVED` **or** `REJECTED`, never both fields populated, never left `PENDING`) — this is the real-DB proof behind the conditional-write guard the Unit tier (`9-7-payments-earnings.md §9.12`) only exercises against sequential mocked calls |
| **Audit log entry is durably persisted on approval, not just "called"** | seed a real `PENDING` refund | call `approveRefund(refundId, adminId)` | a fresh query against the real `AuditLog` store (whichever concrete form the implementation chose — a Prisma table or a structured log sink, per `00-agent-rules.md`'s note that this convention doesn't prescribe which) returns an entry with `actor: adminId, action: 'REFUND_APPROVED', target: refundId`, and a real, persisted `timestamp` — this is the specific escalation `00-agent-rules.md` names for this tier: the Phase 4 Unit-tier test (`9-7-payments-earnings.md §9.12`) only proves `record(...)` was *called*; a mock can report success even if the underlying write never durably lands. This test is the one that actually proves it lands |
| **Audit log entry is durably persisted on rejection, including traceability to the reason** | seed a real `PENDING` refund | call `rejectRefund(refundId, adminId, "Student-caused disruption")` | the real `AuditLog` store contains an entry with `action: 'REFUND_REJECTED', target: refundId`; separately, a fresh query on the `Refund` row itself confirms `rejectionReason: "Student-caused disruption"` — the audit entry and the domain row are checked as two independent real reads, not inferred from one another |

---

### 9.27 Test Case Detail — earning.service.persistence.test.ts

FRs: FR-TU-018, FR-TU-019, FR-MK-009. Traces to `04-database-and-data-model.md §4.2` (TutorEarning — `sessionId` unique) and §4.4.

#### creditEarning — real unique-constraint backstop

| Case | Setup | Action | Expected result |
|---|---|---|---|
| The `TutorEarning(sessionId)` unique constraint is real, not merely documented | seed a real `ScheduledSession` and `TutorProfile`; insert one `TutorEarning(sessionId, ...)` row directly via Prisma | attempt a second direct Prisma insert with the identical `sessionId` | the second insert is rejected by the real database's unique constraint (`P2002`) — a schema-level confirmation, independent of `creditEarning`'s own application-level "check before write" guard (already Unit-tested at `9-7-payments-earnings.md §9.14`) |
| **`creditEarning` called twice concurrently for the same session never produces two earning rows** | seed a real `ScheduledSession`, no existing `TutorEarning` for it | issue two concurrent `creditEarning(sessionId, tutorId, 'FULL')` calls (`Promise.all`, two connections — simulating a session's completion handler firing twice, e.g. a retried job) | exactly one call resolves a new `TutorEarningDTO`; the other either resolves the same, already-created row (idempotent read) or throws a translated `ApiError` on the constraint conflict — whichever the implementation chooses, a fresh query confirms **exactly one** `TutorEarning` row exists for that `sessionId`, never two. This is the real-DB backstop behind the Unit tier's mocked "check before write" case, and matters specifically because the application-level check-then-write is itself a two-step operation that a real concurrent request can interleave inside — exactly the same class of gap the review flags for `formOrJoinCohort` in matching-cohorts |

---

### 9.28 Test Case Detail — payout.service.persistence.test.ts

FRs: FR-TU-019, FR-AD-011. Traces to `04-database-and-data-model.md §4.2` (Payout, TutorEarning).

#### generateMonthlyPayouts — real ledger aggregation

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Payout totals are computed from real, seeded ledger rows | seed one real `TutorProfile` with 4 real, unpaid `TutorEarning` rows in the target period (e.g. `"175.00"` × 3 + `"87.50"` × 1) | call `generateMonthlyPayouts(periodStart, periodEnd)` | a fresh query shows exactly one new `Payout(status: PENDING)` row for that tutor with `totalAmount` equal to the real sum of the 4 seeded rows (`"612.50"`), computed via the project's decimal library against the actual persisted `Decimal` values — not a value asserted from mocked aggregate output |
| Batched earnings are really linked, preventing a second run from double-counting | same seed, first `generateMonthlyPayouts` run already completed | call `generateMonthlyPayouts(periodStart, periodEnd)` again for the identical period | a fresh query confirms the same 4 `TutorEarning` rows now have `payoutId` set to the first run's `Payout.id`; the second run creates **no** additional `Payout` row for this tutor — this is the persistence-tier confirmation of the Unit tier's mocked "second run creates no additional payouts" case (`9-7-payments-earnings.md §9.16`), now checked against rows that a second run would have had to actually re-query and correctly exclude, not a mock told in advance what to return |

---

### 9.29 Coverage Honesty Check (persistence addendum — per PR Steward, at review time)

- [ ] Every case in §9.24–§9.28 above was run against an actually-reachable real test database at review time — confirmed by the CI run, not inferred from the checkboxes alone.
- [ ] The two concurrent-race cases (`createAndActivateConfig`'s activation swap, `approveRefund`/`rejectRefund`'s claim race, `creditEarning`'s double-fire) were each observed to genuinely interleave at the database level at least once across repeated local runs, not merely "passed once."
- [ ] The audit-log persistence cases were verified against whatever concrete storage the implementation actually chose for `AuditLog` (table or log sink) — not assumed to be a Prisma model without checking.
- [ ] The webhook-idempotency case's "called exactly once" assertion on `generateSessionsForCohort` was checked across **both** webhook calls combined, not reset/re-mocked between them (which would silently hide a double-call).
- [ ] No monetary value asserted in §9.28 was hand-typed without first being derived from the actual seeded `TutorEarning.amount` values in the test's own setup.

---

### 9.30 Out of Scope for Automated Testing (and why)

- **`class-delivery-library`'s real schedule generation**, as the seam triggered by `handleChapaWebhook`'s `SUCCESS` path. Mocked here (see the environment convention above) because that module's own persistence tier isn't in the current Phase 5 priority order. This is a genuine, explicitly-flagged gap — not a silent one — and should be closed by adding a cross-module persistence case once `class-delivery-library` gets its own tier, rather than this file quietly assuming a mock is equivalent to the real thing.
- **`paymentPause.service.ts` persistence behavior.** Lower money-integrity risk than the five services covered above (it reschedules rather than moves money), and its own downstream effects reach into the same not-yet-scheduled `class-delivery-library` seam. Deferred, not silently dropped.
- **`promotion.service.ts` persistence behavior.** No concurrency-sensitive write and no ledger risk beyond what the Unit tier (mocked Prisma) already proves correctly — a real-DB pass here would mostly re-prove CRUD works, which is not where this tier's value lies.
- **Real Chapa network behavior**, unchanged from `9-7-payments-earnings.md §9.22` — this tier still mocks the actual outbound HTTP call; only the internal DB-facing behavior around it is real here.
- **The actual outbound money movement for an `APPROVED` refund** — unchanged from `9-7-payments-earnings.md §9.22`; this remains a Chapa-side integration to confirm at build time, not a function this test suite models.

---

**Back to:** [9-7. Backend Test Documentation: Payments & Earnings] · **Next:** proceed to → Phase 5.3 (`9-2-accounts-guardianship-persistence.md`)
