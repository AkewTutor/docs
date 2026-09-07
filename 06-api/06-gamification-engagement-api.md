## Project: AKEWTutor — API Specification
**Feature:** Gamification & Engagement
**Conventions:** see 0.1–0.7 in `00-api-conventions.md`.

**Owns:** XPLedgerEntry, Badge, StudentBadge, TutorBadge, Streak, Challenge, ChallengeProgress. **Depends on:** Accounts & Guardianship (hard, per Feature Decomposition §1.1); soft-integrates with Class Delivery & Library (XP-award trigger on class-attended — no FK, no direct endpoint dependency).

**Links back to:** [00. API Conventions], [05a. Backend Folder & File Structure §6], [Feature Decomposition §1]
**Links forward to:** [8-6. Backend Function-Level Spec: Gamification & Engagement]

---

### 6.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /gamification/xp/me | Student\|Parent | UC-63 | FR-SP-039 |
| GET | /gamification/leaderboard | Student\|Parent | UC-64 | FR-GA-002 |
| GET | /gamification/badges/me | Student\|Parent | UC-63 | FR-GA-003 |
| GET | /admin/badges | Admin | UC-77 | FR-AD-004 |
| POST | /admin/badges | Admin | UC-77 | FR-AD-004, FR-GA-005 — **I2 fix** |
| PATCH | /admin/badges/:badgeId | Admin | UC-77 | FR-AD-004, FR-GA-005 |
| GET | /gamification/challenges | Student\|Parent | UC-65 | FR-SP-040, FR-GA-004 |
| GET | /gamification/challenges/me | Student\|Parent | UC-65 | FR-SP-040, FR-GA-004 |
| POST | /admin/challenges | Admin | UC-65 | FR-GA-004 |
| POST | /admin/students/:studentId/xp-adjustments | Admin | UC-88 | FR-AD-018 — **I2 fix** |

---

### 6.2 Endpoint Detail

#### GET /gamification/xp/me

**Purpose:** The caller's (or, for a Parent, their child's) XP total, streak, and recent ledger activity (UC-63, FR-SP-039, FR-SP-014).

**Auth:** Student|Parent

**Query params:**
```
?studentId=uuid (required for Parent; ignored/forbidden for Student — a Student always gets their own data)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "totalXP": 1240,
    "streak": {
      "currentStreakDays": 5,
      "longestStreakDays": 12,
      "lastActivityDate": "2026-09-06"
    },
    "recentEntries": [
      { "amount": 20, "reason": "CLASS_ATTENDED", "createdAt": "2026-09-06T14:00:00Z" }
    ]
  }
}
```
A streak broken by an inactive period resets `currentStreakDays` without deleting previously earned badges/XP (UC-63 alternate flow) — `totalXP` is never reduced by a streak reset.

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/xp.controller.ts → getMyProgress` · `src/services/xp.service.ts → awardXP` (write path), read via ledger sum

---

#### GET /gamification/leaderboard

**Purpose:** Per-grade leaderboard, weekly and monthly (UC-64, FR-GA-002). Computed live from the XP ledger, never a stored/denormalized table (Doc 04 §4.0 design decision).

**Auth:** Student|Parent

**Query params:**
```
?studentId=uuid (required for Parent)
&period=WEEKLY|MONTHLY, required
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "grade": 8,
    "period": "WEEKLY",
    "rankings": [
      { "rank": 1, "displayName": "Bethel M.", "xp": 340 },
      { "rank": 2, "displayName": "Yonas T.", "xp": 310 }
    ],
    "callerRank": 2
  }
}
```
`displayName` is always first name + last-initial — never a full last name (FR-GA-002, Section 10 Definition of Done #2). A Grade 8 student's `rankings` never includes a student from any other grade, by design (Section 10 Definition of Done #1) — this is not filtered client-side, the query itself is grade-scoped.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/xp.controller.ts → getLeaderboard` · `src/services/xp.service.ts → getLeaderboard`

---

#### GET /gamification/badges/me

**Purpose:** The caller's (or, for a Parent, their child's) earned badges (UC-63, FR-GA-003, FR-SP-014).

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
    "badges": [
      {
        "badgeId": "uuid",
        "name": "5-Session Streak",
        "description": "Attended 5 consecutive scheduled sessions",
        "earnedAt": "2026-08-20T10:00:00Z"
      }
    ]
  }
}
```

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/badge.controller.ts → listMyBadges` · `src/services/badge.service.ts → awardStudentBadge` (write path)

---

#### GET /admin/badges

**Purpose:** List all badge definitions and award history, for review/adjustment (UC-77, FR-AD-004).

**Auth:** Admin

