## Project: AKEWTutor — API Specification
**Feature:** Payments & Earnings
**Conventions:** see 0.1–0.7 in `00-api-conventions.md`. See also 0.5 for the Chapa webhook auth exception, and 0.4 — the payment-pause session reschedule and the monthly payout batch are job/event-driven, exposed here only as read state.

**Owns:** PricingConfig, Payment, PaymentPause, Refund, TutorEarning, Payout, PromotionCode. **Depends on:** Matching & Cohorts, Class Delivery & Library (hard, per Doc 07 §1.1 — `TutorEarning` → `ScheduledSession`).

---

### 7.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| POST | /payments/initiate | Student\|Parent | UC-36, UC-37 | FR-SP-031, FR-PB-001, FR-PB-008 |
| POST | /payments/webhook/chapa | Webhook | UC-36 | FR-PB-001 |
| GET | /payments/history | Student\|Parent | UC-38 | FR-SP-032, FR-PB-002 |
| GET | /payment-pause/status | Student\|Parent | UC-40, UC-41 | FR-PB-005, FR-PB-009 |
| GET | /pricing | Public | UC-80 | FR-PR-001, FR-PR-002, FR-PR-003 |
| PUT | /admin/pricing/:format | Admin | UC-80 | FR-PR-004, FR-AD-009 |
| GET | /admin/refunds | Admin | UC-83 | FR-AD-012 |
| POST | /admin/refunds/:refundId/approve | Admin | UC-83 | FR-AD-012, FR-PB-007 |
| GET | /tutors/me/earnings | Tutor | UC-70 | FR-TU-018, FR-TU-019 |
| GET | /admin/payouts | Admin | UC-82 | FR-AD-011 |
| POST | /admin/payouts/:payoutId/mark-paid | Admin | UC-82 | FR-AD-011 |
| GET | /promotions/active | Public | UC-86 | FR-AD-016 |
| POST | /admin/promotions | Admin | UC-86 | FR-AD-016 |
| PATCH | /admin/promotions/:id | Admin | UC-86 | FR-AD-016 |

---

### 7.2 Endpoint Detail

#### POST /payments/initiate

**Purpose:** Begin a Chapa payment for an Admin-approved booking/auto-match (UC-36), including the independent Grades 6–12 path with no guardian required (UC-37, FR-PB-008).

**Auth:** Student|Parent

**Request body:**
```json
{
  "cohortMembershipId": "uuid, required",
  "promotionCode": "string, optional"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "paymentId": "uuid",
    "chapaCheckoutUrl": "https://checkout.chapa.co/...",
    "amount": "350.00",
    "status": "PENDING"
  }
}
```
`amount` reflects the `PricingConfig` active at initiation time, minus any valid `promotionCode` discount — never retroactively affected by a later Admin price change (Doc 04 PricingConfig notes). The schedule is confirmed only once the webhook below reports `SUCCESS`.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | `cohortMembershipId` is not in a state awaiting payment (e.g. not yet Admin-approved) | "This membership is not awaiting payment" |
| 400 | `promotionCode` invalid, expired, or inactive | "Invalid or expired promotion code" |

**Implemented in:** `src/controllers/payment.controller.ts → initiate` · `src/services/payment.service.ts → initiatePayment` · `src/schemas/payment.schema.ts → initiatePaymentSchema`

---

#### POST /payments/webhook/chapa

**Purpose:** Chapa's callback confirming payment success/failure. Confirms the schedule (all formats) once `SUCCESS` is received; sets `CohortMembership.billingCycleAnchorDate` on the first successful payment for that membership (Doc 04 Payment notes, FR-PB-003).

**Auth:** Webhook — verified via Chapa's signature header (see 0.5), not a JWT. Never called directly by a client application.

**Request body:** Chapa's own webhook payload shape (provider-defined; not reproduced here — see Chapa's integration docs at implementation time).

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "received": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Signature verification fails | "Invalid webhook signature" |

**Implemented in:** `src/controllers/payment.controller.ts → webhook` · `src/services/payment.service.ts → handleChapaWebhook, setBillingCycleAnchor`

---

#### GET /payments/history

**Purpose:** View all past payments (UC-38, FR-SP-032, FR-PB-002).

**Auth:** Student|Parent

**Query params:**
```
?studentId=uuid (required for Parent)
&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "payments": [
      {
        "id": "uuid",
        "amount": "350.00",
        "status": "SUCCESS",
        "billingPeriodStart": "2026-09-01T00:00:00Z",
        "billingPeriodEnd": "2026-09-30T23:59:59Z",
        "createdAt": "2026-09-01T09:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
No payments yet returns `payments: []` — not an error (UC-38 alternate flow).

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/payment.controller.ts → getHistory` · `src/services/payment.service.ts → getPaymentHistory`

