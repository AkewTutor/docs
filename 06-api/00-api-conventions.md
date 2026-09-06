## Project: AKEWTutor — API Specification

**Links back to:** [04. Database & Data Model], [05a. Backend Folder & File Structure], [05b. Frontend Folder & File Structure], [Feature Decomposition]
**Links forward to:** [01. Shared Config API]

Split into feature-based files, mirroring the 8-feature grouping used throughout Docs 04, 05a, 05b, and Feature Decomposition:

- **00-api-conventions.md** — this file: shared rules referenced by every feature file below
- **01-shared-config-api.md** — Shared Config (Auth, Notifications, Policies, Announcements)
- **02-accounts-guardianship-api.md** — Accounts & Guardianship (Students, Parents, Tutors, Subjects, Availability)
- **03-matching-cohorts-api.md** — Matching & Cohorts
- **04-class-delivery-library-api.md** — Class Delivery, Recording & Library
- **05-messaging-api.md** — In-Platform Messaging
- **06-gamification-engagement-api.md** — Gamification & Engagement
- **07-payments-earnings-api.md** — Payments & Earnings
- **08-support-trust-admin-api.md** — Support, Trust & Admin Reporting

Each feature file has its own endpoint table + endpoint detail sections. This file is not repeated in each — feature files reference "see 0.1" etc.

**No subfolders.** All 8 feature files (plus this one) are flat inside `06-api/`, matching the feature-file convention already established in Doc 05a/05b — a feature owning more entities (e.g. `matching-cohorts`, `class-delivery-library`) is still one file, organized into more endpoint-table rows and detail sections, not a nested folder. If any single feature file later grows unwieldy during implementation, split it by sub-area at that point (e.g. `03a-matching-cohorts-student-api.md` / `03b-matching-cohorts-admin-api.md`) rather than pre-emptively here.

---

### 0.1 Base Conventions

**Base path:** `/api/v1`

**Auth types used throughout the feature files:**

| Label | Meaning |
|---|---|
| `Public` | No token required. |
| `Authenticated` | Bearer JWT required; any role (Student, Parent, Tutor, or Admin). |
| `Student` | Bearer JWT required; `req.user.role === STUDENT`. |
| `Parent` | Bearer JWT required; `req.user.role === PARENT`. |
| `Student\|Parent` | Bearer JWT required; either role, further scoped to the caller's own `StudentProfile` or an `ACTIVE` `ParentStudentRelationship` they hold to the target student (Section 4.2's account model, FR-AC-002–007) — a Grade 1–5 parent acting "on the student's behalf" and a Grade 6–12 student acting independently both resolve to the same endpoint. |
| `Tutor` | Bearer JWT required; `req.user.role === TUTOR`. |
| `Admin` | Bearer JWT required; `req.user.role === ADMIN`. |
| `Webhook` | No JWT. Verified instead via the provider's own signature header (see 0.5). Used only by the Chapa payment webhook. |

Role checks are implemented via `authMiddleware` (verifies the JWT, attaches `req.user`) followed by `requireRole(...roles)` where applicable, per Doc 05a Section 0. There is no separate `adminOnly.middleware.ts` — `requireRole(ADMIN)` covers it. Ownership checks beyond role (e.g. "this recording belongs to a cohort this student was a member of") are enforced in the service layer, not the route middleware, and are called out per-endpoint below where they apply.

