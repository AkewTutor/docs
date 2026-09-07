## Project: AKEWTutor — API Specification
**Feature:** Matching & Cohorts
**Conventions:** see 0.1–0.7 in `00-api-conventions.md`. See also 0.4 for system-driven state (group-formation window closing, zero-match escalation, stale-approval flags) that has no trigger endpoint of its own.

**Owns:** MatchRequest, TutorExclusion, Cohort, CohortMembership, FormatSwitchRequest. **Depends on:** Accounts & Guardianship (hard, per Feature Decomposition §1.1) — matching cannot run without a student's academic profile or a tutor's ranked subjects/availability existing first.

**Links back to:** [00. API Conventions], [05a. Backend Folder & File Structure §3], [Feature Decomposition §1]
**Links forward to:** [8-3. Backend Function-Level Spec: Matching & Cohorts]

---

### 3.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /matching/tutors/search | Student\|Parent | UC-20 | FR-SP-017, FR-SP-018, FR-SP-019 |
| GET | /matching/tutors/recommendations | Student\|Parent | UC-21 | FR-SP-020, FR-SP-021, FR-MA-001 |
| GET | /matching/tutors/:tutorId | Student\|Parent | UC-22 | FR-SP-022–027 |
| POST | /matching/select-tutor | Student\|Parent | UC-23 | FR-MA-002, FR-SP-028 |
| POST | /matching/no-exact-match | Student\|Parent | UC-25 | FR-MA-007, FR-SP-029 |
| POST | /matching/group-format | Student\|Parent | UC-28 | FR-MA-012, FR-SP-030 |
| GET | /matching/requests/me | Student\|Parent | UC-20, UC-21, UC-25, UC-26, UC-28 | FR-MA-001–018 |
| GET | /cohorts/me | Student\|Parent | UC-14, UC-28 | FR-SP-030 |
| GET | /cohorts/:cohortId/members | Student\|Parent\|Tutor | UC-22, UC-28 | FR-SP-030 (visibility split, Doc 02 §5.6) |
| GET | /admin/matching/queue | Admin | UC-24, UC-31, UC-35 | FR-AD-005 |
| POST | /admin/matching/:cohortId/approve | Admin | UC-24, UC-31 | FR-MA-004, FR-MA-014, FR-AD-007 |
| POST | /admin/matching/:cohortId/reject | Admin | UC-32 | Section 8 (Admin Rejection Handling), FR-AD-007 |
| POST | /admin/matching/manual-assign | Admin | UC-27, UC-30 | FR-MA-008, FR-MA-017, FR-AD-006 |
| POST | /format-switch | Student\|Parent | UC-61 | FR-SP-045–049 |

---

### 3.2 Endpoint Detail

#### GET /matching/tutors/search

**Purpose:** 1-to-1 tutor search by subject, grade, and filters (UC-20, Path A entry point). Not applicable to 1-to-3/1-to-5 — see `POST /matching/group-format` for that path.

**Auth:** Student|Parent — caller's `formatPreference` must be `ONE_TO_ONE`, or the request is rejected (see error table).

**Query params:**
```
?studentId=uuid (required for Parent)
&subjectId=uuid, required
&grade=integer, required
&scheduleAvailability=object, optional
&budget=decimal, optional — hard filter, excludes tutors priced above this
&language=string, optional — hard filter
&priceMax=decimal, optional — additional design-time filter
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
        "name": "Selam T.",
        "profilePictureUrl": "https://...",
        "verificationStatus": "VERIFIED",
        "pricePerStudentPerHour": "350.00"
      }
    ]
  }
}
```
Zero results is a normal `200` with `tutors: []` (see 0.3) — this is what surfaces the "No Exact Match" option (UC-25) on the client, not an error.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Caller's `formatPreference` is not `ONE_TO_ONE` | "Search is only available for the 1-to-1 format — see /matching/group-format for 1-to-3/1-to-5" |

**Implemented in:** `src/controllers/matching.controller.ts → searchTutors` · `src/services/matching.service.ts → searchOneToOneTutors` · `src/schemas/matching.schema.ts → searchTutorsQuerySchema`

---

#### GET /matching/tutors/recommendations

**Purpose:** 1-to-1 tutor recommendations with a calculated match percentage (UC-21, FR-MA-001, FR-SP-020/021).

**Auth:** Student|Parent