---

#### GET /payment-pause/status

**Purpose:** Check whether a membership is currently paused for non-payment, and how any session falling inside the pause is being handled (UC-40, UC-41, FR-PB-005/009).

**Auth:** Student|Parent

**Query params:**
```
?cohortMembershipId=uuid, required
```

**Success response — 200 (active pause):**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "isPaused": true,
    "startedAt": "2026-09-04T00:00:00Z",
    "reason": "NONPAYMENT",
    "affectedSessions": [
      { "sessionId": "uuid", "originalStart": "2026-09-05T16:00:00Z", "status": "PAYMENT_PAUSE_RESCHEDULED" }
    ]
  }
}
```
Any session inside the pause window is never marked as a `SessionMiss` — no fault classification is generated for it, per FR-PB-009 (this endpoint reports state only; the actual reschedule happens automatically on payment resume, per 0.4).

**Success response — 200 (no active pause):**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "isPaused": false
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/paymentPause.controller.ts → getPauseStatus` · `src/services/paymentPause.service.ts → pauseForNonPayment, resumeOnPayment, rescheduleSessionsDuringPause`

---

#### GET /pricing

**Purpose:** Public read of the current price/hr and split for each format (UC-80, FR-PR-001–003).

**Auth:** Public

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "pricing": [
      {
        "format": "ONE_TO_ONE",
        "pricePerStudentPerHour": "350.00",
        "totalPerHour": "350.00",
        "platformSharePerHour": "100.00",
        "tutorSharePerHour": "250.00"
      },
      {
        "format": "ONE_TO_THREE",
        "pricePerStudentPerHour": "150.00",
        "totalPerHour": "450.00",
        "platformSharePerHour": "150.00",
        "tutorSharePerHour": "300.00"
      },
      {
        "format": "ONE_TO_FIVE",
        "pricePerStudentPerHour": "100.00",
        "totalPerHour": "500.00",
        "platformSharePerHour": "175.00",
        "tutorSharePerHour": "325.00"
      }
    ]
  }
}
```
A partially-formed 1-to-5 class that closes at 3 students (Section 7 Partial Group Formation) is still billed at this same `pricePerStudentPerHour` — the shortfall is absorbed by platform/tutor shares, never reflected as a different rate here (Section 7 Definition of Done #2).

**Error responses:** none.

**Implemented in:** `src/controllers/pricing.controller.ts → getActive` · `src/services/pricing.service.ts → getActiveConfig`

---

#### PUT /admin/pricing/:format

**Purpose:** Update pricing/revenue split for one format, without a code deployment (UC-80, FR-PR-004, FR-AD-009). Versioned — creates a new row and deactivates the old one, per Doc 04's `PricingConfig` design.

**Auth:** Admin

**Path params:** `format` — `ONE_TO_ONE | ONE_TO_THREE | ONE_TO_FIVE`

**Request body:**
```json
{
  "pricePerStudentPerHour": "decimal string, required",
  "totalPerHour": "decimal string, required",
  "platformSharePerHour": "decimal string, required",
  "tutorSharePerHour": "decimal string, required"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "format": "ONE_TO_ONE",
    "pricePerStudentPerHour": "375.00",
    "isActive": true,
    "createdById": "uuid"
  }
}
```
Reflected on the **next new booking only** — a `Payment`/`TutorEarning` already created under the previous config is never retroactively altered (Section 7 Definition of Done #1, Doc 04 PricingConfig notes).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Split doesn't reconcile (`platformSharePerHour + tutorSharePerHour ≠ totalPerHour`) | "Platform and tutor shares must sum to the total per hour" |

**Implemented in:** `src/controllers/pricing.controller.ts → adminUpdate` · `src/services/pricing.service.ts → createAndActivateConfig` · `src/schemas/pricing.schema.ts → updatePricingConfigSchema`

---

#### GET /admin/refunds

**Purpose:** Admin's refund-review queue (UC-83, FR-AD-012).

**Auth:** Admin

**Query params:**
```
?status=PENDING&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "refunds": [
      {
        "id": "uuid",
        "paymentId": "uuid",
        "reason": "TUTOR_DROPOUT",
        "sessionsRemaining": 2,
        "totalSessionsBilled": 8,
        "amount": "87.50"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/refund.controller.ts → adminReview` · `src/services/refund.service.ts → calculateProration`

---

#### POST /admin/refunds/:refundId/approve

**Purpose:** Approve a calculated, prorated refund (UC-83, FR-AD-012, FR-PB-007). Proration is always by sessions delivered, never calendar days (Section 13 Refund Proration Formula) — `amount = (sessionsRemaining / totalSessionsBilled) × payment.amount`. A free make-up session under FR-MK-001 is never counted as undelivered toward `sessionsRemaining` (Section 13 Definition of Done #3).

**Auth:** Admin

**Path params:** `refundId` — Refund UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "amount": "87.50",
    "approvedById": "uuid",
    "approvedAt": "2026-09-06T15:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Refund case doesn't meet the Section 03 policy conditions (e.g. student-caused disruption) | "This case does not meet the refund policy conditions" |

**Implemented in:** `src/controllers/refund.controller.ts → adminApprove` · `src/services/refund.service.ts → approveRefund`

---

#### GET /tutors/me/earnings

**Purpose:** Tutor views earnings, payout date/amount, and full-vs-reduced-rate itemization (UC-70, FR-TU-018/019).

**Auth:** Tutor

**Query params:**
```
?page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "earnings": [
      { "sessionId": "uuid", "amount": "250.00", "rateType": "FULL", "createdAt": "2026-09-05T17:00:00Z" },
      { "sessionId": "uuid", "amount": "125.00", "rateType": "REDUCED_MAKEUP", "createdAt": "2026-09-06T17:00:00Z" }
    ],
    "upcomingPayout": {
      "periodStart": "2026-09-01T00:00:00Z",
      "periodEnd": "2026-09-30T23:59:59Z",
      "estimatedTotal": "3750.00",
      "expectedDate": "2026-10-01T00:00:00Z"
    },
    "page": 1,
    "limit": 20,
    "total": 2
  }
}
```
No manual "request payout" action exists anywhere — `upcomingPayout` is always visible ahead of time and generated automatically (Section 06 Definition of Done #3).

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/earning.controller.ts → getMyEarnings` · `src/services/earning.service.ts → getEarningsForTutor`

