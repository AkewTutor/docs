## Project: AKEWTutor — API Specification
**Feature:** Accounts & Guardianship (Students, Parents, Tutors, Subjects, Availability)
**Conventions:** see 0.1–0.7 in `00-api-conventions.md`.

**Owns:** StudentProfile, ParentProfile, TutorProfile, ParentStudentRelationship, Subject, TutorSubjectRanking, AvailabilitySlot. **Depends on:** Shared Config (hard, per Doc 07 §1.1).

---

### 2.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /students/me/profile | Student\|Parent | UC-12, UC-13, UC-14 | FR-SP-006–010 |
| PATCH | /students/me/profile | Student\|Parent | UC-12 | FR-SP-006 |
| PATCH | /students/me/academic-profile | Student\|Parent | UC-13 | FR-SP-007, FR-SP-008, FR-SP-009, FR-SP-010 |
| POST | /guardianship/students | Parent | UC-04 | FR-AC-002, FR-AC-003 |
| POST | /guardianship/invites/:relationshipId/resend | Parent | UC-04 | FR-AC-003 |
| POST | /guardianship/invites/:token/activate | Student | UC-05 | FR-AC-003, FR-AC-004 |
| POST | /guardianship/guardian-invites | Student | UC-07 | FR-AC-005, FR-AC-006 |
| GET | /guardianship/relationships | Student\|Parent | UC-04, UC-08, UC-09 | FR-AC-006 |
| PATCH | /guardianship/relationships/:id/revoke | Student\|Parent\|Admin | UC-08, UC-09 | FR-AC-007, FR-AC-008 |
| GET | /tutors/me/profile | Tutor | UC-16 | FR-TU-003 |
| PATCH | /tutors/me/profile | Tutor | UC-16 | FR-TU-003 |
| PUT | /tutors/me/subjects | Tutor | UC-17 | FR-TU-006, FR-TU-007, FR-TU-008 |
| GET | /tutors/me/availability | Tutor | UC-19 | FR-TU-009 |
| POST | /tutors/me/availability | Tutor | UC-19 | FR-TU-009 |
| DELETE | /tutors/me/availability/:slotId | Tutor | UC-19 | FR-TU-009 |
| GET | /subjects | Public | UC-79 | FR-AD-013 |
| POST | /admin/subjects | Admin | UC-79 | FR-AD-013 |
| PATCH | /admin/subjects/:id | Admin | UC-79 | FR-AD-013 |
| GET | /admin/tutors/pending | Admin | UC-18 | FR-TU-004, FR-AD-002 |
| POST | /admin/tutors/:tutorId/approve | Admin | UC-18 | FR-TU-004, FR-AD-002 |
| POST | /admin/tutors/:tutorId/reject | Admin | UC-18 | FR-TU-004, FR-AD-002 |
| GET | /admin/people | Admin | UC-76 | FR-AD-001 |
| PATCH | /admin/people/relationships/:id | Admin | UC-76, UC-08 | FR-AD-001, FR-AC-007 |
| POST | /admin/people/:userId/suspend | Admin | UC-78 | FR-AD-003 |

---

### 2.2 Endpoint Detail

#### GET /students/me/profile

**Purpose:** Retrieve the caller's own (or, for a Grades 1–5 parent, their linked student's) full profile, powering the dashboard (UC-14) and profile screens (UC-12, UC-13).

**Auth:** Student|Parent. For a Parent, the response is scoped to a specific `studentId` query param, which must correspond to an `ACTIVE` relationship they hold.

