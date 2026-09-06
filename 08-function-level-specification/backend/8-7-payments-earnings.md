## Project: AKEWTutor — Backend Function-Level Spec: Payments & Earnings
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.5 (Chapa webhook auth exception) and §0.4 (payment-pause session reschedule and monthly payout batch are job/event-driven, exposed here only as read state). **API reference:** `07-payments-earnings-api.md`. **Folder/file reference:** `05a-backend-structure.md` §7.

**Owns:** PricingConfig, Payment, PaymentPause, Refund, TutorEarning, Payout, PromotionCode. **Depends on:** `matching-cohorts`, `class-delivery-library` (hard — `TutorEarning` → `ScheduledSession`).

---

### src/utils/providers/chapa.client.ts (new)

Thin wrapper around Chapa payment initiation and webhook signature verification (§0.5). Shared contract:

```typescript
interface ChapaClient {
  initiateCheckout(amount: string, reference: string, callbackUrl: string): Promise<{ checkoutUrl: string }>;
  verifyWebhookSignature(rawBody: Buffer, signatureHeader: string): boolean;
}
```

Test file mocks the underlying Chapa SDK/HTTP call.

### src/schemas/payment.schema.ts (new)

| Schema | Shape |
|---|---|
| initiatePaymentSchema | `z.object({ body: z.object({ cohortMembershipId: z.string().uuid(), promotionCode: z.string().optional() }) })` |
| applyPromotionSchema | `z.object({ body: z.object({ promotionCode: z.string().min(1), amount: z.string() }) })` — internal helper shape used inside `initiatePayment`, not a standalone endpoint. |

### src/services/payment.service.ts (new)

#### initiatePayment

| Field | Detail |
|---|---|
| Signature | `initiatePayment(callerId, callerRole, cohortMembershipId: string, promotionCode?: string): Promise<{ paymentId, chapaCheckoutUrl, amount, status: 'PENDING' }>` |
| Purpose | Begins a Chapa checkout for an Admin-approved booking/auto-match, including the independent Grade 6–12 path with no guardian required (FR-PB-008). |
| Throws | `ApiError(409, "This membership is not awaiting payment")` — `cohortMembershipId` not in a state awaiting payment (e.g. not yet Admin-approved). `ApiError(400, "Invalid or expired promotion code")` — `promotionCode` invalid/expired/inactive. |
| Side effects | Reads the `PricingConfig` active **at initiation time** — `amount` is locked in now and never retroactively affected by a later Admin price change (Doc 04 PricingConfig notes). Applies the promotion discount via `promotion.service.ts → applyToPayment`. Calls `chapa.client.ts → initiateCheckout`. Creates `Payment(status: PENDING)`. |
| Edge cases | The schedule is confirmed only once the webhook below reports `SUCCESS` — this function never confirms the schedule itself, only starts the checkout flow. |

Test file: `tests/services/payment.service.test.ts`

#### handleChapaWebhook / setBillingCycleAnchor

| Field | Detail |
|---|---|
| Signature | `handleChapaWebhook(rawBody: Buffer, signatureHeader: string): Promise<{ received: true }>` · `setBillingCycleAnchor(cohortMembershipId: string): Promise<void>` |
| Purpose | Confirms the schedule (all formats) once Chapa reports `SUCCESS`; `setBillingCycleAnchor` sets `CohortMembership.billingCycleAnchorDate` **only on the first successful payment** for that membership (FR-PB-003) — never on a subsequent recurring payment, which would incorrectly reset the anchor. |
| Throws | `ApiError(400, "Invalid webhook signature")` — `chapa.client.ts → verifyWebhookSignature` fails. |
| Side effects | Sets `Payment.status: SUCCESS`, generates the confirmed `ScheduledSession` rows via `class-delivery-library`'s `session.service.ts → generateSessionsForCohort` (first payment only), moves `Cohort.status` to confirmed. On `FAILED`, sets `Payment.status: FAILED` and leaves the membership in its prior awaiting-payment state — no automatic retry from this function; the student re-initiates via `initiatePayment`. |
| Edge cases | This handler must be idempotent against Chapa retrying the same webhook delivery — checked via `Payment.status` already being terminal (`SUCCESS`/`FAILED`) before reprocessing. |

Test file: `tests/services/payment.service.test.ts` — includes the signature-failure case and the first-payment-only anchor-set case.

#### getPaymentHistory

| Field | Detail |
|---|---|
| Signature | `getPaymentHistory(callerId, callerRole, studentId, page, limit): Promise<PaginatedPaymentDTO>` |
| Edge cases | No payments yet → `payments: []`, not an error (UC-38 alternate flow). |

Test file: `tests/services/payment.service.test.ts`