**Query params:**
```
?studentId=uuid (required for Parent)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "recommendations": [
      {
        "tutorId": "uuid",
        "name": "Selam T.",
        "profilePictureUrl": "https://...",
        "matchPercentage": 95
      }
    ],
    "matchRequestId": "uuid",
    "zeroMatchSince": null
  }
}
```
When `recommendations` is empty, `zeroMatchSince` reflects the timestamp the zero-match streak began — the client can use this to display a live countdown toward the 48-hour auto-escalation (UC-26); this is read-only, reflecting `zeroMatchEscalation.job.ts` state per 0.4, not something this endpoint triggers.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/matching.controller.ts → getRecommendations` · `src/services/matching.service.ts → recommendTutorsWithMatchPercent`

---

#### GET /matching/tutors/:tutorId

**Purpose:** Full 1-to-1 tutor profile view (UC-22, FR-SP-022–027) — never used for 1-to-3/1-to-5, where students see only name+photo via `GET /cohorts/:cohortId/members`.

**Auth:** Student|Parent

**Path params:** `tutorId` — TutorProfile UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "tutorId": "uuid",
    "name": "Selam T.",
    "profilePictureUrl": "https://...",
    "verificationStatus": "VERIFIED",
    "educationInstitution": "Addis Ababa University",
    "degree": "BSc Mathematics",
    "subjectsAndGrades": [{ "subjectName": "Mathematics", "grades": "1-12" }],
    "uniqueStudentsTaught": 42,
    "availableSlots": [
      { "startTime": "2026-09-08T16:00:00Z", "endTime": "2026-09-08T17:00:00Z" }
    ]
  }
}
```
`uniqueStudentsTaught` is a distinct-student count, never a raw session tally (FR-SP-025 amendment). Optional fields (`degree`) are simply omitted if unset, not treated as an error.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 404 | Tutor doesn't exist, or is not yet `VERIFIED` | "Tutor not found" |

**Implemented in:** `src/controllers/matching.controller.ts` (co-located with `searchTutors`) · `src/services/matching.service.ts`

---

#### POST /matching/select-tutor

**Purpose:** Student selects a preferred tutor from recommendations — Path A (UC-23, FR-MA-002, FR-SP-028). Creates a booking request routed to Admin.

**Auth:** Student|Parent

**Request body:**
```json
{
  "studentId": "uuid, required for Parent",
  "tutorId": "uuid, required"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "cohortId": "uuid",
    "status": "PENDING_ADMIN_APPROVAL",
    "tutorId": "uuid"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `tutorId` is on the caller's `TutorExclusion` list (previously rejected) | "This tutor is not available — please choose from your current recommendations" |
| 409 | Caller already has an active `MatchRequest`/`Cohort` in progress | "You already have a pending or active match" |

**Implemented in:** `src/controllers/matching.controller.ts → selectTutor` · `src/services/matching.service.ts → selectTutor` · `src/schemas/matching.schema.ts → selectTutorSchema`

---

#### POST /matching/no-exact-match

**Purpose:** Student manually triggers Path B — Admin manual assignment (UC-25, FR-MA-007, FR-SP-029). Also the same downstream state as the automatic 48-hour zero-match escalation (UC-26) and the automatic hand-off when a secondary-subject search also fails (FR-TU-008).

**Auth:** Student|Parent

**Request body:**
```json
{
  "studentId": "uuid, required for Parent"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "matchRequestId": "uuid",
    "status": "PENDING_ADMIN_ASSIGNMENT"
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/matching.controller.ts → noExactMatch` · `src/services/matching.service.ts → triggerNoExactMatch`

---

#### POST /matching/group-format

**Purpose:** Entry point into Path C — system auto-match for 1-to-3/1-to-5 (UC-28, FR-MA-012, FR-SP-030). No search or selection UI; the student's preference simply enters the auto-match engine.

**Auth:** Student|Parent — caller's `formatPreference` must be `ONE_TO_THREE` or `ONE_TO_FIVE`.

**Request body:**
```json
{
  "studentId": "uuid, required for Parent",
  "subjectId": "uuid, required"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "matchRequestId": "uuid",
    "status": "SEARCHING"
  }
}
```
No `tutorId`, match percentage, or profile data is ever returned by this endpoint or any subsequent group-format status check, per the no-match-information rule for group formats (Doc 02 §8, Path C).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Caller's `formatPreference` is `ONE_TO_ONE` | "Use /matching/select-tutor or /matching/no-exact-match for the 1-to-1 format" |

**Implemented in:** `src/controllers/matching.controller.ts → requestGroupFormat` · `src/services/matching.service.ts → requestGroupFormat`

---

#### GET /matching/requests/me

**Purpose:** Check the status of the caller's in-progress match request, across any path (UC-20, UC-21, UC-25, UC-26, UC-28).

**Auth:** Student|Parent

**Query params:**
```
?studentId=uuid (required for Parent)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "subjectId": "uuid",
    "format": "ONE_TO_THREE",
    "path": "PATH_C",
    "status": "SEARCHING",
    "zeroMatchSince": null,
    "resultingCohortId": null
  }
}
```
`status: ZERO_MATCH_PENDING` covers both a manual "No Exact Match" click and the automatic 48-hour escalation reaching the same state (Doc 04 MatchRequest notes) — the client does not need to distinguish which triggered it.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 404 | No active `MatchRequest` exists for the caller | "No match request in progress" |

**Implemented in:** `src/controllers/matching.controller.ts` (status handler) · `src/services/matching.service.ts`

---

#### GET /cohorts/me

**Purpose:** The caller's current cohort(s) — confirmed or forming — powering the dashboard (UC-14) and group-format status screen (UC-28).

**Auth:** Student|Parent

**Query params:**
```
?studentId=uuid (required for Parent)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "cohorts": [
      {
        "cohortId": "uuid",
        "format": "ONE_TO_THREE",
        "status": "FORMING",
        "targetGroupSize": 3,
        "groupFormationWindowExpiresAt": "2026-09-08T15:00:00Z",
        "membershipStatus": "PENDING_PAYMENT"
      }
    ]
  }
}
```
No cohort yet (still `SEARCHING` at the `MatchRequest` level) returns `cohorts: []` — not an error.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/cohort.controller.ts → getMyCohort` · `src/services/cohort.service.ts`

