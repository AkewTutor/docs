## Project: AKEWTutor — API Specification
**Feature:** Support, Trust & Admin Reporting
**Conventions:** see 0.1–0.7 in `00-api-conventions.md` — base path `/api/v1`, envelopes, auth labels, common errors, pagination.

**Owns:** ComplaintReport. **Depends on:** Shared Config, Messaging, Class Delivery & Library (hard, per Feature Decomposition §1.1) — a `ComplaintReport` can reference a `MessageThread` or a `ScheduledSession`. **Soft-integrates with:** Payments & Earnings (a refund can be issued as a resolution action, no FK), Accounts & Guardianship (a tutor suspension can be issued as a resolution action, no FK).

> ℹ️ **M1 fix — resolved, not just flagged:** Doc 03's UC-66 main flow originally said it routes to "UC-88 in Section Q," a typo — UC-88 is *Admin manages leaderboards and achievements*, unrelated to disputes. Corrected directly in `03-usecases.md`: UC-66 now points to **UC-87** (*Admin manages complaints and disputes*, FR-AD-017), which is what this file links to below.

---

### 8.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| POST | /complaints | Student\|Parent\|Tutor | UC-66 | FR-SP-042, FR-TU-022, FR-SC-003 |
| GET | /complaints/me | Student\|Parent\|Tutor | UC-66 | FR-SP-042, FR-TU-022, FR-SC-003 |
| GET | /complaints/:complaintId | Student\|Parent\|Tutor | UC-66 | FR-SP-042, FR-TU-022, FR-SC-003 |
| GET | /admin/disputes | Admin | UC-87 | FR-AD-017 |
| GET | /admin/disputes/:complaintId | Admin | UC-87 | FR-AD-017, FR-MS-004 |
| PATCH | /admin/disputes/:complaintId | Admin | UC-87 (resolution may invoke UC-83 refund, UC-78 suspension) | FR-AD-017 |
| GET | /support/contact | Public | UC-67, UC-69 | FR-SP-043, FR-TU-023, FR-PB-006 |
| GET | /admin/reports/platform-health | Admin | UC-90, UC-91 | FR-AD-020, FR-AD-021, FR-AD-022 |
| GET | /admin/reports/tutor-performance | Admin | UC-91 | FR-AD-022 |

---

### 8.2 Endpoint Detail

#### POST /complaints

**Purpose:** File a complaint, claim, or report against a session, tutor, payment, or message thread (UC-66; FR-SP-042 for Student/Parent, FR-TU-022 for Tutor, FR-SC-003 as the shared underlying reporting requirement). This is the sole entry point into the Admin dispute-review queue (UC-87) — every `ComplaintReport` is investigated via `GET /admin/disputes/:complaintId` regardless of category.

**Auth:** Student|Parent|Tutor