**Standard success envelope** (`SuccessResponse`):
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": { }
}
```

**Standard error envelope** (`ErrorResponse`, produced via `throw new ApiError(statusCode, message, errors[])`):
```json
{
  "statusCode": 400,
  "success": false,
  "message": "string",
  "errors": []
}
```

**Validation:** every endpoint accepting input is validated by a Zod schema via the `validate(schema)` middleware before the controller runs, per the schema files listed in Doc 05a (`src/schemas/<feature>.schema.ts`). Feature files below show the shape of the relevant Zod schema per endpoint. A `400` with `errors: []` populated field-by-field is the standard shape for validation failures and is **not** re-documented per endpoint unless the endpoint has additional business-rule validation beyond schema shape (e.g. the tutor two-subject cap, the 12-hour reschedule boundary).

---

### 0.2 Common Error Statuses

Not re-documented on every endpoint unless the condition is specific to that endpoint:

| Status | Meaning |
|---|---|
| 400 | Zod validation failure |
| 401 | Missing, invalid, or expired JWT |
| 403 | Valid JWT, but wrong role or fails an ownership check (e.g. a student requesting another student's recording) |
| 404 | Resource not found |
| 409 | Request conflicts with current state (e.g. a third subject-ranking attempt, a relationship already `ACTIVE`) |
| 500 | Unexpected server error (never intentionally thrown) |

---

### 0.3 Non-Error Edge Cases

An empty or substituted result is not an error. Examples specific to AKEWTutor:

- A 1-to-1 tutor search or recommendation list with zero results → returns an empty array with `200`, not a `404` — this is what starts the "No Exact Match" flow (Path B) client-side, not a server error.
- A 1-to-3/1-to-5 auto-match request that hasn't yet found a candidate cohort → returns `status: "SEARCHING"` with `200`, not a `404` or a `202`.
- A student/parent with no payment history yet, no notifications yet, or no recordings yet → returns an empty array with `200`.
- A recording past its 90-day retention and not marked "Keep permanently" → its signed-URL endpoint returns `404` ("Recording no longer available") — this **is** treated as not-found, since the object is genuinely gone, distinct from an empty list.
- A stale booking approval that hasn't yet crossed the 48-hour/5-day thresholds → returns normally with `200`; the `isOverdue` / `delayNotified` flags are simply `false`. There is no separate "check staleness" endpoint — see 0.4.

---

### 0.4 System-Driven State — No Trigger Endpoint

Several state transitions in AKEWTutor are cron/interval-driven (Doc 05a `src/jobs/*.job.ts`), not client-initiated:

| Job | Effect | Where the resulting state is exposed |
|---|---|---|
| `groupFormationWindow.job.ts` | Closes a `Cohort`'s formation window; forms the class at whatever size was reached | `GET /cohorts/me`, `GET /admin/matching/queue` — `Cohort.status`, `Cohort.groupFormationWindowExpiresAt` |
| `zeroMatchEscalation.job.ts` | Auto-escalates a 48h continuous zero-match `MatchRequest` into Path B | `GET /matching/requests/me` — `MatchRequest.status`, `zeroMatchSince` |
| `staleApproval.job.ts` | Flags a `Cohort`/`MatchRequest` overdue at 48h; escalates + notifies the student at 5 days | `GET /admin/matching/queue` — `adminOverdueNotifiedAt`, `studentDelayNotifiedAt` |
| `recordingMissingCheck.job.ts` | Flags a session's recording `MISSING` at 2h, `ESCALATED` at 24h | `GET /sessions/:sessionId`, `GET /admin/library/recording-compliance` — `ScheduledSession.recordingStatus` |
| `paymentPause.service.rescheduleSessionsDuringPause` (triggered on payment resume, not a standalone cron) | Reschedules any session that fell inside an active pause | `GET /sessions/:sessionId` — `status: "PAYMENT_PAUSE_RESCHEDULED"` |
| `monthlyPayout.job.ts` | Generates the monthly `Payout` batch | `GET /tutors/me/earnings`, `GET /admin/payouts` |

None of these has a corresponding `POST /.../run` endpoint. Client applications poll or re-fetch the relevant `GET` endpoint to observe the resulting state; there is nothing to trigger directly. This is called out explicitly in each feature file wherever it's relevant, so an implementer doesn't go looking for an endpoint that was never meant to exist.

---

### 0.5 Third-Party Integration Notes

| Provider | Used for | Integration shape |
|---|---|---|
| Chapa | Payment processing (Section 16) | `POST /payments/initiate` returns a Chapa checkout URL; Chapa calls back to `POST /payments/webhook/chapa` (Webhook auth — HMAC signature verified via the header Chapa specifies, not a JWT) |
| Geez SMS | Phone verification, SMS notification channel | Called server-side only, from `notification.service.ts` / `auth.service.ts`; no client-facing endpoint |
| Brevo | Email verification, email notification channel | Same as above — server-side only |
| Cloudflare R2 | Recording and Library material storage | Client never talks to R2 directly; `GET /recordings/:id/signed-url` returns a short-lived signed URL generated server-side |
| Jitsi | Video conferencing | No API integration — the tutor generates a link via the public Jitsi instance directly and submits it via `POST /sessions/:sessionId/link`; AKEWTutor's backend stores and delivers the URL but does not call a Jitsi API |

---

### 0.6 Timestamps, Money, and Pagination

- **Timestamps:** all `DateTime` fields are ISO 8601 strings (UTC), matching Doc 04's schema.
- **Money:** all monetary fields (`amount`, `pricePerStudentPerHour`, etc.) are returned as strings representing a `Decimal` (e.g. `"350.00"`), never as a JS `number`, to avoid floating-point precision loss over the wire — matching the `Decimal`-not-`Float` decision in Doc 04 Section 4.0.
- **Pagination:** list endpoints that can grow unbounded (notifications, payment history, admin people/queues, message history, admin reports) accept `?page=1&limit=20` query params, defaulting to `page=1`, `limit=20`. Endpoints returning an inherently small, bounded set (e.g. a tutor's own subject rankings, max 2) are not paginated.

---

### 0.7 Feature-to-Router Mapping

| Feature | Mounted at (examples) | Router file(s) |
|---|---|---|
| Shared Config | `/api/v1/auth`, `/api/v1/notifications`, `/api/v1/admin/announcements`, `/api/v1/policies`, `/api/v1/admin/policies` | `auth.routes.ts`, `notification.routes.ts`, `adminAnnouncement.routes.ts`, `policy.routes.ts` |
| Accounts & Guardianship | `/api/v1/students/me`, `/api/v1/guardianship`, `/api/v1/tutors/me`, `/api/v1/tutors/me/availability`, `/api/v1/subjects`, `/api/v1/admin/subjects`, `/api/v1/admin/tutors`, `/api/v1/admin/people` | `studentProfile.routes.ts`, `guardianship.routes.ts`, `tutorProfile.routes.ts`, `availability.routes.ts`, `subject.routes.ts`, `adminTutorVerification.routes.ts`, `adminPeople.routes.ts` |
| Matching & Cohorts | `/api/v1/matching`, `/api/v1/cohorts`, `/api/v1/admin/matching`, `/api/v1/format-switch` | `matching.routes.ts`, `cohort.routes.ts`, `adminMatching.routes.ts`, `formatSwitch.routes.ts` |
| Class Delivery, Recording & Library | `/api/v1/sessions`, `/api/v1/recording-consent`, `/api/v1/recordings`, `/api/v1/library`, `/api/v1/admin/library`, `/api/v1/reschedule`, `/api/v1/session-miss`, `/api/v1/assessments` | `session.routes.ts`, `recordingConsent.routes.ts`, `recording.routes.ts`, `library.routes.ts`, `reschedule.routes.ts`, `sessionMiss.routes.ts`, `weeklyAssessment.routes.ts` |
| Messaging | `/api/v1/messaging`, `/api/v1/admin/messaging` | `messaging.routes.ts`, `adminMessaging.routes.ts` |
| Gamification & Engagement | `/api/v1/gamification`, `/api/v1/admin/badges`, `/api/v1/admin/challenges` | `xp.routes.ts`, `badge.routes.ts`, `challenge.routes.ts` |
| Payments & Earnings | `/api/v1/payments`, `/api/v1/payment-pause`, `/api/v1/pricing`, `/api/v1/admin/pricing`, `/api/v1/admin/refunds`, `/api/v1/tutors/me/earnings`, `/api/v1/admin/payouts`, `/api/v1/promotions`, `/api/v1/admin/promotions` | `payment.routes.ts`, `paymentPause.routes.ts`, `pricing.routes.ts`, `refund.routes.ts`, `earning.routes.ts`, `payout.routes.ts`, `promotion.routes.ts` |
| Support, Trust & Admin Reporting | `/api/v1/complaints`, `/api/v1/support`, `/api/v1/admin/disputes`, `/api/v1/admin/reports` | `complaint.routes.ts`, `adminDispute.routes.ts`, `adminReporting.routes.ts` |

Note the deliberate path splits, consistent with the feature-ownership model in Doc 07: public subject browsing lives at `/subjects`, admin subject management at `/admin/subjects`; public pricing lives at `/pricing`, admin pricing management at `/admin/pricing`; public active promotions live at `/promotions`, admin promotion management at `/admin/promotions`; public policy reading lives at `/policies`, admin policy publishing at `/admin/policies` — same underlying entity in each pair, two different routers/feature-file sections, matching how the front end (Doc 05b) also splits its public vs. admin pages per feature.

**Implementation reference format:** every endpoint below lists `Implemented in:` pointing to the exact controller → service → schema file paths from Doc 05a, registered in `src/routes/index.ts` per that document's Section 9/10.

---

**Next:** proceed to → [01. Shared Config API]