**Query params:**
```
?category=STUDENT|TUTOR&page=1&limit=20
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "badges": [
      {
        "id": "uuid",
        "name": "5-Session Streak",
        "category": "STUDENT",
        "criteriaDescription": "Attended 5 consecutive scheduled sessions",
        "isActive": true
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
No `rating`-derived field exists on any badge here — `criteriaDescription` is always experience/performance/achievement-based (Doc 04 Badge notes, FC-01).

**Error responses:** none.

**Implemented in:** `src/controllers/badge.controller.ts → adminListAll` · `src/services/badge.service.ts → adminManageBadges`

---

#### POST /admin/badges — **I2 fix**

**Purpose:** Create a new badge definition (UC-77, FR-AD-004, FR-GA-005; Doc 02 §10 v3.2 V1 Badge List callout — "Admin can add more later"). Previously only `PATCH /admin/badges/:badgeId` existed, which requires an existing badge and cannot create one; this closes that gap.

**Auth:** Admin

**Request body:**
```json
{
  "name": "string, required",
  "description": "string, required",
  "category": "string, required — STUDENT | TUTOR",
  "criteriaDescription": "string, required",
  "isActive": "boolean, optional, default true"
}
```
No `rating`-derived field is accepted — `criteriaDescription` must be experience/performance/achievement-based (Doc 04 Badge notes, FC-01); this is documentation guidance for the Admin filling the form, not a server-side content check.

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "name": "Quarter Champion",
    "category": "STUDENT",
    "criteriaDescription": "Reach a 90-day streak",
    "isActive": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `name`/`description`/`criteriaDescription` missing, or `category` not `STUDENT`\|`TUTOR` | "Invalid badge definition" |

**Implemented in:** `src/controllers/badge.controller.ts → adminCreate` · `src/services/badge.service.ts → createBadge` · `src/schemas/badge.schema.ts → createBadgeSchema`

---

#### PATCH /admin/badges/:badgeId

**Purpose:** Adjust a badge's criteria or active status (UC-77).

**Auth:** Admin

**Path params:** `badgeId` — Badge UUID

**Request body:**
```json
{
  "criteriaDescription": "string, optional",
  "isActive": "boolean, optional"
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

**Implemented in:** `src/controllers/badge.controller.ts → adminAdjust` · `src/services/badge.service.ts → adminManageBadges`

---

#### GET /gamification/challenges

**Purpose:** List active weekly/monthly challenges (UC-65, FR-SP-040, FR-GA-004).

**Auth:** Student|Parent — **H3-consistency fix:** opened to Parent alongside `/gamification/challenges/me`, since the two are always consumed together by `ChallengesPage` (Doc 07 §6.5) and this endpoint carries no student-specific or otherwise sensitive data (just the list of currently-running challenges), so there is no reason to block the one call that would otherwise strand the Parent-facing page half-rendered.

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "challenges": [
      {
        "id": "uuid",
        "title": "Complete 3 assessments this week",
        "period": "WEEKLY",
        "startsAt": "2026-09-01T00:00:00Z",
        "endsAt": "2026-09-07T23:59:59Z",
        "targetValue": 3
      }
    ]
  }
}
```

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/challenge.controller.ts → listActive` · `src/services/challenge.service.ts → listActiveChallenges`

---

#### GET /gamification/challenges/me

**Purpose:** The caller's (or, for a Parent, their child's) progress toward each active challenge (UC-65, FR-SP-014).

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
    "progress": [
      { "challengeId": "uuid", "progressValue": 2, "completedAt": null }
    ]
  }
}
```

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/challenge.controller.ts → getMyProgress` · `src/services/challenge.service.ts → trackProgress`

---

#### POST /admin/challenges

**Purpose:** Create a new weekly/monthly challenge (UC-65, FR-GA-004).

**Auth:** Admin

**Request body:**
```json
{
  "title": "string, required",
  "description": "string, required",
  "period": "string, required — WEEKLY | MONTHLY",
  "startsAt": "ISO 8601 datetime, required",
  "endsAt": "ISO 8601 datetime, required",
  "targetValue": "integer, required, > 0"
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
    "title": "Complete 3 assessments this week",
    "period": "WEEKLY"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `endsAt` not after `startsAt` | "End time must be after start time" |

**Implemented in:** `src/controllers/challenge.controller.ts → adminCreate` · `src/services/challenge.service.ts → createChallenge` · `src/schemas/challenge.schema.ts → createChallengeSchema`

---

#### POST /admin/students/:studentId/xp-adjustments — **I2 fix**

**Purpose:** Manually adjust a student's XP total (UC-88, FR-AD-018; Doc 02 §10 v3.2 XP Point Values callout — `XPReason.OTHER`, "the only reason where `amount` is caller-supplied"). This is the sole mechanism for correcting the leaderboard, since the leaderboard is a derived, non-stored `SUM(amount)` view over `XPLedgerEntry` (Doc 04 §4.0) with no separate ranking table to edit directly — a positive adjustment (goodwill) or negative adjustment (correcting an error, e.g. an erroneously-awarded `CLASS_ATTENDED` XP) is written as an ordinary `XPLedgerEntry` row with `reason: OTHER`, so it flows into the leaderboard/badge/streak logic exactly the same way any system-triggered award does.

**Auth:** Admin

**Path params:** `studentId` — StudentProfile UUID

**Request body:**
```json
{
  "amount": "integer, required — positive or negative, non-zero",
  "note": "string, required, 1–500 chars"
}
```
`note` is required here even though `XPLedgerEntry.note` is nullable at the schema level (Doc 04) — every other `XPReason` is self-explanatory from its constant trigger, but `OTHER` has no such context, so this endpoint enforces the audit trail the schema only optionally supports.

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "studentId": "uuid",
    "amount": -20,
    "reason": "OTHER",
    "note": "Reversing an erroneous CLASS_ATTENDED award from session #4821",
    "createdAt": "2026-09-06T15:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `amount` is zero, missing, or non-integer | "amount must be a non-zero integer" |
| 400 | `note` missing or empty | "A note is required for a manual XP adjustment" |
| 404 | No `StudentProfile` matches `studentId` | "Student not found" |

**Implemented in:** `src/controllers/xp.controller.ts → adminAdjust` · `src/services/xp.service.ts → adminAdjustXP` · `src/schemas/xp.schema.ts → adjustXPSchema`

---

**Next:** proceed to → [07. Payments & Earnings API]
