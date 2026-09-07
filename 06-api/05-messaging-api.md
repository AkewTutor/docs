## Project: AKEWTutor — API Specification
**Feature:** In-Platform Messaging
**Conventions:** see 0.1–0.7 in `00-api-conventions.md`. See also 0.4 — thread archiving (90 days post-assignment) is job-driven, exposed here only as `status: "ARCHIVED"`.

**Owns:** MessageThread, Message. **Depends on:** Matching & Cohorts (hard, per Feature Decomposition §1.1) — a `MessageThread` is 1:1 with a `Cohort`.

**Links back to:** [00. API Conventions], [05a. Backend Folder & File Structure §5], [Feature Decomposition §1]
**Links forward to:** [8-5. Backend Function-Level Spec: In-Platform Messaging]

---

### 5.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /messaging/cohorts/:cohortId/thread | Student\|Parent\|Tutor | UC-57, UC-58 | FR-MS-001, FR-MS-002 |
| GET | /messaging/cohorts/:cohortId/messages | Student\|Parent\|Tutor | UC-57, UC-58 | FR-MS-001, FR-MS-003 |
| POST | /messaging/cohorts/:cohortId/messages | Student\|Parent\|Tutor | UC-57, UC-58 | FR-MS-001, FR-MS-002, FR-SC-004 |
| GET | /admin/messaging/threads/:threadId | Admin | UC-60 | FR-MS-004, FR-AD-017 |
| POST | /admin/messaging/threads/:threadId/close | Admin | UC-60 | FR-MS-004 |

---

### 5.2 Endpoint Detail

#### GET /messaging/cohorts/:cohortId/thread

**Purpose:** Retrieve thread metadata for a cohort — a private pair thread for 1-to-1, a single shared thread for 1-to-3/1-to-5 (UC-57, UC-58, FR-MS-001 as amended in v3.1).

**Auth:** Student|Parent|Tutor — caller must be a currently active member/tutor of the cohort; a match not yet confirmed/paid, or one that has since ended, returns `403` (FR-MS-002).

**Path params:** `cohortId` — Cohort UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "cohortId": "uuid",
    "format": "ONE_TO_THREE",
    "status": "ACTIVE",
    "participantCount": 4
  }
}
```
`participantCount` includes the tutor plus every currently active `CohortMembership` on this cohort — for 1-to-1 this is always `2`. There are no private sub-threads within a group thread (UC-58).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Cohort not yet confirmed/paid, or caller no longer an active member | "Messaging is not available for this cohort" |

**Implemented in:** `src/controllers/messaging.controller.ts → getThread` · `src/services/messaging.service.ts → getThreadForCohort`

---

#### GET /messaging/cohorts/:cohortId/messages

**Purpose:** Load message history for a thread, chronologically (UC-57, UC-58, FR-MS-003).

**Auth:** Student|Parent|Tutor — same membership rule as above.

**Path params:** `cohortId` — Cohort UUID

**Query params:**
```
?page=1&limit=50
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "messages": [
      {
        "id": "uuid",
        "senderId": "uuid",
        "senderRole": "TUTOR",
        "body": "Running 5 minutes late, sorry!",
        "createdAt": "2026-09-08T15:56:00Z"
      }
    ],
    "page": 1,
    "limit": 50,
    "total": 1
  }
}
```
A thread archived 90 days after the cohort ended (FR-MS-003) still returns its history here — archiving removes it from the *active* thread list view on the client, it is not a deletion (UC-59).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Cohort not yet confirmed/paid, or caller no longer an active member | "Messaging is not available for this cohort" |

**Implemented in:** `src/controllers/messaging.controller.ts → listMessages` · `src/services/messaging.service.ts → listMessages`

---

#### POST /messaging/cohorts/:cohortId/messages

**Purpose:** Send a text-only message into the cohort's thread (UC-57, UC-58, FR-MS-001/002, FR-SC-004).

**Auth:** Student|Parent|Tutor — caller must be a currently active member/tutor; blocked otherwise (FR-MS-002).

**Rate limited:** 30 / minute, keyed by account — see `00-api-conventions.md` §0.8.

**Path params:** `cohortId` — Cohort UUID

**Request body:**
```json
{
  "body": "string, required, 1-2000 chars, text-only — no attachment/media fields exist"
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
    "senderId": "uuid",
    "body": "Running 5 minutes late, sorry!",
    "createdAt": "2026-09-08T15:56:00Z"
  }
}
```
Sending a message writes a `Notification` (`type: NEW_MESSAGE`) to every other active participant, through the same pipeline as any other notification (FR-NO-011, UC-93).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Cohort not yet confirmed/paid, or caller no longer an active member | "Messaging is not available for this cohort" |
| 403 | Thread `status: CLOSED_BY_ADMIN` | "This conversation has been closed" |
| 429 | Rate limit exceeded | "Too many requests, please try again later" |

**Implemented in:** `src/controllers/messaging.controller.ts → sendMessage` · `src/services/messaging.service.ts → sendMessage` · `src/schemas/messaging.schema.ts → sendMessageSchema`

---

#### GET /admin/messaging/threads/:threadId

**Purpose:** Admin views a thread for dispute investigation (UC-60, FR-MS-004, FR-AD-017).

**Auth:** Admin

**Path params:** `threadId` — MessageThread UUID

**Query params:**
```
?page=1&limit=50
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "cohortId": "uuid",
    "status": "ACTIVE",
    "messages": [
      {
        "id": "uuid",
        "senderId": "uuid",
        "body": "Running 5 minutes late, sorry!",
        "createdAt": "2026-09-08T15:56:00Z"
      }
    ],
    "page": 1,
    "limit": 50,
    "total": 1
  }
}
```

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/adminMessaging.controller.ts → viewThread` · `src/services/adminMessaging.service.ts → viewThreadForDispute`

---

#### POST /admin/messaging/threads/:threadId/close

**Purpose:** Close/report a thread that violates platform rules (UC-60, FR-MS-004).

**Auth:** Admin

**Path params:** `threadId` — MessageThread UUID

**Request body:**
```json
{
  "reason": "string, required — internal note"
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
    "status": "CLOSED_BY_ADMIN",
    "closedById": "uuid",
    "closedAt": "2026-09-06T15:00:00Z"
  }
}
```
Once closed, `POST /messaging/cohorts/:cohortId/messages` returns `403` for that cohort's thread (see above).

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/adminMessaging.controller.ts → closeThread` · `src/services/adminMessaging.service.ts → closeThread`

---

**Next:** proceed to → [06. Gamification & Engagement API]