**Query params (Parent only):**
```
?studentId=uuid, required for Parent, ignored for Student
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "grade": 8,
    "school": "Bole Community School",
    "profilePictureUrl": "https://...",
    "subjectsOfInterest": ["subject-uuid-1"],
    "academicLevel": "Intermediate",
    "learningGoals": "Improve algebra grades before finals",
    "preferredLanguage": "Amharic",
    "learningSchedulePreference": { "days": ["Mon", "Wed"], "timeOfDay": "evening" },
    "teachingStylePreference": "Visual, example-driven",
    "budgetPreference": "300.00",
    "formatPreference": "ONE_TO_ONE",
    "accountStatus": "ACTIVE"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Parent supplies a `studentId` they have no `ACTIVE` relationship to | "Not authorized to view this student's profile" |

**Implemented in:** `src/controllers/studentProfile.controller.ts → getMyProfile` · `src/services/studentProfile.service.ts → getProfile`

---

#### PATCH /students/me/profile

**Purpose:** Edit basic profile fields — name, profile picture (UC-12).

**Auth:** Student|Parent (same scoping rule as above)

**Request body:**
```json
{
  "studentId": "uuid, required for Parent",
  "profilePictureUrl": "string, optional"
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
    "profilePictureUrl": "https://..."
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Invalid image format/size | "Unsupported image format or file too large" |

**Implemented in:** `src/controllers/studentProfile.controller.ts → updateBasicProfile` · `src/services/studentProfile.service.ts → updateBasicProfile`

---

#### PATCH /students/me/academic-profile

**Purpose:** Set or edit the academic profile fields the matching engine depends on (UC-13) — grade, school, subjects, academic level, learning goals, language, schedule, teaching style, budget, and format preference (FR-SP-007–010).

**Auth:** Student|Parent (same scoping rule as above)

**Request body:**
```json
{
  "studentId": "uuid, required for Parent",
  "grade": "integer, optional, 1-12",
  "school": "string, optional",
  "subjectsOfInterest": "array of subject UUIDs, optional",
  "academicLevel": "string, optional",
  "learningGoals": "string, optional",
  "preferredLanguage": "string, optional — hard filter, all formats",
  "learningSchedulePreference": "object, optional",
  "teachingStylePreference": "string, optional — soft factor, 1-to-1 only",
  "budgetPreference": "decimal string, optional — hard filter, 1-to-1 search only",
  "formatPreference": "string, optional — ONE_TO_ONE | ONE_TO_THREE | ONE_TO_FIVE"
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
    "grade": 8,
    "preferredLanguage": "Amharic",
    "budgetPreference": "300.00",
    "formatPreference": "ONE_TO_ONE"
  }
}
```
Setting `formatPreference` here does not itself create a `MatchRequest` — that happens explicitly via the Matching & Cohorts endpoints (`GET /matching/tutors/search`, `POST /matching/group-format`, etc.), consistent with UC-13's postcondition ("the profile contains everything the matching engine needs") vs. UC-20/UC-28 ("student searches" / "is auto-matched") being separate steps.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Required matching fields (`grade`, at least one subject, `formatPreference`) still unset when the caller attempts to proceed to matching | Enforced at the matching endpoints, not here — this endpoint accepts partial updates |

**Implemented in:** `src/controllers/studentProfile.controller.ts → updateAcademicProfile` · `src/services/studentProfile.service.ts → updateAcademicProfile` · `src/schemas/studentProfile.schema.ts → updateAcademicProfileSchema`

---

#### POST /guardianship/students

**Purpose:** Parent adds a student and sends an activation invite (UC-04). Entering the grade here is the field that determines Grades 1–5 vs. 6–12 routing (FR-AC-002).

**Auth:** Parent

**Request body:**
```json
{
  "grade": "integer, required, 1-12",
  "inviteContact": "string, required — email or phone to send the invite to"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "relationshipId": "uuid",
    "studentRecordStatus": "INVITED",
    "relationshipType": "MANDATORY_GUARDIAN",
    "inviteExpiresAt": "2026-09-20T15:00:00Z"
  }
}
```
`relationshipType` is `MANDATORY_GUARDIAN` for `grade` 1–5. If `grade` is 6–12, the system rejects this endpoint — see error table below, since the 6–12 path is student-initiated (`POST /auth/register/student` + optional `POST /guardianship/guardian-invites`), not parent-initiated.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `grade` is 6-12 | "Grades 6–12 students register independently — see /auth/register/student" |

**Implemented in:** `src/controllers/guardianship.controller.ts → addStudent` · `src/services/guardianship.service.ts → addStudentAndInvite` · `src/schemas/guardianship.schema.ts → addStudentSchema`

---

#### POST /guardianship/invites/:relationshipId/resend

**Purpose:** Resend or regenerate a pending invite, restarting the 14-day window (UC-04 alternate flow, FR-AC-003).

**Auth:** Parent — must own the `relationshipId`

**Path params:** `relationshipId` — ParentStudentRelationship UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "relationshipId": "uuid",
    "inviteExpiresAt": "2026-09-27T15:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Relationship doesn't belong to the caller | "Not authorized to manage this invite" |
| 409 | Relationship status is already `ACTIVE` | "This student has already activated their account" |

**Implemented in:** `src/controllers/guardianship.controller.ts → resendInvite` · `src/services/guardianship.service.ts → resendOrRegenerateInvite`

---

#### POST /guardianship/invites/:token/activate

**Purpose:** Student completes registration via a parent-issued invite (UC-05, FR-AC-003, FR-AC-004).

**Auth:** Public (the invite token itself is the credential up to this point; the response returns a session)

**Path params:** `token` — the invite token

**Request body:**
```json
{
  "password": "string, required, min 8 chars",
  "termsAccepted": "boolean, required, must be true"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "accessToken": "jwt-string",
    "studentId": "uuid",
    "relationshipStatus": "ACTIVE"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Token expired | "This invite is no longer valid — ask your parent/guardian to resend it" |
| 404 | Token invalid/unknown | "Invite not found" |

**Implemented in:** `src/controllers/guardianship.controller.ts → activateInvite` · `src/services/guardianship.service.ts → activateInvite`

---

#### POST /guardianship/guardian-invites

**Purpose:** A Grades 6–12 student invites an optional guardian (UC-07, FR-AC-005/006). Does not gate the student's own access at any point — purely additive.

**Auth:** Student (Grades 6–12 only; enforced via `StudentProfile.grade`)

**Request body:**
```json
{
  "inviteContact": "string, required — email or phone"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "relationshipId": "uuid",
    "relationshipType": "OPTIONAL_GUARDIAN",
    "status": "INVITED",
    "inviteExpiresAt": "2026-09-20T15:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Caller's `grade` is 1–5 | "This action is only available to Grade 6–12 students" |

**Implemented in:** `src/controllers/guardianship.controller.ts → inviteGuardian` · `src/services/guardianship.service.ts → inviteOptionalGuardian`

---

#### GET /guardianship/relationships

**Purpose:** List the caller's parent–student relationships, regardless of role (UC-04, UC-08, UC-09).

**Auth:** Student|Parent

**Query params:**
```
?status=ACTIVE (optional filter: INVITED | ACTIVE | REVOKED)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "relationships": [
      {
        "id": "uuid",
        "parentId": "uuid",
        "studentId": "uuid",
        "relationshipType": "MANDATORY_GUARDIAN",
        "status": "ACTIVE",
        "permissions": {},
        "activatedAt": "2026-08-15T10:00:00Z"
      }
    ]
  }
}
```
A caller with no relationships (e.g. an independent Grade 6–12 student who never invited a guardian) returns `relationships: []` — not an error.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/guardianship.controller.ts` (list handler co-located with `activateInvite`/`revokeRelationship`) · `src/services/guardianship.service.ts`

---

#### PATCH /guardianship/relationships/:id/revoke

**Purpose:** Revoke or modify a relationship's permissions (UC-08), or process the sole-guardian removal path into the `GUARDIAN_REQUIRED_HOLD` state (UC-09, FR-AC-007, FR-AC-008).

**Auth:** Student|Parent|Admin — a Grade 1–5 student cannot revoke a `MANDATORY_GUARDIAN` relationship themselves (rejected with `403`); a Grade 6–12 student may revoke an `OPTIONAL_GUARDIAN` relationship they initiated; a Parent may revoke any relationship they're party to; Admin may revoke any relationship for disputes/abuse.

**Path params:** `id` — ParentStudentRelationship UUID

**Request body:**
```json
{
  "permissions": "object, optional — new permission set, if modifying rather than revoking",
  "revoke": "boolean, optional, default false"
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
    "status": "REVOKED",
    "revokedAt": "2026-09-06T15:00:00Z",
    "studentAccountStatus": "GUARDIAN_REQUIRED_HOLD"
  }
}
```
`studentAccountStatus` only appears (and only ever becomes `GUARDIAN_REQUIRED_HOLD`) when the revoked relationship was the student's **sole** `MANDATORY_GUARDIAN` relationship — no data, progress, XP, or recordings are deleted (FR-AC-008). For any other revoke/modify case, this field is omitted.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | A Grade 1–5 student attempts to revoke their own mandatory guardian | "Only a guardian or Admin can remove this relationship" |
| 403 | Caller is a Grade 6–12 student attempting to revoke a relationship they did not initiate | "Not authorized to modify this relationship" |

**Implemented in:** `src/controllers/guardianship.controller.ts → revokeRelationship` · `src/services/guardianship.service.ts → revokeOrModifyRelationship, handleSoleGuardianRemoval` · `src/schemas/guardianship.schema.ts → revokeRelationshipSchema`

---

#### GET /tutors/me/profile

**Purpose:** Retrieve the caller's own tutor profile (UC-16).

**Auth:** Tutor

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "profilePictureUrl": "https://...",
    "bio": "5 years teaching high-school mathematics.",
    "experienceDescription": "5 years, private and group tutoring",
    "educationInstitution": "Addis Ababa University",
    "degree": "BSc Mathematics",
    "verificationStatus": "VERIFIED",
    "verifiedAt": "2026-07-01T10:00:00Z"
  }
}
```

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/tutorProfile.controller.ts → getMyProfile` · `src/services/tutorProfile.service.ts → getProfile`

---

#### PATCH /tutors/me/profile

**Purpose:** Edit qualification/experience fields (UC-16, FR-TU-003).

**Auth:** Tutor

**Request body:**
```json
{
  "profilePictureUrl": "string, optional",
  "bio": "string, optional",
  "experienceDescription": "string, optional",
  "educationInstitution": "string, optional",
  "degree": "string, optional"
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
    "experienceDescription": "6 years, private and group tutoring"
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/tutorProfile.controller.ts → updateProfile` · `src/services/tutorProfile.service.ts → updateProfile` · `src/schemas/tutorProfile.schema.ts → updateTutorProfileSchema`

---

#### PUT /tutors/me/subjects

**Purpose:** Set the tutor's ranked subjects, hard-capped at two (UC-17, FR-TU-006/007/008). A full replace (`PUT`, not `PATCH`), since rank order is meaningful and must be resolved atomically.

**Auth:** Tutor

**Request body:**
```json
{
  "subjects": [
    { "subjectId": "uuid", "rank": 1 },
    { "subjectId": "uuid", "rank": 2 }
  ]
}
```
`subjects` — array, required, 1–2 items, each `rank` unique within the array (`1` or `2`).

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "subjects": [
      { "subjectId": "uuid", "subjectName": "Mathematics", "rank": 1 },
      { "subjectId": "uuid", "subjectName": "Physics", "rank": 2 }
    ]
  }
}
```
Each ranked subject applies across the full Grade 1–12 span — no grade-range field exists to set here (Doc 02 §6.2, Doc 04 TutorSubjectRanking notes).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | More than 2 subjects submitted | "A tutor may rank a maximum of two subjects" |
| 400 | Same `subjectId` submitted twice, or duplicate `rank` values | "Each subject may be ranked once, and ranks must be unique" |

**Implemented in:** `src/controllers/tutorProfile.controller.ts → rankSubjects` · `src/services/tutorProfile.service.ts → rankSubjects` · `src/schemas/tutorProfile.schema.ts → rankSubjectsSchema`

---

#### GET /tutors/me/availability

**Purpose:** List the tutor's own availability slots (UC-19).

**Auth:** Tutor

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "slots": [
      {
        "id": "uuid",
        "dayOfWeek": 1,
        "startTime": "2026-01-05T16:00:00Z",
        "endTime": "2026-01-05T17:00:00Z",
        "isRecurring": true
      }
    ]
  }
}
```

**Error responses:** none beyond common auth.

**Implemented in:** `src/controllers/availability.controller.ts → listMyAvailability` · `src/services/availability.service.ts → listSlots`

---

#### POST /tutors/me/availability

**Purpose:** Add a recurring or one-off availability slot (UC-19, FR-TU-009).

**Auth:** Tutor

**Request body:**
```json
{
  "dayOfWeek": "integer, optional, 0-6 — required if isRecurring is true",
  "startTime": "ISO 8601 datetime, required",
  "endTime": "ISO 8601 datetime, required",
  "isRecurring": "boolean, required"
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
    "dayOfWeek": 1,
    "startTime": "2026-01-05T16:00:00Z",
    "endTime": "2026-01-05T17:00:00Z",
    "isRecurring": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `endTime` not after `startTime` | "End time must be after start time" |

**Implemented in:** `src/controllers/availability.controller.ts → addSlot` · `src/services/availability.service.ts → setSlots` · `src/schemas/availability.schema.ts → createSlotSchema`

---

#### DELETE /tutors/me/availability/:slotId

**Purpose:** Remove an availability slot (UC-19). Blocked if a confirmed `ScheduledSession` depends on it.

**Auth:** Tutor — must own the slot

**Path params:** `slotId` — AvailabilitySlot UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "deleted": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | A confirmed session currently depends on this slot | "This slot is in use by a confirmed session and cannot be removed until it is resolved" |
| 403 | Slot belongs to a different tutor | "Not authorized to remove this slot" |

**Implemented in:** `src/controllers/availability.controller.ts → removeSlot` · `src/services/availability.service.ts → removeSlot` · `src/schemas/availability.schema.ts → deleteSlotSchema`

---

#### GET /subjects

**Purpose:** List active subjects for use in profile/search forms (public catalog read).

**Auth:** Public

**Query params:**
```
?includeInactive=false (Admin only, ignored for Public)
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "subjects": [
      { "id": "uuid", "name": "Mathematics", "isActive": true }
    ]
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/subject.controller.ts → listSubjects` · `src/services/subject.service.ts → listSubjects`

---

#### POST /admin/subjects

**Purpose:** Create a new subject (UC-79, FR-AD-013).

**Auth:** Admin

**Request body:**
```json
{
  "name": "string, required, unique"
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
    "name": "Chemistry",
    "isActive": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | `name` already exists | "A subject with this name already exists" |

**Implemented in:** `src/controllers/subject.controller.ts → createSubject` · `src/services/subject.service.ts → createSubject`

---

#### PATCH /admin/subjects/:id

**Purpose:** Deactivate (or reactivate) a subject without deleting historical matching/teaching data (UC-79, Doc 04 §4.4).

**Auth:** Admin

**Path params:** `id` — Subject UUID

**Request body:**
```json
{
  "isActive": "boolean, required"
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

**Implemented in:** `src/controllers/subject.controller.ts → deactivateSubject` · `src/services/subject.service.ts → deactivateSubject`

---

#### GET /admin/tutors/pending

**Purpose:** List tutors awaiting verification (UC-18, FR-TU-004, FR-AD-002).

**Auth:** Admin

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
    "tutors": [
      {
        "id": "uuid",
        "userId": "uuid",
        "experienceDescription": "5 years, private and group tutoring",
        "educationInstitution": "Addis Ababa University",
        "verificationStatus": "PENDING",
        "createdAt": "2026-09-01T10:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/adminTutorVerification.controller.ts → listPending` · `src/services/adminTutorVerification.service.ts → listPendingTutors`

---

#### POST /admin/tutors/:tutorId/approve

**Purpose:** Approve a tutor, making them visible/matchable (UC-18).

**Auth:** Admin

**Path params:** `tutorId` — TutorProfile UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "verificationStatus": "VERIFIED",
    "verifiedAt": "2026-09-06T15:00:00Z",
    "verifiedById": "uuid"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Tutor already `VERIFIED` or `REJECTED` | "This tutor has already been reviewed" |

**Implemented in:** `src/controllers/adminTutorVerification.controller.ts → approve` · `src/services/adminTutorVerification.service.ts → approveTutor`

---

#### POST /admin/tutors/:tutorId/reject

**Purpose:** Reject a tutor's application, allowing resubmission (UC-18 alternate flow).

**Auth:** Admin

**Path params:** `tutorId` — TutorProfile UUID

**Request body:**
```json
{
  "reason": "string, optional — internal note, not shown verbatim to the tutor"
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
    "verificationStatus": "REJECTED"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | Tutor already `VERIFIED` or `REJECTED` | "This tutor has already been reviewed" |

**Implemented in:** `src/controllers/adminTutorVerification.controller.ts → reject` · `src/services/adminTutorVerification.service.ts → rejectTutor`

---

#### GET /admin/people

**Purpose:** Manage students, parents, and tutors, and view relationship records (UC-76, FR-AD-001).

**Auth:** Admin

**Query params:**
```
?role=STUDENT&page=1&limit=20&search=string
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "users": [
      {
        "id": "uuid",
        "role": "STUDENT",
        "email": "string|null",
        "phone": "string|null",
        "createdAt": "2026-08-01T10:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

**Error responses:** none.

**Implemented in:** `src/controllers/adminPeople.controller.ts → listUsers` · `src/services/adminPeople.service.ts → listUsers`

---

#### PATCH /admin/people/relationships/:id

**Purpose:** Directly manage a parent–student relationship record as Admin (UC-76), including the Admin-initiated path into UC-09's sole-guardian-removal flow.

**Auth:** Admin

**Path params:** `id` — ParentStudentRelationship UUID

**Request body:**
```json
{
  "status": "string, optional — INVITED | ACTIVE | REVOKED",
  "permissions": "object, optional"
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
    "status": "REVOKED",
    "revokedById": "uuid"
  }
}
```

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/adminPeople.controller.ts → editRelationship` · `src/services/adminPeople.service.ts → manageRelationshipRecords`

---

#### POST /admin/people/:userId/suspend

**Purpose:** Suspend or restrict any account (UC-78, FR-AD-003, FR-SC-007), including tutors escalated via `FR-MK-003` (see Doc 04 `class-delivery-library`). If the suspended user is a tutor with active students, this also triggers the group-continuity/re-match flow (UC-33/UC-34) in Matching & Cohorts.

**Auth:** Admin

**Path params:** `userId` — User UUID

**Request body:**
```json
{
  "reason": "string, required — internal note, not disclosed to affected students (UC-34)",
  "restrictionType": "string, required — SUSPENDED | RESTRICTED"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "userId": "uuid",
    "restrictionType": "SUSPENDED",
    "affectedCohortIds": ["uuid"]
  }
}
```
`affectedCohortIds` is populated only when suspending a Tutor with active cohorts — it does not itself perform the re-match; it flags which cohorts `POST /admin/matching/manual-assign` (Matching & Cohorts feature) will need to process, or which will be picked up automatically per Doc 02 §11.3.

**Error responses:** none beyond common 404.

**Implemented in:** `src/controllers/adminPeople.controller.ts → suspendAccount` · `src/services/adminPeople.service.ts → suspendAccount, restrictAccount`

---

**Next:** proceed to → [03. Matching & Cohorts API]