### src/controllers/payment.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| initiate | `paymentService.initiatePayment(req.user.id, req.user.role, req.body.cohortMembershipId, req.body.promotionCode)` | 200 |
| webhook | `paymentService.handleChapaWebhook(req.rawBody, req.headers['chapa-signature'])` | 200 |
| getHistory | `paymentService.getPaymentHistory(req.user.id, req.user.role, req.query.studentId, req.query.page, req.query.limit)` | 200 |

### src/routes/payment.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | /initiate | `authMiddleware, validate(initiatePaymentSchema)` | initiate |
| POST | /webhook/chapa | — (Webhook auth: signature-verified inside the handler, not `authMiddleware`; this route needs the raw request body, so it must be excluded from the global `express.json()` body-parsing or use a raw-body capture, unlike every other route in the app) | webhook |
| GET | /history | `authMiddleware` | getHistory |

Mounted at `/payments`. The webhook route is the one deliberate deviation from the app's otherwise-uniform JSON body-parsing middleware order — worth flagging explicitly in `src/app.ts` as a comment, since it's easy to accidentally "clean up" during implementation and break signature verification.

---

### src/services/paymentPause.service.ts (new)

#### pauseForNonPayment / resumeOnPayment

| Field | Detail |
|---|---|
| Signature | `pauseForNonPayment(cohortMembershipId: string): Promise<PaymentPauseDTO>` · `resumeOnPayment(cohortMembershipId: string): Promise<void>` |
| Purpose | Event-driven — `pauseForNonPayment` triggers when a billing cycle's payment isn't received by its due date; `resumeOnPayment` triggers on the next successful `Payment` for the membership. Neither is a client-facing endpoint. |
| Side effects | (pause) Creates `PaymentPause(reason: NONPAYMENT, startedAt)`. (resume) Sets `endedAt` on the pause, then calls `rescheduleSessionsDuringPause`. |

#### rescheduleSessionsDuringPause

| Field | Detail |
|---|---|
| Signature | `rescheduleSessionsDuringPause(cohortMembershipId: string): Promise<{ rescheduled: string[] }>` |
| Purpose | Any `ScheduledSession` that fell inside the pause window is rescheduled — never marked as a `SessionMiss`, no fault classification generated (FR-PB-009). This is triggered on payment resume, not a standalone cron (§0.4). |
| Side effects | Sets affected sessions to `status: PAYMENT_PAUSE_RESCHEDULED`, calling into `class-delivery-library`'s `session.service.ts`. |

Test file: `tests/services/paymentPause.service.test.ts` — includes the no-miss-no-refund-during-pause case explicitly.

### src/controllers/paymentPause.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getPauseStatus | direct read of `PaymentPause` + affected sessions for `cohortMembershipId` | 200 |

### src/routes/paymentPause.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /status | `authMiddleware` | getPauseStatus |

Mounted at `/payment-pause`.

---

### src/schemas/pricing.schema.ts (new)

| Schema | Shape |
|---|---|
| updatePricingConfigSchema | `z.object({ params: z.object({ format: z.enum(['ONE_TO_ONE','ONE_TO_THREE','ONE_TO_FIVE']) }), body: z.object({ pricePerStudentPerHour: z.string(), totalPerHour: z.string(), platformSharePerHour: z.string(), tutorSharePerHour: z.string() }).refine(b => new Decimal(b.platformSharePerHour).plus(b.tutorSharePerHour).equals(new Decimal(b.totalPerHour)), "Platform and tutor shares must sum to the total per hour") })` |

### src/services/pricing.service.ts (new)

#### getActiveConfig

