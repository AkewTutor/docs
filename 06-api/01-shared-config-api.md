## Project: AKEWTutor — API Specification
**Feature:** Shared Config (Auth, Notifications, Policies, Announcements)
**Conventions:** see 0.1–0.7 in `00-api-conventions.md` — base path `/api/v1`, envelopes, auth labels, common errors, pagination.

**Owns:** User, Notification, PolicyDocument. **Depends on:** nothing (foundation feature, per Doc 07 §1.1).

---

### 1.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| POST | /auth/register/student | Public | UC-06 | FR-SP-001, FR-SP-004, FR-SP-005, FR-SP-007, FR-AC-005 |
| POST | /auth/register/parent | Public | UC-03 | FR-SP-002, FR-SP-004, FR-SP-005, FR-AC-001, FR-AC-002 |
| POST | /auth/register/tutor | Public | UC-15 | FR-TU-001, FR-TU-002 |
| POST | /auth/login | Public | UC-10 | FR-SP-003 |
| POST | /auth/logout | Authenticated | UC-10 | FR-SP-003 |
| POST | /auth/verify-contact | Public | UC-11 | FR-SP-004 |
| POST | /auth/resend-verification | Public | UC-11 | FR-SP-004 |
| POST | /auth/forgot-password | Public | UC-10 | FR-SP-003 |
| POST | /auth/reset-password | Public | UC-10 | FR-SP-003 |
| GET | /notifications | Authenticated | UC-92, UC-93 | FR-NO-001–011 |
| PATCH | /notifications/:id/read | Authenticated | UC-92 | FR-NO-001–011 |
| GET | /policies/:type | Public | UC-02 | Section 03 (Refund Policy), FR-SC-001 |
| POST | /admin/policies | Admin | UC-85 | FR-AD-015 |
| POST | /admin/announcements | Admin | UC-89 | FR-AD-019 |
| GET | /admin/announcements | Admin | UC-89 | FR-AD-019 |

---

### 1.2 Endpoint Detail

#### POST /auth/register/student

**Purpose:** Register a Grade 6–12 student independently (UC-06). A Grade 1–5 value submitted here is rejected — that path is `POST /guardianship/students`, initiated by a parent (UC-04), not this endpoint (see FR-AC-002/005 routing rule, Doc 02 §4.2).

**Auth:** Public