**Request body:**
```json
{
  "category": "string, required — SESSION_ISSUE | TUTOR_CONDUCT | PAYMENT_ISSUE | MESSAGE_ISSUE | OTHER",
  "description": "string, required, 10-2000 chars",
  "relatedCohortId": "uuid, optional",
  "relatedSessionId": "uuid, optional",
  "relatedPaymentId": "uuid, optional"
}
```
At least one of `relatedCohortId` / `relatedSessionId` / `relatedPaymentId` is required unless `category` is `OTHER` — enforced as a business-rule validation beyond the base Zod shape (see 0.1).

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "category": "SESSION_ISSUE",
    "status": "OPEN",
    "createdAt": "2026-09-06T15:00:00Z"
  }
}
```
Filing a complaint writes a `Notification` to the Admin queue, through the same pipeline as any other notification type (see 01-shared-config-api.md §1.2 `POST /admin/announcements` for the pattern), and does not itself close, mute, or otherwise affect the referenced message thread or session — that is a separate Admin action (see `PATCH /admin/disputes/:complaintId` below and `POST /admin/messaging/threads/:threadId/close`).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | No related entity provided and `category` is not `OTHER` | "A complaint must reference a session, payment, or cohort unless filed as a general (OTHER) report" |
| 403 | `relatedCohortId`/`relatedSessionId`/`relatedPaymentId` does not belong to the caller | "You can only file a complaint about your own sessions, payments, or cohorts" |

**Implemented in:** `src/controllers/complaint.controller.ts → fileComplaint` · `src/services/complaint.service.ts → createComplaint` · `src/schemas/complaint.schema.ts → createComplaintSchema`

---

#### GET /complaints/me

**Purpose:** List the caller's own filed complaints and their current status (UC-66, FR-SP-042/FR-TU-022/FR-SC-003 — same requirement as filing; Doc 02 does not carry a separate FR ID for viewing one's own complaint history).

**Auth:** Student|Parent|Tutor

**Query params:**
```
?status=OPEN|UNDER_REVIEW|RESOLVED|DISMISSED&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "complaints": [
      {
        "id": "uuid",
        "category": "SESSION_ISSUE",
        "status": "UNDER_REVIEW",
        "createdAt": "2026-09-06T15:00:00Z",
        "resolvedAt": null
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
A caller with no complaints filed yet returns `complaints: []` — not an error (see 0.3).

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/complaint.controller.ts → listMyComplaints` · `src/services/complaint.service.ts → listForUser`

---

#### GET /complaints/:complaintId

**Purpose:** View the detail and resolution outcome of a single complaint the caller filed (UC-66, FR-SP-042/FR-TU-022/FR-SC-003). Does not expose internal Admin resolution notes — see `GET /admin/disputes/:complaintId` for the Admin-facing view of the same record.

**Auth:** Student|Parent|Tutor — caller must be the original reporter; otherwise `403`.

**Path params:** `complaintId` — ComplaintReport UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "category": "SESSION_ISSUE",
    "description": "Tutor did not join the session at the scheduled time.",
    "status": "RESOLVED",
    "resolutionAction": "REFUND_ISSUED",
    "createdAt": "2026-09-06T15:00:00Z",
    "resolvedAt": "2026-09-07T09:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller is not the original reporter | "Not authorized to view this complaint" |
| 404 | Complaint doesn't exist | "Complaint not found" |

**Implemented in:** `src/controllers/complaint.controller.ts → getMyComplaint` · `src/services/complaint.service.ts → getForReporter`

---

#### GET /admin/disputes

**Purpose:** List the Admin dispute-review queue (UC-87, FR-AD-017 — "allow Admin to manage complaints, claims, and disputes"), filterable by status and category so Admin can triage session-conduct issues separately from payment disputes.

**Auth:** Admin

**Query params:**
```
?status=OPEN|UNDER_REVIEW|RESOLVED|DISMISSED&category=SESSION_ISSUE|TUTOR_CONDUCT|PAYMENT_ISSUE|MESSAGE_ISSUE|OTHER&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "complaints": [
      {
        "id": "uuid",
        "reporterRole": "PARENT",
        "category": "SESSION_ISSUE",
        "status": "OPEN",
        "relatedSessionId": "uuid",
        "createdAt": "2026-09-06T15:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
An empty queue returns `complaints: []` with `200` (see 0.3) — this is the normal steady state, not an error condition.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/adminDispute.controller.ts → listQueue` · `src/services/adminDispute.service.ts → listDisputeQueue`

---

#### GET /admin/disputes/:complaintId

**Purpose:** View full complaint detail for investigation (UC-87, FR-AD-017), including any linked message thread the complaint references — this is where UC-87 Step 1 ("Admin reviews the complaint, including any flagged message threads (UC-60)") and FR-MS-004 (Admin's ability to view a thread for dispute investigation) connect the two features. This endpoint returns *references* to related resources rather than embedding them — Admin follows up with `GET /admin/messaging/threads/:threadId` (05-messaging-api.md §5.2) or `GET /sessions/:sessionId` (04-class-delivery-library-api.md) as needed, keeping each feature's data ownership intact rather than duplicating it here.

**Auth:** Admin

**Path params:** `complaintId` — ComplaintReport UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "reporterId": "uuid",
    "reporterRole": "PARENT",
    "category": "SESSION_ISSUE",
    "description": "Tutor did not join the session at the scheduled time.",
    "status": "OPEN",
    "relatedCohortId": "uuid",
    "relatedSessionId": "uuid",
    "relatedPaymentId": null,
    "relatedThreadId": "uuid",
    "createdAt": "2026-09-06T15:00:00Z",
    "resolutionAction": null,
    "resolutionNotes": null,
    "resolvedById": null,
    "resolvedAt": null
  }
}
```

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/adminDispute.controller.ts → getDisputeDetail` · `src/services/adminDispute.service.ts → getDisputeForReview`

---

#### PATCH /admin/disputes/:complaintId

**Purpose:** Resolve or dismiss a complaint (UC-87 Step 2, FR-AD-017). Doc 03 names the two possible downstream actions explicitly: "Admin resolves it — which may include a refund (UC-83), suspension (UC-78), or re-matching action." This is the one endpoint in the feature that reaches into another feature's data as a side effect: setting `resolutionAction: REFUND_ISSUED` triggers a service-layer call into Payments & Earnings (`refund.service.ts`, corresponding to UC-83, no FK — see Feature Decomposition §1.1 soft-dependency note), and `TUTOR_SUSPENDED` triggers a service-layer call into Accounts & Guardianship (`tutorProfile.service.ts`, corresponding to UC-78) to set the tutor's account status. A re-matching action is not modeled as a `resolutionAction` value here — it would be initiated separately via Matching & Cohorts' own endpoints, with the complaint simply marked `RESOLVED` once that's done. Neither the refund nor suspension call is exposed as a separate client-facing endpoint here; both are internal to this action.

**H4 fix:** the refund side effect no longer accepts a free-form `refundAmount`. It computes `sessionsRemaining`/`totalSessionsBilled` server-side from the affected student's current billing cycle — the same non-nullable, fully-audited proration formula used everywhere else refunds are issued (Section 13; Doc 04 `Refund` notes) — tagged `reason: ADMIN_DISPUTE_RESOLUTION`. Admin's only input is *which* membership's cycle to prorate against, not the amount.

**Auth:** Admin

**Path params:** `complaintId` — ComplaintReport UUID

**Request body:**
```json
{
  "status": "string, required — UNDER_REVIEW | RESOLVED | DISMISSED",
  "resolutionAction": "string, optional — NO_ACTION | WARNING_ISSUED | REFUND_ISSUED | TUTOR_SUSPENDED, required if status is RESOLVED",
  "resolutionNotes": "string, required, internal note explaining the decision",
  "affectedCohortMembershipId": "string (UUID), required if resolutionAction is REFUND_ISSUED — identifies which student's CohortMembership/current billing cycle to prorate the refund against (the dispute-detail view, GET /admin/disputes/:complaintId, surfaces the candidate membership(s) linked to the complaint's aboutSessionId/aboutThreadId for Admin to pick from — most disputes resolve to exactly one, but a group-cohort complaint naming the whole class could in principle need one refund call per affected membership)"
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
    "status": "RESOLVED",
    "resolutionAction": "REFUND_ISSUED",
    "resolvedById": "uuid",
    "resolvedAt": "2026-09-07T09:00:00Z"
  }
}
```
The reporter receives a `Notification` (`type: COMPLAINT_RESOLVED`) once `status` moves to `RESOLVED` or `DISMISSED`, through the standard notification pipeline.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `status: RESOLVED` submitted without `resolutionAction` | "A resolution action is required to resolve a complaint" |
| 400 | `resolutionAction: REFUND_ISSUED` submitted without `affectedCohortMembershipId` | "affectedCohortMembershipId is required for this resolution action" |
| 400 | `affectedCohortMembershipId` has no active, paid billing cycle to prorate | "affectedCohortMembershipId does not have an active, paid billing cycle to prorate" |
| 409 | Complaint already `RESOLVED` or `DISMISSED` | "This complaint has already been closed" |

**Implemented in:** `src/controllers/adminDispute.controller.ts → resolveDispute` · `src/services/adminDispute.service.ts → resolveDispute` · `src/schemas/complaint.schema.ts → resolveDisputeSchema`

---

#### GET /support/contact

**Purpose:** Return AKEWTutor's support contact channel — this single read serves two related use cases from Doc 03 Section L: UC-67 (*Contact customer support*, FR-SP-043/FR-TU-023 — the general "reach a human" channel) and UC-69 (*Use the emergency contact channel*, FR-PB-006 — the phone/Telegram channel for urgent, time-sensitive issues around a session). Both resolve to the same published contact details; AKEWTutor does not distinguish "general support" from "emergency" by routing to a different channel, only by what the user is contacting about. This exists because support contact is deliberately manual, not an in-app ticketing flow (Doc 01 §1.7 Assumption #4; Doc 03 UC-69 explicitly notes "no automated confidentiality or response-time guarantee is implied by this channel; handling is manual by design") — there is no `POST /support/*` endpoint because there is nothing for the backend to route; this is a static, Admin-configurable read.

**Auth:** Public

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "phone": "+251900000000",
    "telegramHandle": "@akewtutor_support",
    "hours": "Mon–Sat, 8:00–20:00 EAT"
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/complaint.controller.ts → getSupportContact` · `src/services/complaint.service.ts → getSupportContactInfo`

---

#### GET /admin/reports/platform-health

**Purpose:** Admin-facing operational dashboard summary, combining two related Doc 03 use cases: UC-90 (*Admin uses the stale-approval and support-management tools*, FR-AD-020 — the `overdueMatchApprovals` and `recordingComplianceEscalations` figures) and UC-91 (*Admin views platform statistics and reports*, FR-AD-021/FR-AD-022 — platform-wide activity stats and tutor performance/badge oversight). This endpoint reads across feature boundaries (soft dependency only — no FK), matching the note in Feature Decomposition §1.1 that `support-trust-admin` is "an integration surface, not a domain of its own data."

**Auth:** Admin

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "openDisputes": 4,
    "overdueMatchApprovals": 2,
    "recordingComplianceEscalations": 1,
    "pendingPayoutBatches": 0,
    "generatedAt": "2026-09-06T15:00:00Z"
  }
}
```
This is a read-only, computed-on-request summary — it is not a stored `Report` entity, and there is no corresponding `POST` (consistent with the system-driven-state pattern in 0.4: the underlying flags like `adminOverdueNotifiedAt` and `recordingStatus: ESCALATED` are each already job-maintained in their owning feature; this endpoint only aggregates counts across them).

**Error responses:** none.

**Implemented in:** `src/controllers/adminReporting.controller.ts → getPlatformHealth` · `src/services/adminReporting.service.ts → aggregatePlatformHealth`

---

#### GET /admin/reports/activity

**Gap closed (follow-up to H5).** Previously implied only by Doc 05a's `getActivityHistory` service signature and UC-91/FR-AD-021's "platform statistics, reports, and activity history" — flagged in `08-function-level-specification/backend/8-8-support-trust-admin.md` as not yet independently documented. Closed here the same way H5 closed `tutor-performance`: a real endpoint entry rather than an implied one. Backs UC-91 (FR-AD-021), reading across every feature as a chronological, filterable event log — same "integration surface, no owned data" pattern as `platform-health` and `tutor-performance` above.

**Auth:** Admin

**Query params:**
```
?page=1&limit=20 (00-api-conventions §0.3 pagination pattern)
?dateRange=7d|30d|90d|all (default 30d)
?eventType=BOOKING|PAYMENT|DISPUTE|TUTOR_VERIFICATION|REFUND|PAYOUT (optional filter; omit for all types)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "events": [
      {
        "id": "uuid",
        "eventType": "DISPUTE",
        "summary": "Complaint filed against tutor by student",
        "relatedEntityType": "ComplaintReport",
        "relatedEntityId": "uuid",
        "occurredAt": "2026-09-05T10:12:00Z"
      }
    ],
    "pagination": { "page": 1, "limit": 20, "total": 138, "totalPages": 7 }
  }
}
```
Each `eventType` maps to a read of an existing table's `createdAt` (or equivalent lifecycle timestamp), not a new stored `ActivityEvent` entity — consistent with `platform-health`/`tutor-performance`'s "computed-on-request, no new source of truth" pattern: `BOOKING` from `Cohort.createdAt`/`adminApprovedAt`, `PAYMENT` from `Payment.createdAt`, `DISPUTE` from `ComplaintReport.createdAt`, `TUTOR_VERIFICATION` from `TutorProfile.verifiedAt`, `REFUND` from `Refund.createdAt`, `PAYOUT` from `Payout.createdAt`. The merged, paginated, chronological view across these six sources is what `getActivityHistory` computes — no cross-table denormalized log is ever written.

**Error responses:** `400` if `dateRange`/`eventType` fails the enum shape (Zod, per 00-api-conventions §0.1).

**Implemented in:** `src/controllers/adminReporting.controller.ts → getActivity` · `src/services/adminReporting.service.ts → getActivityHistory`

---

#### GET /admin/reports/tutor-performance

**H5 fix — new endpoint.** `TutorPerformanceTable.tsx` (Doc 05b's file inventory, Doc 07 §8.5) had no backing data source in earlier drafts; this closes that gap rather than leaving it flagged. Backs UC-91 (FR-AD-022 — tutor performance/badge oversight), reading across `accounts-guardianship`, `class-delivery-library`, `gamification-engagement`, and `support-trust-admin` (soft dependency only, same integration-surface pattern as `platform-health` above — no FK).

**Auth:** Admin

**Query params:**
```
?page=1&limit=20 (00-api-conventions §0.3 pagination pattern)
?tutorId=uuid (optional — single-tutor lookup instead of the full paginated list)
?sortBy=uniqueStudentsTaught|badgeCount|complaintCount (default uniqueStudentsTaught, descending)
?verificationStatus=PENDING|VERIFIED|REJECTED (optional filter, default VERIFIED-only — a pending/rejected tutor has no meaningful performance history yet)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "tutors": [
      {
        "tutorId": "uuid",
        "fullName": "string",
        "verificationStatus": "VERIFIED",
        "uniqueStudentsTaught": 14,
        "activeCohortCount": 3,
        "completedSessionCount": 210,
        "tutorCausedMissCount": 2,
        "badgeCount": 5,
        "complaintCount": 1,
        "createdAt": "2026-01-10T09:00:00Z"
      }
    ],
    "pagination": { "page": 1, "limit": 20, "total": 47, "totalPages": 3 }
  }
}
```
Every figure is computed on request from existing owned/related data, not a new stored aggregate table: `uniqueStudentsTaught` from `TutorProfile` (M6 fix — the platform-wide canonical field name, see below), `completedSessionCount`/`tutorCausedMissCount` from `ScheduledSession`/`SessionMiss` (Class Delivery & Library), `badgeCount` from `TutorBadge` (Gamification & Engagement), `complaintCount` from `ComplaintReport.aboutUserId` (this feature's own data). This mirrors `platform-health`'s already-established pattern of a read-only, computed-on-request response with no corresponding `POST`.

**Error responses:** none beyond common auth/validation.

**Implemented in:** `src/controllers/adminReporting.controller.ts → getTutorPerformance` · `src/services/adminReporting.service.ts → getTutorPerformanceHistory`

---

**Next:** proceed to → [07. Frontend Specification]
