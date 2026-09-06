## Project: AKEWTutor — API Specification
**Feature:** Class Delivery, Recording & Library
**Conventions:** see 0.1–0.7 in `00-api-conventions.md`. See also 0.4 — the recording-missing escalation and the payment-pause reschedule are job-driven, exposed here only as read state.

**Owns:** ScheduledSession, RescheduleRequest, SessionMiss, RecordingConsent, Recording, LibraryMaterial, WeeklyAssessment. **Depends on:** Matching & Cohorts (hard, per Doc 07 §1.1) — every session belongs to a confirmed `Cohort`.

---

### 4.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /sessions | Student\|Parent\|Tutor | UC-14, UC-42, UC-44 | FR-CD-001, FR-CD-002, FR-SP-033 |
| GET | /sessions/:sessionId | Student\|Parent\|Tutor | UC-42, UC-44, UC-48 | FR-CD-001–009 |
| POST | /sessions/:sessionId/link | Tutor | UC-43 | FR-TU-013, FR-CD-003 |
| POST | /sessions/:sessionId/complete | Tutor | UC-44 | FR-CD-001 |
| GET | /recording-consent/status | Student\|Parent\|Tutor | UC-45 | FR-SC-008 |
| POST | /recording-consent/acknowledge | Student\|Parent\|Tutor | UC-45 | FR-SC-008, FR-SC-009 |
| POST | /recordings | Tutor | UC-45, UC-46 | FR-TU-014, FR-CD-004, FR-CD-005 |
| GET | /recordings/me | Student\|Parent | UC-47 | FR-SP-035, FR-SP-036 |
| GET | /recordings/:recordingId/signed-url | Student\|Parent | UC-47 | FR-SP-035, FR-SP-036 |
| POST | /recordings/:recordingId/keep-permanently | Student\|Parent | UC-49 | FR-SP-037 |
| GET | /admin/library/recording-compliance | Admin | UC-48, UC-84 | FR-AD-014, Section 9.1 |
| POST | /library/materials | Tutor | UC-72 | FR-TU-016 |
| GET | /library/cohorts/:cohortId/materials | Student\|Parent\|Tutor | UC-47, UC-72 | FR-CD-009 |
| PATCH | /admin/library/materials/:id | Admin | UC-84 | FR-AD-014 |
| POST | /reschedule | Student\|Parent\|Tutor | UC-53, UC-54 | FR-MK-004, FR-MK-006, FR-MK-007, FR-MK-008 |
| GET | /session-miss | Tutor\|Admin | UC-52 | FR-MK-003 |
| POST | /session-miss | Admin | UC-50, UC-51 | FR-MK-001, FR-MK-002 |
| POST | /assessments | Tutor | UC-71 | FR-TU-017 |
| GET | /assessments/cohort-membership/:id | Student\|Parent\|Tutor | UC-62, UC-71 | FR-SP-038 |

---

### 4.2 Endpoint Detail

#### GET /sessions

**Purpose:** List the caller's scheduled sessions, past and upcoming (UC-14 dashboard tile, UC-42, UC-44).

**Auth:** Student|Parent|Tutor

**Query params:**
```
?studentId=uuid (required for Parent)
&status=SCHEDULED|COMPLETED|MISSED|RESCHEDULED|PAYMENT_PAUSE_RESCHEDULED|CANCELLED (optional)
&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "sessions": [
      {
        "id": "uuid",
        "cohortId": "uuid",
        "scheduledStart": "2026-09-08T16:00:00Z",
        "scheduledEnd": "2026-09-08T17:00:00Z",
        "status": "SCHEDULED",
        "isMakeup": false,
        "recordingStatus": "PENDING"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
No sessions yet (student still matching) returns `sessions: []` — not an error.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/session.controller.ts → listMySessions` · `src/services/session.service.ts → generateSessionsForCohort` (session generation), read path via Prisma directly

---

#### GET /sessions/:sessionId

**Purpose:** Full detail for one session, including its recording-compliance state (UC-42, UC-44, UC-48).

**Auth:** Student|Parent|Tutor — caller must be a member/tutor of the session's cohort.