**Request body:**
```json
{
  "email": "string, optional (one of email/phone required)",
  "phone": "string, optional (one of email/phone required)",
  "password": "string, required, min 8 chars",
  "grade": "integer, required, 6-12",
  "termsAccepted": "boolean, required, must be true"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "userId": "uuid",
    "role": "STUDENT",
    "studentProfileId": "uuid",
    "grade": 8,
    "accountStatus": "ACTIVE",
    "verificationRequired": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `grade` is 1–5 | "Students in Grades 1–5 require a parent-initiated account — see /guardianship/students" |
| 409 | `email`/`phone` already registered | "An account with this email/phone already exists" |

**Implemented in:** `src/controllers/auth.controller.ts → register` · `src/services/auth.service.ts → registerUser` · `src/schemas/auth.schema.ts → registerStudentSchema`

---

#### POST /auth/register/parent

**Purpose:** Register a Parent/Guardian account (UC-03). The account is created in `PENDING` onboarding state — full functionality unlocks only once a linked student activates (FR-AC-004, via `POST /guardianship/students` and `POST /guardianship/invites/:token/activate`).

**Auth:** Public

**Request body:**
```json
{
  "email": "string, optional (one of email/phone required)",
  "phone": "string, optional (one of email/phone required)",
  "password": "string, required, min 8 chars",
  "termsAccepted": "boolean, required, must be true"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "userId": "uuid",
    "role": "PARENT",
    "parentProfileId": "uuid",
    "onboardingStatus": "PENDING",
    "verificationRequired": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | `email`/`phone` already registered | "An account with this email/phone already exists" |

**Implemented in:** `src/controllers/auth.controller.ts → register` · `src/services/auth.service.ts → registerUser` · `src/schemas/auth.schema.ts → registerParentSchema`

---

#### POST /auth/register/tutor

**Purpose:** Register a Tutor account (UC-15). The account is created with `verificationStatus: PENDING` — not visible or matchable to students until `POST /admin/tutors/:tutorId/approve` (UC-18).

**Auth:** Public

**Request body:**
```json
{
  "email": "string, optional (one of email/phone required)",
  "phone": "string, optional (one of email/phone required)",
  "password": "string, required, min 8 chars",
  "termsAccepted": "boolean, required, must be true"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "userId": "uuid",
    "role": "TUTOR",
    "tutorProfileId": "uuid",
    "verificationStatus": "PENDING",
    "verificationRequired": true
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 409 | `email`/`phone` already registered | "An account with this email/phone already exists" |

**Implemented in:** `src/controllers/auth.controller.ts → register` · `src/services/auth.service.ts → registerUser` · `src/schemas/auth.schema.ts → registerTutorSchema`

---

#### POST /auth/login

**Purpose:** Authenticate any role and start a session (UC-10).

**Auth:** Public

**Request body:**
```json
{
  "identifier": "string, required — email or phone",
  "password": "string, required"
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
    "user": {
      "id": "uuid",
      "role": "STUDENT",
      "email": "string|null",
      "phone": "string|null"
    }
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 401 | Invalid credentials | "Invalid email/phone or password" — deliberately generic; never indicates which field was wrong (UC-10 alternate flow) |

**Implemented in:** `src/controllers/auth.controller.ts → login` · `src/services/auth.service.ts → login` · `src/schemas/auth.schema.ts → loginSchema`

---

#### POST /auth/logout

**Purpose:** End the current session (UC-10).

**Auth:** Authenticated

**Request body:** none

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {}
}
```

**Error responses:** none beyond the common 401.

**Implemented in:** `src/controllers/auth.controller.ts → logout` · `src/services/auth.service.ts → logout`

---

#### POST /auth/verify-contact

**Purpose:** Confirm a code/link sent to email or phone during registration (UC-11).

**Auth:** Public

**Request body:**
```json
{
  "userId": "uuid, required",
  "code": "string, required"
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
    "emailVerifiedAt": "2026-09-01T10:00:00Z",
    "phoneVerifiedAt": null
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Code expired or incorrect | "Invalid or expired code — request a new one" |

**Implemented in:** `src/controllers/auth.controller.ts → verify` · `src/services/auth.service.ts → verifyContact` · `src/schemas/auth.schema.ts → verifyContactSchema`

---

#### POST /auth/resend-verification

**Purpose:** Resend a verification code without penalizing the original registration attempt (UC-11 alternate flow).

**Auth:** Public

**Request body:**
```json
{
  "userId": "uuid, required"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "resent": true
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/auth.controller.ts → verify` · `src/services/auth.service.ts → resendVerification`

---

#### POST /auth/forgot-password

**Purpose:** Request a password-reset code/link via the verified contact method (UC-10).

**Auth:** Public

**Request body:**
```json
{
  "identifier": "string, required — email or phone"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {}
}
```
Always returns `200` regardless of whether `identifier` matches an account, to avoid confirming account existence.

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/auth.controller.ts → forgotPassword` · `src/services/auth.service.ts → requestPasswordReset` · `src/schemas/auth.schema.ts → passwordResetRequestSchema`

---

#### POST /auth/reset-password

**Purpose:** Complete a password reset (UC-10).

**Auth:** Public

**Request body:**
```json
{
  "userId": "uuid, required",
  "code": "string, required",
  "newPassword": "string, required, min 8 chars"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {}
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Code expired or already used | "This reset link is no longer valid — request a new one" |

**Implemented in:** `src/controllers/auth.controller.ts → resetPassword` · `src/services/auth.service.ts → resetPassword` · `src/schemas/auth.schema.ts → passwordResetSchema`

---

#### GET /notifications

**Purpose:** List the current user's notifications (UC-92, UC-93) — every FR-NO event category, including new-message notifications (FR-NO-011).

**Auth:** Authenticated

**Query params:**
```
?page=1&limit=20&unreadOnly=false
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "notifications": [
      {
        "id": "uuid",
        "type": "CLASS_REMINDER",
        "payload": { "sessionId": "uuid", "scheduledStart": "2026-09-06T14:00:00Z" },
        "channel": "PUSH",
        "status": "SENT",
        "sentAt": "2026-09-06T13:00:00Z",
        "readAt": null,
        "createdAt": "2026-09-06T13:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```
An account with no notifications yet returns `notifications: []` — not an error (see 0.3).

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/notification.controller.ts → listMyNotifications` · `src/services/notification.service.ts → listForUser` · `src/schemas/notification.schema.ts → listNotificationsQuerySchema`

---

#### PATCH /notifications/:id/read

**Purpose:** Mark a single notification as read.

**Auth:** Authenticated — the notification must belong to the caller (`userId` match; otherwise `403`).

**Path params:** `id` — Notification UUID

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "readAt": "2026-09-06T15:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 403 | Notification belongs to a different user | "Not authorized to modify this notification" |
| 404 | Notification doesn't exist | "Notification not found" |

**Implemented in:** `src/controllers/notification.controller.ts → markAsRead` · `src/services/notification.service.ts → markRead`

---

#### GET /policies/:type

**Purpose:** Read the current published version of a public policy page (UC-02) — Privacy, Terms, Safety, Refund, or Rules & Regulations.

**Auth:** Public

**Path params:** `type` — one of `PRIVACY | TERMS | SAFETY | REFUND | RULES`

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "type": "REFUND",
    "version": 3,
    "content": "markdown string",
    "publishedAt": "2026-08-01T00:00:00Z"
  }
}
```
Always returns the highest `version` for the requested `type` (Doc 04 §4.2.9).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `type` not one of the five valid values | "Invalid policy type" |
| 404 | No published version exists yet for this `type` | "Policy not yet published" |

**Implemented in:** `src/controllers/policy.controller.ts → getPolicy` · `src/services/policy.service.ts → getCurrentPolicy`

---

#### POST /admin/policies

**Purpose:** Publish a new version of a policy document (UC-85). Creates a new versioned row rather than editing in place, matching the `PolicyDocument(type, version)` design in Doc 04.

**Auth:** Admin

**Request body:**
```json
{
  "type": "string, required — PRIVACY | TERMS | SAFETY | REFUND | RULES",
  "content": "string, required — markdown"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "OK",
  "data": {
    "type": "REFUND",
    "version": 4,
    "publishedAt": "2026-09-06T15:00:00Z"
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/policy.controller.ts → publishPolicy` · `src/services/policy.service.ts → publishNewVersion` · `src/schemas/policy.schema.ts → publishPolicySchema`

---

#### POST /admin/announcements

**Purpose:** Compose and send a platform-wide announcement (UC-89, FR-AD-019).

**Auth:** Admin

**Request body:**
```json
{
  "title": "string, required",
  "body": "string, required",
  "audienceRoles": "array of STUDENT|PARENT|TUTOR, required, min 1"
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
    "title": "Scheduled maintenance Sept 10",
    "audienceRoles": ["STUDENT", "PARENT", "TUTOR"],
    "createdAt": "2026-09-06T15:00:00Z"
  }
}
```
Sending an announcement writes one `Notification` row per targeted `User`, through the same pipeline as every other notification type (Section 12 Definition of Done #2).

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/adminAnnouncement.controller.ts → createAnnouncement` · `src/services/adminAnnouncement.service.ts → composePlatformAnnouncement`

---

#### GET /admin/announcements

**Purpose:** List previously sent platform announcements (UC-89).

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
    "announcements": [
      {
        "id": "uuid",
        "title": "Scheduled maintenance Sept 10",
        "audienceRoles": ["STUDENT", "PARENT", "TUTOR"],
        "createdAt": "2026-09-06T15:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

**Error responses:** none beyond common validation.

**Implemented in:** `src/controllers/adminAnnouncement.controller.ts → listAnnouncements` · `src/services/adminAnnouncement.service.ts → adjustNotificationRules`

---

**Next:** proceed to → [02. Accounts & Guardianship API]
