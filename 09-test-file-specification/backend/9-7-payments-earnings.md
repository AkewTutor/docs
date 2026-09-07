## Project: AKEWTutor — Backend Test Documentation: Payments & Earnings
**Links back to:** [05a. Backend Folder & File Structure §7], [8-7. Function-Level Spec: Payments & Earnings]
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.5 (Chapa webhook auth exception) and §0.4 (payment-pause reschedule and monthly payout batch are job/event-driven, exposed here only as read state). See also Doc 02 Section 13's sessions-delivered proration formula and monetary-rounding rule (M7 fix).

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** PricingConfig, Payment, PaymentPause, Refund, TutorEarning, Payout, PromotionCode. **Depends on:** `matching-cohorts`, `class-delivery-library` (hard — `TutorEarning` → `ScheduledSession`).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| chapa.client.ts | FR-PB-001 | NFR-007 |
| payment.service.ts | FR-PB-001–004, FR-PB-008, FR-PB-009 (anchor-only path) | NFR-007 |
| paymentPause.service.ts | FR-PB-005, FR-PB-009 | — |
| pricing.service.ts | FR-AD-009, Section 7 Definition of Done | — |
| refund.service.ts | FR-PB-007, FR-AD-012, FR-SP-048, Section 13 proration formula | — |
| earning.service.ts | FR-MK-009, FR-AD-011, FR-TU-019 | — |
| payout.service.ts | FR-TU-019, FR-AD-011 | — |
| promotion.service.ts | FR-AD-016 | — |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/utils/providers/chapa.client.ts | tests/utils/providers/chapa.client.test.ts | Unit (mocked Chapa SDK/HTTP call) | ☐ |
| src/schemas/payment.schema.ts | tests/schemas/payment.schema.test.ts | Unit | ☐ |
| src/services/payment.service.ts | tests/services/payment.service.test.ts | Unit (mocked Prisma, mocked `chapa.client.ts`, mocked `promotion.service.ts`, mocked `class-delivery-library`'s `session.service.generateSessionsForCohort`) | ☐ |
| src/controllers/payment.controller.ts | tests/controllers/payment.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/payment.routes.ts | tests/routes/payment.routes.test.ts | Integration (supertest, raw-body-aware for the webhook route) | ☐ |
| src/services/paymentPause.service.ts | tests/services/paymentPause.service.test.ts | Unit (mocked Prisma, mocked `session.service.ts`) | ☐ |
| src/controllers/paymentPause.controller.ts | tests/controllers/paymentPause.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/paymentPause.routes.ts | tests/routes/paymentPause.routes.test.ts | Integration (supertest) | ☐ |
| src/schemas/pricing.schema.ts | tests/schemas/pricing.schema.test.ts | Unit | ☐ |
| src/services/pricing.service.ts | tests/services/pricing.service.test.ts | Unit (mocked Prisma, real `Decimal` arithmetic) | ☐ |
| src/controllers/pricing.controller.ts | tests/controllers/pricing.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/pricing.routes.ts | tests/routes/pricing.routes.test.ts | Integration (supertest) | ☐ |
| src/schemas/refund.schema.ts | tests/schemas/refund.schema.test.ts | Unit | ☐ |
| src/services/refund.service.ts | tests/services/refund.service.test.ts | Unit (mocked Prisma, real `Decimal` arithmetic) | ☐ |
| src/controllers/refund.controller.ts | tests/controllers/refund.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/refund.routes.ts | tests/routes/refund.routes.test.ts | Integration (supertest) | ☐ |
| src/services/earning.service.ts | tests/services/earning.service.test.ts | Unit (mocked Prisma, real `Decimal` arithmetic) | ☐ |
| src/controllers/earning.controller.ts | tests/controllers/earning.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/earning.routes.ts | tests/routes/earning.routes.test.ts | Integration (supertest) | ☐ |
| src/services/payout.service.ts | tests/services/payout.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/payout.controller.ts | tests/controllers/payout.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/payout.routes.ts | tests/routes/payout.routes.test.ts | Integration (supertest) | ☐ |
| src/schemas/promotion.schema.ts | tests/schemas/promotion.schema.test.ts | Unit | ☐ |
| src/services/promotion.service.ts | tests/services/promotion.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/promotion.controller.ts | tests/controllers/promotion.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/promotion.routes.ts | tests/routes/promotion.routes.test.ts | Integration (supertest) | ☐ |
| src/jobs/paymentReminder.job.ts | — | Underlying logic covered via `notification.service.test.ts` (shared-config) call assertions; interval wrapper excluded | — |
| src/jobs/monthlyPayout.job.ts | — | Underlying logic covered via `payout.service.test.ts`'s `generateMonthlyPayouts`/idempotency cases; interval wrapper excluded | — |

---

### 9.2 Test Case Detail — chapa.client.test.ts

**OWASP: A10:2021 – Server-Side Request Forgery (outbound checkout call — destination host must be fixed, not attacker-influenced), A02:2021 – Cryptographic Failures (webhook signature verification correctness).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| initiateCheckout targets the fixed Chapa host | mock the underlying HTTP client | call `initiateCheckout(amount, reference, callbackUrl)` | assert the request target is the hardcoded Chapa API base URL — none of `amount`/`reference`/`callbackUrl` (ultimately traceable to caller/booking data) can influence *where* the request is sent, only its payload (OWASP A10) |
| initiateCheckout returns the checkout URL | mock a successful Chapa response | call `initiateCheckout(...)` | resolves `{ checkoutUrl }` |
| verifyWebhookSignature accepts a valid signature | sign a raw body with the configured webhook secret | call `verifyWebhookSignature(rawBody, validSignatureHeader)` | resolves `true` |
| verifyWebhookSignature rejects a tampered body | sign a raw body, then mutate a single byte of `rawBody` before verifying | call `verifyWebhookSignature(mutatedBody, originalSignatureHeader)` | resolves `false` — the signature must be computed over the exact bytes received, not a re-serialized/parsed version (OWASP A02 — a classic HMAC verification bypass if verification is done against re-encoded JSON instead of the raw buffer) |
| verifyWebhookSignature rejects a missing/malformed header | — | call `verifyWebhookSignature(rawBody, undefined)` and `verifyWebhookSignature(rawBody, "not-a-real-signature")` | both resolve `false`, never throw an unhandled exception |

---

### 9.3 Test Case Detail — payment.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| initiatePaymentSchema requires a valid cohortMembershipId | — | parse `{ body: { cohortMembershipId: "not-a-uuid" } }` | fails |
| initiatePaymentSchema's promotionCode is optional | — | parse `{ body: { cohortMembershipId: uuid } }` (no `promotionCode`) | passes |
| Mass-assignment guard on amount | — | parse `{ body: { cohortMembershipId: uuid, amount: "0.01" } }` | the unknown `amount` key is stripped/rejected — confirms a client cannot pass its own price, since `amount` is always server-derived from the active `PricingConfig` (OWASP A08:2021 — mass assignment onto a monetary field) |

---

### 9.4 Test Case Detail — payment.service.test.ts

FRs: FR-PB-001–004, FR-PB-008. NFRs: NFR-007. **OWASP: A01:2021 – Broken Access Control (membership/ownership scoping), A04:2021 – Insecure Design (server-locked pricing is a business-integrity control, not just a validation nicety).**

#### initiatePayment

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Begins a Chapa checkout for an Admin-approved membership | mock `CohortMembership` awaiting payment, active `PricingConfig` | call `initiatePayment(callerId, callerRole, cohortMembershipId)` | resolves `{ paymentId, chapaCheckoutUrl, amount, status: 'PENDING' }`; creates `Payment(status: PENDING)` |
| Amount is locked in at initiation time | mock `PricingConfig` active now at rate X | call `initiatePayment(...)`, then mock an Admin price change to rate Y, then re-fetch the same `Payment` | the stored `Payment.amount` remains derived from rate X — never retroactively affected by the later change (Doc 04 `PricingConfig` notes) |
| Membership not awaiting payment | mock `CohortMembership` in a state not yet Admin-approved | call `initiatePayment(...)` | throws `ApiError(409, "This membership is not awaiting payment")` |
| Invalid/expired promotion code | mock `promotion.service.applyToPayment` to throw for the given code | call `initiatePayment(callerId, callerRole, id, "BADCODE")` | throws `ApiError(400, "Invalid or expired promotion code")` |
| Grade 6–12 student pays independently, no guardian required | mock caller is a Student in the 6–12 band with no `ParentStudentRelationship` on the membership | call `initiatePayment(studentId, 'STUDENT', cohortMembershipId)` | resolves successfully — no guardian-approval check blocks this path (FR-PB-008) |
| Never confirms the schedule itself | spy on `session.service.generateSessionsForCohort` | call `initiatePayment(...)` | assert it was never called — schedule confirmation only happens once the webhook reports `SUCCESS` (below) |

#### handleChapaWebhook / setBillingCycleAnchor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Invalid signature rejected | mock `chapa.client.verifyWebhookSignature` → `false` | call `handleChapaWebhook(rawBody, badSignature)` | throws `ApiError(400, "Invalid webhook signature")` |
| SUCCESS confirms the schedule and sets the anchor on first payment | mock a valid signature, `event: SUCCESS`, this is the membership's first successful `Payment` | call `handleChapaWebhook(...)` | sets `Payment.status: SUCCESS`; calls `session.service.generateSessionsForCohort`; sets `CohortMembership.billingCycleAnchorDate` to now |
| Anchor is never reset on a subsequent recurring payment | mock the membership already has a `billingCycleAnchorDate` set from a prior cycle | call `handleChapaWebhook(...)` for this cycle's payment | `billingCycleAnchorDate` remains unchanged — tested as a distinct branch from the first-payment case, not inferred from it |
| FAILED leaves the membership awaiting payment | mock a valid signature, `event: FAILED` | call `handleChapaWebhook(...)` | sets `Payment.status: FAILED`; no schedule generated; no automatic retry initiated — the student must re-`initiatePayment` |
| Idempotent against a duplicate webhook delivery | mock `Payment.status` already `SUCCESS` | call `handleChapaWebhook(...)` again with the same payload | no duplicate schedule generation, no duplicate anchor-set — the handler checks terminal status before reprocessing (guards against Chapa's own retry behavior; ties OWASP A08:2021 — replay of a previously-processed integrity-bearing event) |

#### getPaymentHistory

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns paginated history for the caller/target student | mock several `Payment` rows | call `getPaymentHistory(callerId, callerRole, studentId, 1, 20)` | resolves `PaginatedPaymentDTO` |
| No payments yet | mock an empty result | call `getPaymentHistory(...)` | resolves `{ payments: [], page, limit, total: 0 }`, not an error (UC-38) |
| Parent cannot view a non-relation student's history (IDOR) | mock no `ACTIVE` `ParentStudentRelationship` between caller and `studentId` | call `getPaymentHistory(parentId, 'PARENT', otherStudentId, 1, 20)` | throws `ApiError(403, ...)` |

---

### 9.5 Test Case Detail — payment.controller.test.ts / payment.routes.test.ts

**OWASP: A01:2021 – Broken Access Control, A02:2021 – Cryptographic Failures (webhook route's deliberate non-JWT auth path).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| initiate and getHistory require auth | no Authorization header | request `POST /payments/initiate`, `GET /payments/history` | both `401` |
| webhook route is unauthenticated by JWT but signature-checked | no Authorization header, valid Chapa signature header | request `POST /payments/webhook/chapa` | reaches the handler (no `401`) — auth is verified inside the handler via `chapa.client.verifyWebhookSignature`, not `authMiddleware`, per §0.5's deliberate exception |
| webhook route receives the raw body, not JSON-parsed | inspect what the handler receives | request `POST /payments/webhook/chapa` with a raw signed payload | `req.rawBody` (or equivalent) is the exact bytes Chapa sent — confirms this route is excluded from the app's global `express.json()` parsing, since `verifyWebhookSignature` needs the untouched raw buffer (a broken signature check here is a real security regression, not just a bug) |
| initiate forwards req.user.id/role, never a client-suppliable payer | mock service | call controller | `initiatePayment` called with `req.user.id`/`req.user.role`, not any body field |
| getHistory studentId resolution | mock service | call controller as Student vs. Parent with `?studentId=` | Student ignores any override; Parent's query value is forwarded for the service-layer relationship check |

---

### 9.6 Test Case Detail — paymentPause.service.test.ts

FRs: FR-PB-005, FR-PB-009. **OWASP: none client-facing (event-driven, no direct input) — however, the no-miss-no-refund guarantee is a direct anti-pattern check against silently mis-billing a paused student.**

#### pauseForNonPayment / resumeOnPayment

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Pauses on missed due date | mock a billing cycle past its due date with no successful payment | call `pauseForNonPayment(cohortMembershipId)` | creates `PaymentPause(reason: NONPAYMENT, startedAt)` |
| Resumes on next successful payment | mock an active pause | call `resumeOnPayment(cohortMembershipId)` | sets `endedAt` on the pause; calls `rescheduleSessionsDuringPause` |

#### rescheduleSessionsDuringPause

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sessions inside the pause window are rescheduled, not miss-flagged | mock 2 `ScheduledSession`s falling inside `[startedAt, endedAt]` | call `rescheduleSessionsDuringPause(cohortMembershipId)` | both sessions set to `status: PAYMENT_PAUSE_RESCHEDULED`; assert `sessionMiss.service.recordTutorCausedMiss`/`recordStudentCausedMiss` were never called for either — no fault classification is generated (FR-PB-009) |
| No refund and no earnings entry generated for a paused session | spy on `refund.service`/`earning.service` calls | call `rescheduleSessionsDuringPause(...)` | assert neither was invoked for the affected sessions — tested by spying on the downstream calls directly, not merely inferred from the return value |
| Sessions outside the window are untouched | mock a session scheduled before `startedAt` | call `rescheduleSessionsDuringPause(...)` | that session's status is unchanged |

---

### 9.7 Test Case Detail — paymentPause.controller.test.ts / paymentPause.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getPauseStatus requires auth | no Authorization header | request `GET /payment-pause/status` | `401` |
| Returns current pause + affected sessions for the caller's membership | mock service | call controller | resolves the pause status scoped to the caller |

---

### 9.8 Test Case Detail — pricing.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Rejects a mismatched share/total split | — | parse `{ body: { pricePerStudentPerHour: "100", totalPerHour: "100", platformSharePerHour: "40", tutorSharePerHour: "50" } }` | fails the `.refine` — 40 + 50 ≠ 100 |
| Accepts a reconciled split | — | parse with `platformSharePerHour + tutorSharePerHour === totalPerHour` | passes |
| format path param restricted to the 3 known formats | — | parse `{ params: { format: "ONE_TO_TEN" } }` | fails |

---

### 9.9 Test Case Detail — pricing.service.test.ts

FRs: FR-AD-009, Section 7 Definition of Done #2. **OWASP: A01:2021 – Broken Access Control (Admin-only mutate), A04:2021 – Insecure Design (versioned atomic activation prevents a race that could leave two active configs for the same format).**

#### getActiveConfig

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns one active row per format | mock 3 active configs, one per format, plus older inactive versions | call `getActiveConfig()` | resolves exactly 3 rows, all `isActive: true` |
| Partially-formed group still bills at the full per-student rate | mock a 1-to-5 config that closed with only 3 students enrolled | call `getActiveConfig()` (read path used by billing) | the returned `pricePerStudentPerHour` is unchanged/unadjusted — this function has no "adjusted rate" branch, and none should be introduced (Section 7 Definition of Done #2) |

#### createAndActivateConfig

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Mismatched split rejected at the service layer too | bypass the schema (simulate an internal call) with a non-reconciling split | call `createAndActivateConfig(format, input, adminId)` | throws `ApiError(400, "Platform and tutor shares must sum to the total per hour")` — redundant with the schema, retained as authoritative |
| Deactivates the old config and activates the new one atomically | mock an existing active config for the format | call `createAndActivateConfig(...)` | assert both operations occur inside a single `prisma.$transaction([...])` call — a partial write must never leave two active configs (or none) for the same format simultaneously |
| Change applies to the next new booking only | mock a `Payment`/`TutorEarning` already created under the previous config | call `createAndActivateConfig(...)` with a new rate, then re-fetch the prior `Payment` | the prior `Payment.amount` is unchanged — never retroactively altered (Section 7 Definition of Done #1) |

---

### 9.10 Test Case Detail — pricing.controller.test.ts / pricing.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getActive is public | no Authorization header | request `GET /pricing` | `200`, not `401` |
| adminUpdate requires Admin | no Authorization header, then a Tutor token | request `PUT /admin/pricing/:format` | `401` then `403` |
| adminUpdate validates body | valid Admin token | request with a non-reconciling split | rejected by `validate(updatePricingConfigSchema)`, controller never called |

---

### 9.11 Test Case Detail — refund.schema.test.ts — **I1 fix**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Rejects an empty rejectionReason | — | parse `{ params: { refundId: <uuid> }, body: { rejectionReason: "" } }` | fails — `rejectionReason` requires `.min(1)` |
| Rejects a non-UUID refundId | — | parse `{ params: { refundId: "not-a-uuid" }, body: { rejectionReason: "Student-caused disruption" } }` | fails |
| Accepts a valid payload | — | parse `{ params: { refundId: <uuid> }, body: { rejectionReason: "Student-caused disruption" } }` | passes |

---

### 9.12 Test Case Detail — refund.service.test.ts

FRs: FR-PB-007, FR-AD-012, FR-SP-048, Section 13. **OWASP: A01:2021 – Broken Access Control (Admin-only), A04:2021 – Insecure Design (proration correctness is a direct monetary-integrity concern).**

#### calculateProration

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sessions-delivered proration formula, worked example | mock `Cohort.sessionsPerWeek: 2` (`totalSessionsBilled: 8`), `sessionsRemaining: 3`, `payment.amount: "800.00"` | call `calculateProration(paymentId, reason)` | resolves `amount: "300.00"` — `(3/8) × 800`, rounded to 2 decimal places per the v3.2 monetary-rounding rule |
| Rounding is applied, never stored unrounded | mock a fractional result (e.g. `(1/3) × 100 = 33.333...`) | call `calculateProration(...)` | resolves `amount: "33.33"` — rounded to 2 decimals, never a raw unrounded `Decimal` |
| Proration is by sessions, never calendar days | mock a scenario where sessions-remaining and calendar-days-remaining would produce different answers | call `calculateProration(...)` | the resolved amount matches the sessions-based formula, not a calendar-day-based one |
| A free make-up session is never double-counted as undelivered | mock a `SessionMiss`-triggered free make-up session already delivered (FR-MK-001), alongside genuinely undelivered sessions | call `calculateProration(...)` | `sessionsRemaining` counts only genuinely undelivered *billed* sessions — the make-up session is not additionally subtracted as if it were a separate undelivered slot (Section 13 Definition of Done #4) |
| totalSessionsBilled is the current cycle's count, not a lifetime total | mock a cohort several billing cycles into its lifetime | call `calculateProration(...)` | `totalSessionsBilled` reflects only `sessionsPerWeek × 4` for the *current* 28-day cycle, not an accumulated multi-cycle count |

#### createPendingRefund — **I1 fix**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates a PENDING refund with the calculated amount already stored | mock `calculateProration` resolving `{ sessionsRemaining: 3, totalSessionsBilled: 8, amount: "300.00" }` | call `createPendingRefund(paymentId, 'TUTOR_DROPOUT')` | a `Refund` row is created with `status: PENDING`, `amount: "300.00"`, `approvedById: null`, `approvedAt: null` |
| Never recalculates on later approval | create a pending refund, then mock `calculateProration` to return a *different* amount, then call `approveRefund` | call `approveRefund(refundId, adminId)` | the persisted `amount` from creation time is unchanged; `calculateProration` is not invoked a second time |

#### approveRefund

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Approves a qualifying, PENDING refund | mock a `PENDING` `Refund` row meeting Section 3 policy conditions | call `approveRefund(refundId, adminId)` | resolves `{ id, status: 'APPROVED', amount, approvedById: adminId, approvedAt }`; sets `Refund.status: APPROVED` |
| Rejects (throws) a non-qualifying case | mock a `PENDING` row from a student-caused-disruption scenario (doesn't meet policy conditions) | call `approveRefund(refundId, adminId)` | throws `ApiError(409, "This case does not meet the refund policy conditions")` |
| Refund not found — **I1 fix** | mock no matching `Refund` row | call `approveRefund(refundId, adminId)` | throws `ApiError(404, "Refund not found")` |
| Cannot approve an already-actioned refund — **I1 fix** | mock a `Refund` row already at `status: APPROVED` | call `approveRefund(refundId, adminId)` | throws `ApiError(409, "This refund has already been actioned")` |
| Cannot approve a rejected refund — **I1 fix** | mock a `Refund` row already at `status: REJECTED` | call `approveRefund(refundId, adminId)` | throws `ApiError(409, "This refund has already been actioned")` |

#### rejectRefund — **I1 fix**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Rejects a PENDING refund | mock a `PENDING` `Refund` row | call `rejectRefund(refundId, adminId, "Student-caused disruption")` | resolves `{ id, status: 'REJECTED', rejectedById: adminId, rejectedAt, rejectionReason: "Student-caused disruption" }`; `approvedById`/`approvedAt` remain null |
| Refund not found | mock no matching `Refund` row | call `rejectRefund(refundId, adminId, "reason")` | throws `ApiError(404, "Refund not found")` |
| Cannot reject an already-actioned refund | mock a `Refund` row already at `status: APPROVED` or `REJECTED` | call `rejectRefund(refundId, adminId, "reason")` | throws `ApiError(409, "This refund has already been actioned")` |
| No money movement or side effects | mock a `PENDING` refund; spy on `earning.service`/Chapa-call sites | call `rejectRefund(...)` | no downstream money-movement call is made — only the `Refund` row's status/audit fields change |

---

### 9.13 Test Case Detail — refund.controller.test.ts / refund.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All three routes require Admin — **I1 fix** | no Authorization header, then a Parent token | request `GET /admin/refunds`, `POST /admin/refunds/:id/approve`, and `POST /admin/refunds/:id/reject` | `401` then `403` on each |
| adminApprove passes req.user.id as approver | mock service | call controller with a valid Admin token | `approveRefund` called with `req.user.id`, never a client-suppliable approver id |
| adminReject passes req.user.id as rejector and requires a body — **I1 fix** | mock service; valid Admin token | call controller with `{ rejectionReason: "..." }` | `rejectRefund` called with `req.user.id` and `req.body.rejectionReason`, never a client-suppliable rejector id |
| adminReject validates rejectionReason — **I1 fix** | valid Admin token | request `POST /admin/refunds/:id/reject` with an empty body | rejected by `validate(rejectRefundSchema)` with `400`, controller never called |
| adminReview reads persisted rows, not a live calculation — **I1 fix** | seed `Refund` rows at `PENDING` and `APPROVED` | request `GET /admin/refunds?status=PENDING` | only the `PENDING` rows are returned, with their already-stored `amount`; `calculateProration` is not invoked by this handler |

---

### 9.14 Test Case Detail — earning.service.test.ts

FRs: FR-MK-009, FR-AD-011, FR-TU-019. **OWASP: A04:2021 – Insecure Design (the reduced make-up rate is a monetary business rule with a hard-coded, easy-to-invert condition).**

#### creditEarning

| Case | Setup | Action | Expected result |
|---|---|---|---|
| FULL rate credits the tutor's normal share | mock `tutorSharePerHour: "175.00"` | call `creditEarning(sessionId, tutorId, 'FULL')` | resolves `TutorEarningDTO` with `amount: "175.00"` |
| REDUCED_MAKEUP rate credits exactly 50% | same `tutorSharePerHour: "175.00"` | call `creditEarning(sessionId, tutorId, 'REDUCED_MAKEUP')` | resolves `amount: "87.50"` |
| REDUCED_MAKEUP rounding on a fractional split | mock `tutorSharePerHour: "175.01"` (50% = 87.505) | call `creditEarning(sessionId, tutorId, 'REDUCED_MAKEUP')` | resolves `amount: "87.51"` or `"87.50"` per the project's defined rounding mode — asserted against the project's chosen half-up/half-even convention (Section 13's M7 rounding rule), never left as an unrounded 3-decimal value |
| REDUCED_MAKEUP never applies to a reschedule | mock a session delivered via `reschedule.service.ts` (not a tutor-caused-miss make-up) | call `creditEarning(sessionId, tutorId, 'FULL')` for that session | resolves the full rate — reschedules always pay full share, confirming the reduced rate is never accidentally applied outside the tutor-caused-miss path |
| REDUCED_MAKEUP never applies following a student-caused miss | mock a make-up-like session that actually followed a student-caused miss | call `creditEarning(...)` for it | resolves the `FULL` rate — the reduced rate applies only when the make-up followed the *tutor's own* miss (FR-MK-009) |
| References the ScheduledSession as the hard FK | inspect the created row | call `creditEarning(sessionId, tutorId, 'FULL')` | `TutorEarning.scheduledSessionId === sessionId` |

#### getEarningsForTutor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| upcomingPayout is always computed, never a stored draft | mock several unpaid `TutorEarning` rows for the current period, no `Payout` row yet exists | call `getEarningsForTutor(tutorId, 1, 20)` | resolves `upcomingPayout` derived live from those unpaid rows — there is no manual "request payout" action anywhere in this feature (Section 06 Definition of Done #3) |
| Includes reduced-rate sessions in the earnings list | mock a mix of `FULL` and `REDUCED_MAKEUP` earnings | call `getEarningsForTutor(...)` | resolves both, each showing its own `amount` |
| No earnings yet | mock an empty result | call `getEarningsForTutor(...)` | resolves `{ earnings: [], upcomingPayout: { amount: "0.00", ... }, page, limit, total: 0 }`, not an error |

---

### 9.15 Test Case Detail — earning.controller.test.ts / earning.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requires auth | no Authorization header | request `GET /tutors/me/earnings` | `401` |
| Always scoped to the caller | mock service | call controller | `getEarningsForTutor` called with `req.user.id`, never a client-suppliable `tutorId` — a Tutor can never view another tutor's earnings through this endpoint |

---

### 9.16 Test Case Detail — payout.service.test.ts

FRs: FR-TU-019, FR-AD-011. **OWASP: A01:2021 – Broken Access Control (Admin-only), A04:2021 – Insecure Design (no client-facing creation path is itself a deliberate control against a client forging a payout).**

#### generateMonthlyPayouts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Batches unpaid earnings per tutor for the period | mock 2 tutors with unpaid `TutorEarning` rows in the window, 1 tutor with none | call `generateMonthlyPayouts(periodStart, periodEnd)` | creates exactly 2 `Payout(status: PENDING)` rows, one per tutor with unpaid earnings |
| Marks batched earnings to prevent double-counting | mock the same earnings | call `generateMonthlyPayouts(...)` twice for the same period | the second run creates no additional payouts/duplicate batching — batched `TutorEarning` rows are excluded from a subsequent run |
| No manual client-facing creation path exists | — | (documentation-level check) confirm no route in `payout.routes.ts` maps to `generateMonthlyPayouts` | no such route exists — matching §0.4's explicit "no `POST /admin/payouts` to create one manually" note |

#### markPaid / adminAdjust

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Marks a pending payout paid | mock `Payout.status: PENDING` | call `markPaid(payoutId, adminId)` | resolves `{ id, status: 'PAID', paidAt }` |
| Rejects marking an already-paid payout | mock `Payout.status: PAID` | call `markPaid(payoutId, adminId)` again | throws `ApiError(409, "This payout has already been marked as paid")` |
| adminAdjust corrects a batch before it's paid | mock `Payout.status: PENDING`, a dispute-driven correction (UC-82) | call `adminAdjust(payoutId, adminId, input)` | resolves an updated `PayoutDTO` reflecting the correction |

---

### 9.17 Test Case Detail — payout.controller.test.ts / payout.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Both routes require Admin | no Authorization header, then a Tutor token | request `GET /admin/payouts` and `POST /admin/payouts/:id/mark-paid` | `401` then `403` — a Tutor must never mark their own (or anyone's) payout paid |
| adminMarkPaid passes req.user.id | mock service | call controller | `markPaid` called with `req.user.id`, never a client-suppliable admin id |

---

### 9.18 Test Case Detail — promotion.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requires validTo after validFrom | — | parse with `validTo` before `validFrom` | fails the `.refine` |
| Accepts a valid ascending range | — | parse with `validTo` after `validFrom` | passes |
| discountType restricted to PERCENT\|FIXED_ETB | — | parse `{ body: { ..., discountType: "COUPON" } }` | fails |

---

### 9.19 Test Case Detail — promotion.service.test.ts

FRs: FR-AD-016. **OWASP: A01:2021 – Broken Access Control (Admin-only create), A04:2021 – Insecure Design (expired/inactive-code rejection is the central anti-abuse control here).**

#### createPromotion / listActivePromotions / applyToPayment

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates a new promotion | — | call `createPromotion(input, adminId)` | resolves `PromotionCodeDTO` |
| Rejects a duplicate code | mock a `PromotionCode` already exists with this `code` | call `createPromotion(sameCodeInput, adminId)` | throws `ApiError(409, "A promotion with this code already exists")` |
| Rejects a non-ascending date range at the service layer too | bypass the schema, simulate an internal call | call `createPromotion(...)` | throws `ApiError(400, "End date must be after start date")` |
| listActivePromotions excludes expired/inactive codes | mock one active, one expired, one not-yet-valid code | call `listActivePromotions()` | resolves only the currently-active one |
| applyToPayment computes a PERCENT discount | mock `discountType: PERCENT`, `discountValue: "10"` | call `applyToPayment(code, "1000.00")` | resolves `discountedAmount: "900.00"` |
| applyToPayment computes a FIXED_ETB discount | mock `discountType: FIXED_ETB`, `discountValue: "50"` | call `applyToPayment(code, "1000.00")` | resolves `discountedAmount: "950.00"` |
| applyToPayment rejects an unknown/expired/inactive code | mock lookup fails/expired | call `applyToPayment("BADCODE", "1000.00")` | surfaces as `ApiError(400, "Invalid or expired promotion code")` from `payment.service.initiatePayment`'s perspective (9.4) |
| A FIXED_ETB discount never produces a negative amount | mock `discountValue` larger than `baseAmount` | call `applyToPayment(code, "10.00")` with a `discountValue: "50"` | resolves `discountedAmount: "0.00"`, not a negative value — flagged as a floor the implementer must enforce, since Doc 8-7 doesn't explicitly state this floor |

---

### 9.20 Test Case Detail — promotion.controller.test.ts / promotion.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listActive is public | no Authorization header | request `GET /promotions/active` | `200` |
| adminCreate/adminEdit require Admin | no Authorization header, then a Student token | request `POST /admin/promotions` and `PATCH /admin/promotions/:id` | `401` then `403` |
| adminCreate validates body | valid Admin token | request with a non-ascending date range | rejected by `validate(createPromotionSchema)` |

---

### 9.21 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `calculateProration`'s sessions-delivered formula is tested against the authoritative worked example from Doc 02 §13 (not just an arbitrarily invented set of numbers), and the make-up-session-not-double-counted case is tested with an actual delivered make-up session in the fixture, not merely asserted from the formula in the abstract.
- [ ] `handleChapaWebhook`'s idempotency is tested by replaying the *identical* payload against an already-`SUCCESS` `Payment`, asserting zero additional side effects (no second `generateSessionsForCohort` call, no anchor reset) — not just that the endpoint returns `200` twice.
- [ ] `verifyWebhookSignature`'s tamper-rejection case mutates the actual raw bytes being verified, not a re-serialized copy — a test that re-stringifies JSON before comparing could pass even with a byte-sensitive HMAC bug in the real implementation.
- [ ] `creditEarning`'s `REDUCED_MAKEUP` rate is tested as a genuinely separate branch from both the reschedule-pays-full case and the student-caused-miss-pays-full case — three distinct scenarios, not two collapsed into "make-up sessions pay less."
- [ ] `createAndActivateConfig`'s atomic deactivate-then-activate is tested by asserting a single `$transaction` call wraps both operations — not by checking the end state alone, which wouldn't catch a non-atomic two-step write that could race.
- [ ] `rescheduleSessionsDuringPause`'s no-miss-no-refund guarantee is tested by spying on `sessionMiss.service`/`refund.service`/`earning.service` calls directly and asserting they were never invoked — not inferred from the returned session status alone.
- [ ] No test in this file hardcodes a `Decimal` string result that wasn't actually derived from the formula under test — every monetary expected-value in 9.11/9.13/9.18 is computed from the same inputs the mock setup describes.

---

### 9.22 Out of Scope for Automated Testing (and why)

- **Real Chapa checkout/webhook network behavior** — `chapa.client.ts` is unit-tested against a mocked HTTP layer only; actual provider auth, webhook delivery reliability, and payload-shape drift need a manual or separately-tracked integration pass, same posture as `sms.client.ts`/`email.client.ts` in 9-1.
- **The actual money movement for an approved refund** — `approveRefund` marks `Refund.status: APPROVED` in this system's data model; the outbound Chapa-side refund call is explicitly flagged in Doc 8-7 as an implementation detail to confirm against Chapa's refund API at build time, not modeled as a separate function here.
- **`monthlyPayout.job.ts`/`paymentReminder.job.ts` interval scheduling** — no business logic of their own; the logic they call is already covered directly via `payout.service.test.ts` and the shared `notification.service.test.ts`.
- **Production `Decimal` precision/library configuration** (rounding-mode global settings, currency-locale formatting) — unit tests confirm the functions compute and round correctly per the documented rule; library-level configuration correctness is a setup/config-review concern.
- **Server-side rate limiting on `initiatePayment`/promotion-code application** — resolved (Doc 02 NFR-013): `rateLimiter.middleware.ts` (10/hour per account, covering both payment initiation and promo-code application since they share one endpoint) is applied at the route level and unit-tested in `9-1-shared-config.md`'s `rateLimiter.middleware.test.ts` section — not re-tested per-feature, since the middleware itself is feature-agnostic and its application here is a one-line route change (`8-7-payments-earnings.md`).

---

**Next:** proceed to → [9-8. Backend Test Documentation: Support, Trust & Admin Reporting]