---

#### GET /cohorts/:cohortId/members

**Purpose:** List a cohort's members, applying the profile-visibility split from Doc 02 §5.6 (UC-22, UC-28) — full detail is never returned here; that's `GET /matching/tutors/:tutorId` for 1-to-1 only.

**Auth:** Student|Parent|Tutor — caller must be a current member of the cohort (student/parent) or the assigned tutor; otherwise `403`.

**Path params:** `cohortId` — Cohort UUID

**Success response — 200 (as seen by a student in a 1-to-3/1-to-5 cohort):**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "cohortId": "uuid",
    "format": "ONE_TO_THREE",
    "tutor": {
      "name": "Selam T.",
      "profilePictureUrl": "https://..."
    }
  }
}
```
No `education`, `totalStudentsCount`, or `matchPercentage` field is ever present in the group-format response shape — this is enforced by the service layer returning a different, smaller DTO for group formats, not by the client hiding fields it received.

**Success response — 200 (as seen by the assigned tutor, any format):**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "cohortId": "uuid",
    "format": "ONE_TO_THREE",
    "students": [
      { "studentId": "uuid", "firstName": "Bethel", "grade": 6 }
    ]
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller is not a current member/assigned tutor of this cohort | "Not authorized to view this cohort" |

**Implemented in:** `src/controllers/cohort.controller.ts → getCohortMembers` · `src/services/cohort.service.ts`

---

#### GET /admin/matching/queue

**Purpose:** Admin's pending-approvals queue across all paths, with overdue items surfaced (UC-24, UC-31, UC-35, FR-AD-005). Reflects `staleApproval.job.ts` state per 0.4 — there is no separate "check staleness" call.

**Auth:** Admin

**Query params:**
```
?page=1&limit=20&overdueOnly=false&path=PATH_A|PATH_B|PATH_C
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "queue": [
      {
        "cohortId": "uuid",
        "path": "PATH_A",
        "format": "ONE_TO_ONE",
        "tutorId": "uuid",
        "studentIds": ["uuid"],
        "createdAt": "2026-09-04T10:00:00Z",
        "isOverdue": true,
        "adminOverdueNotifiedAt": "2026-09-06T10:00:00Z",
        "studentDelayNotifiedAt": null
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
Overdue items (48h+) are sorted to the top regardless of `overdueOnly` (Section 11 Definition of Done #1). A case that hasn't crossed 48 hours yet returns `isOverdue: false` — not omitted, not an error (0.3).

**Error responses:** none.

**Implemented in:** `src/controllers/adminMatching.controller.ts → listQueue` · `src/services/adminMatching.service.ts → listPendingApprovals`

---

#### POST /admin/matching/:cohortId/approve

**Purpose:** Approve a Path A booking or Path C auto-match (UC-24, UC-31, FR-MA-004/014, FR-AD-007).

**Auth:** Admin

**Path params:** `cohortId` — Cohort UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "cohortId": "uuid",
    "status": "PENDING_PAYMENT",
    "adminApprovedAt": "2026-09-06T15:00:00Z",
    "adminApprovedById": "uuid"
  }
}
```
Approval moves the cohort to `PENDING_PAYMENT` — schedule confirmation still requires payment completion via the Payments & Earnings feature (`POST /payments/initiate`), per FR-MA-005/010/015.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Cohort is not in a pending-approval state | "This case is not awaiting approval" |

**Implemented in:** `src/controllers/adminMatching.controller.ts → approve` · `src/services/adminMatching.service.ts → approveBooking`

---

#### POST /admin/matching/:cohortId/reject

**Purpose:** Reject a booking/auto-match, re-routing the student per path (UC-32, Section 8 Admin Rejection Handling).

**Auth:** Admin

**Path params:** `cohortId` — Cohort UUID

**Request body:**
```json
{
  "internalReason": "string, optional — never disclosed to the student"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "cohortId": "uuid",
    "status": "CANCELLED",
    "endedReason": "ADMIN_REJECTED",
    "reroute": {
      "path": "PATH_A",
      "tutorExcluded": "uuid",
      "newMatchRequestId": "uuid"
    }
  }
}
```
For Path A, `reroute.tutorExcluded` is the rejected tutor's ID, now written to `TutorExclusion` and excluded from the student's next recommendation set. For Path C, `reroute` has no `tutorExcluded` field — the request simply re-enters the auto-match queue. The student-facing notification is always the generic "assignment could not be confirmed" (UC-32) — `internalReason` never appears in any student-facing endpoint response.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Cohort is not in a pending-approval state | "This case is not awaiting approval" |

**Implemented in:** `src/controllers/adminMatching.controller.ts → reject` · `src/services/adminMatching.service.ts → rejectBooking`

---

#### POST /admin/matching/manual-assign

**Purpose:** Admin manually assigns a tutor for a Path B case (UC-27), or manually assembles a group for a Path C double-fail case (UC-30, FR-MA-008/017, FR-AD-006). Also used for tutor-exit group re-matching when a single eligible tutor can't take the full group (UC-33).

**Auth:** Admin

**Request body:**
```json
{
  "matchRequestIds": "array of MatchRequest UUIDs, required, min 1 — one for Path B, one or more for Path C group assembly",
  "tutorId": "uuid, required"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "cohortId": "uuid",
    "status": "PENDING_PAYMENT",
    "tutorId": "uuid",
    "studentIds": ["uuid"]
  }
}
```
Manual assignment goes straight to `PENDING_PAYMENT`, not another approval step, since this action **is** the Admin approval (UC-27's flow has no separate approve click after assignment).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `tutorId` is not `VERIFIED`, or lacks the subject/grade match for one of the `matchRequestIds` | "Selected tutor is not eligible for this assignment" |

**Implemented in:** `src/controllers/adminMatching.controller.ts → manualAssign` · `src/services/adminMatching.service.ts → manuallyAssignTutor, manuallyAssembleGroup`

---

#### POST /format-switch

**Purpose:** Student-initiated format change — immediately cancels the current match and spawns a fresh matching cycle under the new format (UC-61, FR-SP-045–049).

**Auth:** Student|Parent

**Request body:**
```json
{
  "studentId": "uuid, required for Parent",
  "toFormat": "string, required — ONE_TO_ONE | ONE_TO_THREE | ONE_TO_FIVE"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "formatSwitchRequestId": "uuid",
    "fromFormat": "ONE_TO_ONE",
    "toFormat": "ONE_TO_THREE",
    "oldMembershipStatus": "ENDED",
    "newMatchRequestId": "uuid",
    "refundId": "uuid|null"
  }
}
```
`refundId` is `null` only if there were zero remaining paid sessions in the current billing cycle to prorate; otherwise it references the `Refund` created via the sessions-delivered formula (Section 13, FR-PB-007), payable through the Payments & Earnings feature. The rest of an old group-format cohort, if any, is left intact and unaffected (UC-61 alternate flow) — this endpoint never touches other members' `CohortMembership` rows.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Caller has no active cohort membership to switch from | "No active assignment to switch from" |
| 400 | `toFormat` equals the current format | "You are already in this format" |

**Implemented in:** `src/controllers/formatSwitch.controller.ts → requestSwitch` · `src/services/formatSwitch.service.ts → requestSwitch` · `src/schemas/formatSwitch.schema.ts → requestFormatSwitchSchema`

---

**Next:** proceed to → [04. Class Delivery, Recording & Library API]