---

#### GET /admin/payouts

**Purpose:** Admin oversight of monthly payout batches (UC-82, FR-AD-011).

**Auth:** Admin

**Query params:**
```
?tutorId=uuid&status=PENDING|PAID&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "payouts": [
      {
        "id": "uuid",
        "tutorId": "uuid",
        "periodStart": "2026-08-01T00:00:00Z",
        "periodEnd": "2026-08-31T23:59:59Z",
        "totalAmount": "3600.00",
        "status": "PENDING"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
`Payout` rows are created automatically by `monthlyPayout.job.ts` (0.4) — there is no `POST /admin/payouts` to create one manually.

**Error responses:** none.

**Implemented in:** `src/controllers/payout.controller.ts → adminList` · `src/services/payout.service.ts → generateMonthlyPayouts` (batch creation, job-triggered)

---

#### POST /admin/payouts/:payoutId/mark-paid

**Purpose:** Mark a payout batch as paid, or adjust it if a dispute arose first (UC-82).

**Auth:** Admin

**Path params:** `payoutId` — Payout UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "status": "PAID",
    "paidAt": "2026-10-01T09:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Payout already `PAID` | "This payout has already been marked as paid" |

**Implemented in:** `src/controllers/payout.controller.ts → adminMarkPaid` · `src/services/payout.service.ts → markPaid, adminAdjust`

---

#### GET /promotions/active

**Purpose:** Public read of currently active promotional codes/offers (UC-86).

**Auth:** Public

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "promotions": [
      {
        "code": "BACKTOSCHOOL2026",
        "discountType": "PERCENT",
        "discountValue": "10.00",
        "validTo": "2026-09-30T23:59:59Z"
      }
    ]
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/promotion.controller.ts → listActive` · `src/services/promotion.service.ts → listActivePromotions`

---

#### POST /admin/promotions

**Purpose:** Create a promotional discount (UC-86, FR-AD-016).

**Auth:** Admin

**Request body:**
```json
{
  "code": "string, required, unique",
  "discountType": "string, required — PERCENT | FIXED_ETB",
  "discountValue": "decimal string, required",
  "validFrom": "ISO 8601 datetime, required",
  "validTo": "ISO 8601 datetime, required"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "code": "BACKTOSCHOOL2026",
    "isActive": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | `code` already exists | "A promotion with this code already exists" |
| 400 | `validTo` not after `validFrom` | "End date must be after start date" |

**Implemented in:** `src/controllers/promotion.controller.ts → adminCreate` · `src/services/promotion.service.ts → createPromotion` · `src/schemas/promotion.schema.ts → createPromotionSchema`

---

#### PATCH /admin/promotions/:id

**Purpose:** Edit or deactivate a promotional code (UC-86).

**Auth:** Admin

**Path params:** `id` — PromotionCode UUID

**Request body:**
```json
{
  "isActive": "boolean, optional",
  "validTo": "ISO 8601 datetime, optional"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "isActive": false
  }
}
```

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/promotion.controller.ts` (co-located) · `src/services/promotion.service.ts`

---

**Next:** proceed to → [08. Support, Trust & Admin Reporting API]