**Path params:** `sessionId` — ScheduledSession UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "cohortId": "uuid",
    "scheduledStart": "2026-09-08T16:00:00Z",
    "scheduledEnd": "2026-09-08T17:00:00Z",
    "jitsiLinkUrl": "https://meet.jit.si/akewtutor-xyz",
    "jitsiLinkSentAt": "2026-09-08T15:25:00Z",
    "status": "SCHEDULED",
    "isMakeup": false,
    "makeupForSessionId": null,
    "recordingStatus": "PENDING"
  }
}
```
`status: "PAYMENT_PAUSE_RESCHEDULED"` is a possible value here reflecting the job-driven pause-reschedule flow (0.4) — this endpoint never triggers that state, only reports it.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller is not a member/tutor of this session's cohort | "Not authorized to view this session" |

**Implemented in:** `src/controllers/session.controller.ts → getSession` · `src/services/session.service.ts`

---

#### POST /sessions/:sessionId/link

**Purpose:** Tutor provides the Jitsi session link, required at least 30 minutes before class (UC-43, FR-TU-013, FR-CD-003).

**Auth:** Tutor — must be the session's assigned tutor.

**Path params:** `sessionId` — ScheduledSession UUID

**Request body:**
```json
{
  "jitsiLinkUrl": "string, required, valid URL"
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
    "jitsiLinkUrl": "https://meet.jit.si/akewtutor-xyz",
    "jitsiLinkSentAt": "2026-09-08T15:25:00Z",
    "providedLateNotice": false
  }
}
```
`providedLateNotice: true` if submitted with less than 30 minutes remaining before `scheduledStart` — this does not block submission (the student still needs the link), but is a flag surfaced for tutor-caused-miss evaluation (Section H, UC-50) if the class ends up disrupted as a result.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller is not this session's assigned tutor | "Not authorized to provide a link for this session" |

**Implemented in:** `src/controllers/session.controller.ts → provideLink` · `src/services/session.service.ts → provideJitsiLink` · `src/schemas/session.schema.ts → provideJitsiLinkSchema`

---

#### POST /sessions/:sessionId/complete

**Purpose:** Mark a session as delivered (UC-44).

**Auth:** Tutor — must be the session's assigned tutor.

**Path params:** `sessionId` — ScheduledSession UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "status": "COMPLETED"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Session already `COMPLETED`, `MISSED`, or `CANCELLED` | "This session's status cannot be changed" |

**Implemented in:** `src/controllers/session.controller.ts` (co-located) · `src/services/session.service.ts → markCompleted`

---

#### GET /recording-consent/status

**Purpose:** Check whether both parties have acknowledged recording consent for a given pairing (UC-45, FR-SC-008).

**Auth:** Student|Parent|Tutor

**Query params:**
```
?tutorId=uuid, required
&studentId=uuid, required
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "tutorId": "uuid",
    "studentId": "uuid",
    "tutorAcknowledgedAt": "2026-09-01T10:00:00Z",
    "studentOrParentAcknowledgedAt": null,
    "consentComplete": false
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/recordingConsent.controller.ts → getStatus` · `src/services/recordingConsent.service.ts → getConsentStatus`

---

#### POST /recording-consent/acknowledge

**Purpose:** Record a one-time consent acknowledgment for a tutor–student pairing (UC-45, FR-SC-008/009).

**Auth:** Student|Parent|Tutor

**Request body:**
```json
{
  "tutorId": "uuid, required",
  "studentId": "uuid, required"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "tutorId": "uuid",
    "studentId": "uuid",
    "acknowledgedByUserId": "uuid",
    "consentComplete": true
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/recordingConsent.controller.ts → acknowledge` · `src/services/recordingConsent.service.ts → acknowledgeAsStudentOrParent, acknowledgeAsTutor` · `src/schemas/recordingConsent.schema.ts → acknowledgeConsentSchema`

---

#### POST /recordings

**Purpose:** Tutor uploads a session recording (UC-45, UC-46, FR-TU-014, FR-CD-004/005). Blocked at the service layer if consent (per pairing) is not yet complete on both sides (Section 14 Definition of Done #1).

**Auth:** Tutor — must be the session's assigned tutor.

**Request body (multipart):**
```
sessionId: uuid, required
file: binary, required — video file
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "sessionId": "uuid",
    "storageKey": "recordings/2026/09/uuid.mp4",
    "encoding": "720p",
    "expiresAt": "2026-12-06T00:00:00Z",
    "keepPermanently": false
  }
}
```
`expiresAt` defaults to 90 days from upload. Upload automatically routes the recording to the correct student's Library — no manual filing step.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Consent not yet complete for this pairing (first session) | "Recording consent must be acknowledged by both parties before this session can be recorded" |
| 403 | Caller is not the session's assigned tutor | "Not authorized to upload a recording for this session" |

**Implemented in:** `src/controllers/recording.controller.ts → upload` · `src/services/recording.service.ts → uploadRecording` · `src/utils/providers/storage.client.ts`

---

#### GET /recordings/me

**Purpose:** List the caller's own class recordings and materials (UC-47, FR-SP-035–037).

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
    "recordings": [
      {
        "id": "uuid",
        "sessionId": "uuid",
        "createdAt": "2026-09-01T17:00:00Z",
        "expiresAt": "2026-11-30T17:00:00Z",
        "keepPermanently": false
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
A recording past `expiresAt` and not `keepPermanently` is excluded from this list entirely (expected expiry, not an error — UC-47 alternate flow).

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/recording.controller.ts → getMyRecordings` · `src/services/recording.service.ts`