| Field | Detail |
|---|---|
| Signature | `getActiveConfig(): Promise<PricingConfigDTO[]>` |
| Output | One row per format, `isActive: true` only. |
| Edge cases | A partially-formed 1-to-5 class that closes at 3 students is still billed at this same `pricePerStudentPerHour` — the shortfall is absorbed by platform/tutor shares; this function has no "adjusted rate" branch and must never introduce one (Section 7 Definition of Done #2). |

#### createAndActivateConfig

| Field | Detail |
|---|---|
| Signature | `createAndActivateConfig(format: Format, input, adminId: string): Promise<PricingConfigDTO>` |
| Throws | `ApiError(400, "Platform and tutor shares must sum to the total per hour")` — redundant with the schema refine, retained as the authoritative business rule. |
| Side effects | Versioned: `prisma.$transaction([deactivate old active config for format, create new config with isActive: true])` — atomic, since a partial write would leave two active configs (or none) for the same format simultaneously. |
| Edge cases | Reflected on the **next new booking only** — a `Payment`/`TutorEarning` already created under the previous config is never retroactively altered (Section 7 Definition of Done #1). |

Test file: `tests/services/pricing.service.test.ts` — includes the versioning/next-booking-only case and the atomic deactivate-then-activate transaction.

### src/controllers/pricing.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getActive | `pricingService.getActiveConfig()` | 200 |
| adminUpdate | `pricingService.createAndActivateConfig(req.params.format, req.body, req.user.id)` | 201 |

### src/routes/pricing.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /pricing | — | getActive |
| PUT | /admin/pricing/:format | `authMiddleware, requireRole('ADMIN'), validate(updatePricingConfigSchema)` | adminUpdate |

Same one-router-two-mount-points pattern as `policy.routes.ts`.

---

### src/services/refund.service.ts (new)

#### calculateProration

| Field | Detail |
|---|---|
| Signature | `calculateProration(paymentId: string, reason: RefundReason): Promise<RefundCalculationDTO>` |
| Purpose | Always by sessions delivered, never calendar days (Section 13). `amount = round((sessionsRemaining / totalSessionsBilled) × payment.amount, 2)`, where `totalSessionsBilled = payment.cohortMembership.cohort.sessionsPerWeek × 4` (Doc 02 §7 v3.2 callout, Doc 04 `Cohort.sessionsPerWeek`) — the current 28-day cycle's billed count, not a running lifetime total. Rounded to 2 decimal places per Section 13's v3.2 monetary-rounding rule (M7 fix) — never stored as an unrounded `Decimal`. |
| Edge cases | A free make-up session under FR-MK-001 is never counted as undelivered toward `sessionsRemaining` — it's a same-cost substitute for a session already billed, not an additional undelivered session (Section 13 Definition of Done #4). This is enforced by counting *distinct billed sessions*, not raw `ScheduledSession` rows, when computing `sessionsRemaining`. |

Test file: `tests/services/refund.service.test.ts` — includes the sessions-delivered proration formula case explicitly, and the make-up-session-not-double-counted case.

#### approveRefund

| Field | Detail |
|---|---|
| Signature | `approveRefund(refundId: string, adminId: string): Promise<{ id, amount, approvedById, approvedAt }>` |
| Throws | `ApiError(409, "This case does not meet the refund policy conditions")` — the refund case doesn't meet the Section 03 policy conditions (e.g. student-caused disruption). |
| Side effects | Marks `Refund.status: APPROVED`; the actual money movement back to the payer is a Chapa-side concern outside this function's scope per Doc 04/06 (not modeled as a separate outbound API call in the source docs — flagged as an implementation detail to confirm against Chapa's refund API at build time). |

Test file: `tests/services/refund.service.test.ts`

### src/controllers/refund.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| adminReview | `refundService.calculateProration`-backed paginated queue read, filtered by `status` | 200 |
| adminApprove | `refundService.approveRefund(req.params.refundId, req.user.id)` | 200 |

### src/routes/refund.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware, requireRole('ADMIN')` | adminReview |
| POST | /:refundId/approve | `authMiddleware, requireRole('ADMIN')` | adminApprove |

Mounted at `/admin/refunds`.

---

### src/services/earning.service.ts (new)

#### creditEarning

| Field | Detail |
|---|---|
| Signature | `creditEarning(sessionId: string, tutorId: string, rateType: 'FULL' \| 'REDUCED_MAKEUP'): Promise<TutorEarningDTO>` |
| Purpose | Event-triggered on session completion (`FULL`) or make-up-session completion (`REDUCED_MAKEUP`, flagged by `sessionMiss.service.ts → recordTutorCausedMiss`) — not directly client-facing. |
| Side effects | Inserts a `TutorEarning` row referencing the `ScheduledSession` (the hard FK named in this feature's dependency on `class-delivery-library`, per Feature Decomposition §1.1). `amount = tutorSharePerHour` for `rateType: FULL`; `amount = round(tutorSharePerHour × 0.5, 2)` for `rateType: REDUCED_MAKEUP` (FR-MK-009) — rounded to 2 decimal places per Section 13's v3.2 monetary-rounding rule (M7 fix) if the 50% split lands on a fractional subunit. |

#### getEarningsForTutor

| Field | Detail |
|---|---|
| Signature | `getEarningsForTutor(tutorId: string, page, limit): Promise<{ earnings: TutorEarningDTO[]; upcomingPayout: UpcomingPayoutDTO; page; limit; total }>` |
| Edge cases | `upcomingPayout` is always computed and visible ahead of time — there is no manual "request payout" action anywhere in this feature (Section 06 Definition of Done #3); it is derived from unpaid `TutorEarning` rows for the current period, not a stored draft `Payout` row. |

Test file: `tests/services/earning.service.test.ts`

### src/controllers/earning.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getMyEarnings | `earningService.getEarningsForTutor(req.user.id, req.query.page, req.query.limit)` | 200 |

### src/routes/earning.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware` | getMyEarnings |

Mounted at `/tutors/me/earnings`.

---

### src/services/payout.service.ts (new)

#### generateMonthlyPayouts

| Field | Detail |
|---|---|
| Signature | `generateMonthlyPayouts(periodStart: string, periodEnd: string): Promise<{ created: number }>` |
| Purpose | Batches unpaid `TutorEarning` rows into one `Payout(status: PENDING)` per tutor for the period — called by `monthlyPayout.job.ts`, never a client-triggerable action (§0.4: "there is no `POST /admin/payouts` to create one manually"). |
| Side effects | Bulk insert of `Payout` rows, one per tutor with unpaid earnings in the window; marks the underlying `TutorEarning` rows as batched (not yet paid) to prevent double-counting in a subsequent run. |

Test file: `tests/services/payout.service.test.ts` — includes the no-manual-request-step case (asserting no client-facing creation path exists).

#### markPaid / adminAdjust

| Field | Detail |
|---|---|
| Signature | `markPaid(payoutId: string, adminId: string): Promise<{ id, status: 'PAID', paidAt }>` · `adminAdjust(payoutId: string, adminId: string, input): Promise<PayoutDTO>` |
| Throws | (markPaid) `ApiError(409, "This payout has already been marked as paid")` — already `PAID`. |
| Purpose | `adminAdjust` handles the case where a payout batch needs correction after a dispute arose before being marked paid (UC-82). |

Test file: `tests/services/payout.service.test.ts`

### src/controllers/payout.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| adminList | `payoutService` read, filtered by `tutorId`/`status` | 200 |
| adminMarkPaid | `payoutService.markPaid(req.params.payoutId, req.user.id)` | 200 |

### src/routes/payout.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware, requireRole('ADMIN')` | adminList |
| POST | /:payoutId/mark-paid | `authMiddleware, requireRole('ADMIN')` | adminMarkPaid |

Mounted at `/admin/payouts`.

---

### src/schemas/promotion.schema.ts (new)

| Schema | Shape |
|---|---|
| createPromotionSchema | `z.object({ body: z.object({ code: z.string().min(1), discountType: z.enum(['PERCENT','FIXED_ETB']), discountValue: z.string(), validFrom: z.string().datetime(), validTo: z.string().datetime() }).refine(b => new Date(b.validTo) > new Date(b.validFrom), "End date must be after start date") })` |

### src/services/promotion.service.ts (new)

#### createPromotion / listActivePromotions / applyToPayment

| Field | Detail |
|---|---|
| Signature | `createPromotion(input, adminId: string): Promise<PromotionCodeDTO>` · `listActivePromotions(): Promise<PromotionCodeDTO[]>` · `applyToPayment(code: string, baseAmount: string): Promise<{ discountedAmount: string }>` |
| Throws | (create) `ApiError(409, "A promotion with this code already exists")` — unique constraint. `ApiError(400, "End date must be after start date")`. (applyToPayment, called from `payment.service.ts → initiatePayment`) surfaces as `ApiError(400, "Invalid or expired promotion code")` from the caller's perspective when the code is inactive/expired/unknown. |
| Side effects | `applyToPayment` computes `PERCENT` or `FIXED_ETB` discount against `baseAmount`, returning the post-discount total for `initiatePayment` to lock in. |

Test file: `tests/services/promotion.service.test.ts`

### src/controllers/promotion.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listActive | `promotionService.listActivePromotions()` | 200 |
| adminCreate | `promotionService.createPromotion(req.body, req.user.id)` | 201 |
| adminEdit | `promotionService.updatePromotion` (co-located, backs `PATCH /admin/promotions/:id`) | 200 |

### src/routes/promotion.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /promotions/active | — | listActive |
| POST | /admin/promotions | `authMiddleware, requireRole('ADMIN'), validate(createPromotionSchema)` | adminCreate |
| PATCH | /admin/promotions/:id | `authMiddleware, requireRole('ADMIN')` | adminEdit |

Same one-router-two-mount-points pattern.

---

### src/jobs/paymentReminder.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan for upcoming billing due dates, anchored per-student off `CohortMembership.billingCycleAnchorDate`. |
| Effect | Sends the 3-day-before reminder via `notification.service.ts`. |

### src/jobs/monthlyPayout.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Fixed monthly cycle. |
| Effect | Runs `payout.service.ts → generateMonthlyPayouts` for the closed period. |
| Idempotency | Should guard against double-running for the same period (e.g. check no `Payout` rows already exist for that `periodStart`/`periodEnd` before batching). |

---

**Next:** proceed to → [8-8. Backend: Support, Trust & Admin]