---

#### GET /recordings/:recordingId/signed-url

**Purpose:** Generate a short-lived signed URL to play back a recording (UC-47, FR-SP-035/036). Access is scoped strictly to the student's own recordings — never another student's, even via a guessed ID.

**Auth:** Student|Parent — caller must have (or have had) a `CohortMembership` on this recording's session's `Cohort`.

**Path params:** `recordingId` — Recording UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "signedUrl": "https://r2.akewtutor.com/...?signature=...",
    "expiresIn": 900
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller was never a member of this recording's cohort | "Not authorized to access this recording" |
| 404 | Recording past retention and not `keepPermanently`, or `deletedAt` set | "Recording no longer available" |

**Implemented in:** `src/controllers/recording.controller.ts → getSignedUrl` · `src/services/recording.service.ts → getSignedUrl`

---

#### POST /recordings/:recordingId/keep-permanently

**Purpose:** Bypass the default 90-day auto-deletion for a specific recording (UC-49, FR-SP-037).

**Auth:** Student|Parent — same ownership rule as above.

**Path params:** `recordingId` — Recording UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "keepPermanently": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller was never a member of this recording's cohort | "Not authorized to modify this recording" |

**Implemented in:** `src/controllers/recording.controller.ts → keepPermanently` · `src/services/recording.service.ts → keepPermanently`

---

#### GET /admin/library/recording-compliance

**Purpose:** Admin's "recording missing"/escalated queue (UC-48, UC-84, FR-AD-014, Section 9.1). Reflects `recordingMissingCheck.job.ts` state per 0.4.

**Auth:** Admin

**Query params:**
```
?status=MISSING|ESCALATED&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "sessions": [
      {
        "sessionId": "uuid",
        "cohortId": "uuid",
        "tutorId": "uuid",
        "scheduledEnd": "2026-09-06T13:00:00Z",
        "recordingStatus": "MISSING"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/library.controller.ts → adminOverride` (co-located admin queue read) · `src/services/recording.service.ts → flagMissing, escalateMissing`

---

#### POST /library/materials

**Purpose:** Tutor uploads a PDF/note/book to a cohort's Library (UC-72, FR-TU-016, FR-CD-009).

**Auth:** Tutor — must be the cohort's assigned tutor.

**Request body (multipart):**
```
cohortId: uuid, required
title: string, required
fileType: string, required — PDF | NOTE | BOOK
file: binary, required
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "cohortId": "uuid",
    "title": "Algebra Practice Set 3",
    "fileType": "PDF",
    "fileUrl": "https://r2.akewtutor.com/..."
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Unsupported file type/format | "Unsupported file type" |
| 403 | Caller is not this cohort's assigned tutor | "Not authorized to upload to this cohort" |

**Implemented in:** `src/controllers/library.controller.ts → upload` · `src/services/library.service.ts → uploadMaterial` · `src/schemas/library.schema.ts → uploadMaterialSchema`

---

#### GET /library/cohorts/:cohortId/materials

**Purpose:** List a cohort's Library materials, alongside the student's own recordings (UC-47, UC-72).

**Auth:** Student|Parent|Tutor — caller must be a member/tutor of the cohort.

**Path params:** `cohortId` — Cohort UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "materials": [
      {
        "id": "uuid",
        "title": "Algebra Practice Set 3",
        "fileType": "PDF",
        "fileUrl": "https://r2.akewtutor.com/...",
        "createdAt": "2026-09-01T10:00:00Z"
      }
    ]
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller is not a member/tutor of this cohort | "Not authorized to view this cohort's materials" |

**Implemented in:** `src/controllers/library.controller.ts → listForCohort` · `src/services/library.service.ts → listCohortMaterials`

---

#### PATCH /admin/library/materials/:id

**Purpose:** Admin retention/access override on a Library item (UC-84, FR-AD-014).

**Auth:** Admin

**Path params:** `id` — LibraryMaterial UUID

**Request body:**
```json
{
  "title": "string, optional",
  "remove": "boolean, optional"
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
    "title": "Algebra Practice Set 3 (revised)"
  }
}
```

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/library.controller.ts → adminOverride` · `src/services/library.service.ts → adminManageLibrary`

---

#### POST /reschedule

**Purpose:** Either party requests moving a specific upcoming session's time (UC-53, UC-54, FR-MK-004/006/007/008). Classified server-side as `FREE_RESCHEDULE` (≥12h notice) or `SAME_DAY_MISS` (<12h notice).

**Auth:** Student|Parent|Tutor — caller must be a member/tutor of the session's cohort.

**Request body:**
```json
{
  "sessionId": "uuid, required",
  "requestedNewStart": "ISO 8601 datetime, required — must fall within the tutor's existing availability"
}
```

**Success response — 200 (free reschedule):**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "sessionId": "uuid",
    "requestedNewStart": "2026-09-09T16:00:00Z",
    "noticeHours": "18.0",
    "classification": "FREE_RESCHEDULE",
    "freeReschedulesUsedThisMonth": 1
  }
}
```

**Success response — 200 (same-day miss classification, <12h notice):**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "sessionId": "uuid",
    "requestedNewStart": "2026-09-08T18:00:00Z",
    "noticeHours": "3.5",
    "classification": "SAME_DAY_MISS",
    "sessionMissId": "uuid"
  }
}
```
A `SAME_DAY_MISS` classification writes a `SessionMiss` row immediately (UC-54), attributed to whichever party requested it, and is evaluated under the same tutor-caused/student-caused rules as any other miss (`GET /session-miss`).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `requestedNewStart` falls outside the tutor's existing availability | "Requested time is outside the tutor's availability" |
| 409 | Caller has already used 2 free reschedules this month and this request would qualify as free | "Free reschedule limit reached for this month — further changes require Admin review" |

**Implemented in:** `src/controllers/reschedule.controller.ts → requestReschedule` · `src/services/reschedule.service.ts → requestReschedule, enforceMonthlyCap` · `src/schemas/reschedule.schema.ts → requestRescheduleSchema`

---

#### GET /session-miss

**Purpose:** List recorded misses — a tutor viewing their own record, or Admin viewing the escalation list (UC-52, FR-MK-003).

**Auth:** Tutor (own record only)|Admin (any)

**Query params:**
```
?tutorId=uuid (Admin only, optional filter)
&causedBy=TUTOR|STUDENT (optional)
&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "misses": [
      {
        "id": "uuid",
        "sessionId": "uuid",
        "causedBy": "TUTOR",
        "missType": "NO_SHOW",
        "makeupSessionId": "uuid",
        "createdAt": "2026-09-01T16:05:00Z"
      }
    ],
    "escalationFlag": true,
    "page": 1,
    "limit": 20,
    "total": 2
  }
}
```
`escalationFlag: true` when this tutor has 2+ `causedBy: TUTOR` misses within a rolling 30-day window (FR-MK-003) — computed at query time, not a stored counter (Doc 04 SessionMiss notes).

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/sessionMiss.controller.ts → listMisses` · `src/services/sessionMiss.service.ts → checkTutorEscalation`

---

#### POST /session-miss

**Purpose:** Record a missed session and apply the correct fault-based consequence (UC-50 tutor-caused, UC-51 student-caused, FR-MK-001/002/009). Admin-triggered here as the system-of-record entry point (e.g. following a complaint or manual review); the same effect also results automatically from a `SAME_DAY_MISS` reschedule classification (`POST /reschedule`).

**Auth:** Admin

**Request body:**
```json
{
  "sessionId": "uuid, required",
  "causedBy": "string, required — TUTOR | STUDENT",
  "missType": "string, required — NO_SHOW | LATE_CANCELLATION | TECHNICAL_FAILURE"
}
```

**Success response — 201 (tutor-caused):**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "sessionId": "uuid",
    "causedBy": "TUTOR",
    "missType": "NO_SHOW",
    "makeupSessionId": "uuid",
    "makeupDeadline": "2026-09-13T00:00:00Z",
    "tutorEarningRateForMakeup": "REDUCED_MAKEUP"
  }
}
```
A mandatory free make-up session is auto-queued within 7 days; no refund/credit is issued for the missed session itself; the tutor is paid a flat 50% share for the make-up (FR-MK-009, credited via the Payments & Earnings feature once delivered).

**Success response — 201 (student-caused):**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "sessionId": "uuid",
    "causedBy": "STUDENT",
    "missType": "NO_SHOW",
    "makeupSessionId": null,
    "tutorEarningRateForOriginalSession": "FULL"
  }
}
```
No make-up, no refund; the session counts as delivered against that month's billed sessions; the tutor is paid their normal full share.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | A `SessionMiss` already exists for this `sessionId` | "A miss has already been recorded for this session" |

**Implemented in:** `src/controllers/sessionMiss.controller.ts → reportMiss` · `src/services/sessionMiss.service.ts → recordTutorCausedMiss, recordStudentCausedMiss`

---

#### POST /assessments

**Purpose:** Tutor submits weekly feedback/assessment for a student (UC-71, FR-TU-017).

**Auth:** Tutor — must be the submitting tutor for this `cohortMembershipId`.

**Request body:**
```json
{
  "cohortMembershipId": "uuid, required",
  "weekStartDate": "ISO 8601 date, required",
  "scoreSummary": "string, optional",
  "tutorFeedback": "string, required"
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
    "cohortMembershipId": "uuid",
    "weekStartDate": "2026-09-01",
    "tutorFeedback": "Strong improvement on quadratic equations this week."
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | An assessment already exists for this `cohortMembershipId` + `weekStartDate` | "An assessment for this week has already been submitted" |

**Implemented in:** `src/controllers/weeklyAssessment.controller.ts → submit` · `src/services/weeklyAssessment.service.ts → submitAssessment` · `src/schemas/weeklyAssessment.schema.ts → submitAssessmentSchema`

---

#### GET /assessments/cohort-membership/:id

**Purpose:** View assessment history for one assignment (UC-62, UC-71).

**Auth:** Student|Parent|Tutor — caller must be the student/parent on this membership, or the submitting tutor.

**Path params:** `id` — CohortMembership UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "assessments": [
      {
        "id": "uuid",
        "weekStartDate": "2026-09-01",
        "scoreSummary": "82%",
        "tutorFeedback": "Strong improvement on quadratic equations this week.",
        "createdAt": "2026-09-06T09:00:00Z"
      }
    ]
  }
}
```
No assessment submitted yet for the current week returns the list without that week's entry — not an error (UC-62 alternate flow).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller is not party to this membership | "Not authorized to view these assessments" |

**Implemented in:** `src/controllers/weeklyAssessment.controller.ts → listForMembership` · `src/services/weeklyAssessment.service.ts → getAssessmentsForStudent`

---

**Next:** proceed to → [05. Messaging API]
