## Project: AKEWTutor — Backend Function-Level Spec: Shared Config (Auth, Notifications, Policies, Announcements)
**Conventions:** see `00-api-conventions.md` §0.1–0.7. **API reference:** `01-shared-config-api.md`. **Folder/file reference:** `05a-backend-structure.md` §0 (cross-cutting), §1 (shared-config).

Covers every backend file not owned by a single feature — Prisma schema/seed, cross-cutting middleware and utils, job scheduler registration — plus shared-config's own owned files (auth, notifications, policies, admin announcements). Shared-config is the foundation feature (Doc 07 §1.1's "no dependency of any kind"), which is why the cross-cutting infra is documented here rather than in its own separate file.

---

## Section 0 — Cross-Cutting Foundations

### prisma/schema.prisma (new)

All 39 entities and every enum from Doc 04, in full — not restated field-by-field here (Doc 04 is the source of truth for column types/constraints). Two decisions worth flagging explicitly since they're easy to get wrong during implementation:

- **`role` on `User`** is `UserRole { STUDENT, PARENT, TUTOR, ADMIN }` — a single base `User` table plus one optional 1:1 profile table per role (`StudentProfile`, `ParentProfile`, `TutorProfile`); Admin has no profile table (Doc 04 §"One base User table" note).
- **Money fields** (`pricePerStudentPerHour`, `amount`, etc.) are `Decimal`, never `Float`, matching `00-api-conventions.md` §0.6.

### prisma/seed.ts (new)

| Field | Detail |
|---|---|
| Purpose | Bootstraps the first Admin `User` row and the base `Subject` catalog — there is no admin-registration endpoint (Doc 05a §0), so this is the only way an Admin ever comes to exist. |
| Reads from env | `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD` |
| Behavior | 1. Read seed env vars. 2. Hash the password via `password.ts`'s bcrypt wrapper (same salt rounds used everywhere else). 3. `prisma.user.upsert()` keyed on the seed email, `role: ADMIN` — upsert, not create, so re-running the seed is safe. 4. `prisma.subject.upsert()` for the initial subject list (e.g. Mathematics, Physics, Chemistry, English, Amharic), keyed on `name`, `isActive: true`. |
| Throws | If `ADMIN_SEED_EMAIL`/`ADMIN_SEED_PASSWORD` are missing, exit the process with a clear error — fail fast, don't silently skip seeding. |
| Edge cases | Re-running the seed after the Admin's password was changed via normal means would overwrite it back to the seed value — acceptable for a single-seed-admin bootstrap script, but worth a code comment for future maintainers. |

Test file: not required — seed scripts are excluded from the mirrored-test-file rule (Doc 05a §0 standing rule applies to services, not seed scripts).

### src/middleware/auth.middleware.ts (new)

| Field | Detail |
|---|---|
| Function | `authMiddleware(req, res, next)` |
| Purpose | Verifies the JWT from the `Authorization: Bearer` header via `jwt.ts`, attaches `req.user = { id, role }`. |
| Throws | `ApiError(401, "Missing, invalid, or expired token")` — no header, malformed token, expired token, or signature mismatch. |
| Edge cases | Does not itself hit the database to re-fetch the full `User` row — `req.user` carries only what's in the token payload (`id`, `role`); any handler needing fresher data (e.g. `accountStatus`) queries explicitly. |

| Function | Detail |
|---|---|
| `requireRole(...roles: Role[])` | Higher-order middleware; returns a function that checks `roles.includes(req.user.role)`, throwing `ApiError(403, "Insufficient permissions")` otherwise. Covers the Admin-only case directly (`requireRole('ADMIN')`) — no separate `adminOnly.middleware.ts` exists (Doc 05a §0). |

Test file: `tests/utils/auth.middleware.test.ts` — covers missing/expired/malformed token, and a role mismatch.

### src/middleware/errorHandler.middleware.ts (new)

| Field | Detail |
|---|---|
| Function | `errorHandlerMiddleware(err, req, res, next)` |
| Purpose | Centralized error → HTTP response mapping. Any `ApiError` (statusCode, message, errors[]) is serialized directly into the standard error envelope (`00-api-conventions.md` §0.1). Any unrecognized/unexpected error is caught, logged server-side, and returned as a generic `500` — the raw error/stack is never leaked to the client. |
| Edge cases | A Zod validation failure thrown by `validate.middleware.ts` is mapped to `400` with `errors` populated field-by-field, per §0.1/§0.2. |

Test file: `tests/utils/errorHandler.middleware.test.ts`

### src/middleware/validate.middleware.ts (new)

| Field | Detail |
|---|---|
| Function | `validate(schema: ZodSchema)` |
| Purpose | Parses `{ body, query, params }` against the given Zod schema before the controller runs; on failure throws `ApiError(400, message, errors[])` with one entry per failed field. |

Test file: `tests/utils/validate.middleware.test.ts`

### src/utils/jwt.ts (new)

| Function | Detail |
|---|---|
| `signAccessToken(payload: { id, role })` | Signs a JWT using `JWT_SECRET`; expiry per the base template's standard (a fixed short-to-medium lived access token — no refresh-token model exists in Doc 04, so re-login is the only renewal path). |
| `verifyAccessToken(token: string)` | Verifies signature + expiry; throws on any failure (caught by `authMiddleware`, not re-caught here). |

Test file: `tests/utils/jwt.test.ts`

### src/utils/password.ts (new)

| Function | Detail |
|---|---|
| `hashPassword(plain: string)` | bcrypt hash, salt rounds from `config/env.ts`. |
| `comparePassword(plain: string, hash: string)` | bcrypt compare. |

Test file: `tests/utils/password.test.ts`

### src/utils/prisma.ts (new)

Single shared `PrismaClient` instance, imported everywhere a service needs DB access — never instantiated per-request. No function-level detail beyond the singleton export.

### src/utils/pagination.ts (new)

| Function | Detail |
|---|---|
| `paginate(page?: number, limit?: number)` | Normalizes `?page=1&limit=20` defaults per `00-api-conventions.md` §0.6, returns `{ skip, take, page, limit }` for use in every paginated Prisma `findMany`. |
| `paginatedResponse(items, total, page, limit)` | Wraps a query result into the standard `{ items, page, limit, total }` shape used consistently across every list endpoint in Docs 06-api 01–08. |

Test file: `tests/utils/pagination.test.ts`

### src/jobs/scheduler.ts (new)

| Field | Detail |
|---|---|
| Purpose | Registers every cron/interval job listed under each feature's own function-level spec file (8-2 through 8-8) at process startup — a single place that imports and schedules `notificationRetry.job.ts`, `inviteReminder.job.ts`, `groupFormationWindow.job.ts`, `zeroMatchEscalation.job.ts`, `staleApproval.job.ts`, `classReminder.job.ts`, `recordingMissingCheck.job.ts`, `archiveMessageThreads.job.ts`, `paymentReminder.job.ts`, and `monthlyPayout.job.ts`. |
| Edge cases | This file has no business logic of its own — each job's actual behavior is documented in the function-level spec of the feature that owns it, not here, to avoid duplicating the same table twice. |

---

## Section 1 — Feature: `shared-config`

**Owns:** User, Notification, PolicyDocument. **Depends on:** nothing (foundation feature).

### src/schemas/auth.schema.ts (new)

| Schema | Shape |
|---|---|
| registerStudentSchema | `z.object({ body: z.object({ email: z.string().email().optional(), phone: z.string().optional(), password: z.string().min(8), grade: z.number().int().min(6).max(12), termsAccepted: z.literal(true) }).refine(b => b.email \|\| b.phone, "email or phone required") })` |
| registerParentSchema | Same base shape minus `grade`. |
| registerTutorSchema | Same base shape minus `grade`. |
| loginSchema | `z.object({ body: z.object({ identifier: z.string().min(1), password: z.string().min(1) }) })` |
| passwordResetRequestSchema | `z.object({ body: z.object({ identifier: z.string().min(1) }) })` |
| passwordResetSchema | `z.object({ body: z.object({ userId: z.string().uuid(), code: z.string().min(1), newPassword: z.string().min(8) }) })` |
| verifyContactSchema | `z.object({ body: z.object({ userId: z.string().uuid(), code: z.string().min(1) }) })` |

The `grade` 1–5 vs. 6–12 routing rule (API spec §1.2) is enforced as a business-rule check inside `registerUser`, not the schema — a 1–5 value passed here is syntactically valid but semantically rejected with a `400`, since the schema alone can't express "this endpoint only accepts one sub-range while the sibling parent-initiated endpoint accepts the other."

### src/services/auth.service.ts (new)

#### registerUser

| Field | Detail |
|---|---|
| Signature | `registerUser(role: 'STUDENT'\|'PARENT'\|'TUTOR', input): Promise<RegisterResultDTO>` |
| Purpose | Role-aware registration — creates the base `User` row plus the matching profile row (`StudentProfile`/`ParentProfile`/`TutorProfile`) in one transaction. |
| Inputs | `role`, the already-Zod-validated body |
| Output | Shape per API spec §1.2 (varies by role — `studentProfileId`+`grade` for student, `parentProfileId`+`onboardingStatus` for parent, `tutorProfileId`+`verificationStatus` for tutor) |
| Throws | `ApiError(400, ...)` — student registration with `grade` 1–5 (routing rule). `ApiError(409, "An account with this email/phone already exists")` — unique constraint violation on email/phone. |
| Side effects | `prisma.$transaction([user.create, profile.create])`; dispatches a verification code via `notification.service.ts` → `sms.client.ts`/`email.client.ts` depending on which of email/phone was supplied. |
| Edge cases | Tutor accounts are created with `verificationStatus: PENDING` and are not visible/matchable until an Admin approves (`accounts-guardianship`'s `adminTutorVerification.service.ts`) — this function's job ends at account creation, not verification. Parent accounts start `onboardingStatus: PENDING` until a linked student activates. |

Test file: `tests/services/auth.service.test.ts` — includes duplicate-email/phone 409 case and the grade-routing 400 case.

#### login

| Field | Detail |
|---|---|
| Signature | `login(identifier: string, password: string): Promise<{ accessToken: string; user: AuthUserDTO }>` |
| Purpose | Authenticate any role by email or phone. |
| Throws | `ApiError(401, "Invalid email/phone or password")` — identifier not found, or password mismatch; identical message and status either way, never revealing which field was wrong (API spec §1.2). |
| Side effects | `bcrypt.compare` via `password.ts`; on success, `jwt.ts` → `signAccessToken({ id, role })`. |
| Edge cases | As with the Shadow Economy reference pattern, `bcrypt.compare` should still execute (against a dummy hash) even when `identifier` isn't found, so response timing doesn't leak account existence — called out explicitly as a hardening detail to implement, not to skip as an optimization. |

Test file: `tests/services/auth.service.test.ts` — explicitly covers "identifier not found" and "wrong password" returning the identical message and comparable timing behavior.

#### logout

| Field | Detail |
|---|---|
| Signature | `logout(): Promise<void>` |
| Purpose | End the current session. |
| Side effects | None — auth is stateless JWT with no session table in Doc 04's schema; logout is effectively "the client discards the token." This is a deliberate, honest V1 no-op (not an oversight) — matching the same design tension called out in the Admin Access reference pattern for single-token systems. |

Test file: `tests/services/auth.service.test.ts`

#### requestPasswordReset

| Field | Detail |
|---|---|
| Signature | `requestPasswordReset(identifier: string): Promise<void>` |
| Purpose | Sends a reset code to the verified contact method. |
| Throws | Never — always resolves successfully regardless of whether `identifier` matches an account (API spec §1.2's deliberate non-disclosure). |
| Side effects | If a matching `User` exists, generates a reset code and dispatches via `notification.service.ts`; if not, does nothing but still returns the same success shape. |
| Edge cases | This function must not let a downstream error (e.g. SMS provider failure) leak a different response shape than the no-match case — both paths converge on the identical `200` response at the controller. |

Test file: `tests/services/auth.service.test.ts` — covers both matching and non-matching identifiers producing identical outward behavior.

#### resetPassword

| Field | Detail |
|---|---|
| Signature | `resetPassword(userId: string, code: string, newPassword: string): Promise<void>` |
| Throws | `ApiError(400, "This reset link is no longer valid — request a new one")` — code expired or already used. |
| Side effects | Hashes `newPassword`, updates `User.passwordHash`, invalidates the reset code (single-use). |

Test file: `tests/services/auth.service.test.ts`

#### verifyContact / resendVerification

| Field | Detail |
|---|---|
| Signature | `verifyContact(userId: string, code: string): Promise<{ emailVerifiedAt, phoneVerifiedAt }>` · `resendVerification(userId: string): Promise<void>` |
| Throws | (verify) `ApiError(400, "Invalid or expired code — request a new one")`. |
| Side effects | (verify) Sets `emailVerifiedAt`/`phoneVerifiedAt` depending on which contact method the code was issued for. (resend) Regenerates and redispatches a code without penalizing the original registration attempt — no rate limit is enforced server-side on this endpoint per the API spec, so any throttling is a client-side courtesy only. |

Test file: `tests/services/auth.service.test.ts`

### src/controllers/auth.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| register | `authService.registerUser(role, req.body)` — `role` fixed per route (`/register/student`, `/register/parent`, `/register/tutor`) | 201 |
| login | `authService.login(req.body.identifier, req.body.password)` | 200 |
| logout | `authService.logout()` | 200, `{}` |
| verify | `authService.verifyContact` or `authService.resendVerification`, branched by route | 200 |
| forgotPassword | `authService.requestPasswordReset(req.body.identifier)` | 200, `{}` always |
| resetPassword | `authService.resetPassword(req.body.userId, req.body.code, req.body.newPassword)` | 200, `{}` |

### src/routes/auth.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | /register/student | `validate(registerStudentSchema)` | register |
| POST | /register/parent | `validate(registerParentSchema)` | register |
| POST | /register/tutor | `validate(registerTutorSchema)` | register |
| POST | /login | `validate(loginSchema)` | login |
| POST | /logout | `authMiddleware` | logout |
| POST | /verify-contact | `validate(verifyContactSchema)` | verify |
| POST | /resend-verification | — | verify |
| POST | /forgot-password | `validate(passwordResetRequestSchema)` | forgotPassword |
| POST | /reset-password | `validate(passwordResetSchema)` | resetPassword |

All public except `logout`. Mounted at `/auth`.

---

### src/utils/providers/sms.client.ts, email.client.ts (new)

Not given per-function tables — thin wrappers around Geez SMS and Brevo respectively, per `00-api-conventions.md` §0.5. Shared contract each exposes:

```typescript
interface NotificationProviderClient {
  send(to: string, subjectOrTemplate: string, body: string): Promise<{ success: boolean; providerRef?: string }>;
}
```

Called only from `notification.service.ts` and `auth.service.ts` — neither has a client-facing endpoint of its own. Each has its own test file mocking the underlying provider call.

### src/schemas/notification.schema.ts (new)

| Schema | Shape |
|---|---|
| listNotificationsQuerySchema | `z.object({ query: z.object({ page: z.coerce.number().int().min(1).default(1), limit: z.coerce.number().int().min(1).max(100).default(20), unreadOnly: z.coerce.boolean().default(false) }) })` |

### src/services/notification.service.ts (new)

#### dispatchNotification

| Field | Detail |
|---|---|
| Signature | `dispatchNotification(userId: string, type: NotificationType, payload: object): Promise<void>` |
| Purpose | Single entry point every other feature calls to notify a user — writes one `Notification` row per targeted user and routes delivery by `User.preferredNotificationChannel`. |
| Side effects | Creates `Notification(status: PENDING)`, then calls `sms.client.ts`/`email.client.ts`/a push provider depending on channel; on success sets `status: SENT, sentAt`; on failure sets `status: FAILED` for `notificationRetry.job.ts` to pick up later. |
| Edge cases | This function never throws back into the caller — a downstream delivery failure must never block the business action that triggered it (e.g. a failed SMS must not roll back a completed class session). All error handling here terminates in a `FAILED` row, not a re-thrown exception. |

Test file: `tests/services/notification.service.test.ts` — covers both delivery success and swallowed-failure-writes-FAILED-row paths.

#### listForUser / markRead

| Field | Detail |
|---|---|
| Signature | `listForUser(userId: string, unreadOnly: boolean, page, limit): Promise<PaginatedNotificationsDTO>` · `markRead(notificationId: string, userId: string): Promise<{ id, readAt }>` |
| Throws | (markRead) `ApiError(403, "Not authorized to modify this notification")` — `Notification.userId !== userId`. `ApiError(404, "Notification not found")`. |
| Edge cases | (listForUser) No notifications yet → `notifications: []`, `200` — not an error, per §0.3. |

Test file: `tests/services/notification.service.test.ts`

#### retryFailed

| Field | Detail |
|---|---|
| Signature | `retryFailed(): Promise<{ retried: number }>` |
| Purpose | Called by `notificationRetry.job.ts` on an interval — re-attempts every `Notification.status = FAILED` row. |
| Edge cases | Does not retry indefinitely — a max-attempt cap (e.g. 3) is a config constant to set at implementation time; a row that exhausts retries stays `FAILED` permanently rather than looping forever. |

Test file: `tests/services/notification.service.test.ts`

### src/controllers/notification.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listMyNotifications | `notificationService.listForUser(req.user.id, req.query.unreadOnly, req.query.page, req.query.limit)` | 200 |
| markAsRead | `notificationService.markRead(req.params.id, req.user.id)` | 200 |

### src/routes/notification.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware, validate(listNotificationsQuerySchema)` | listMyNotifications |
| PATCH | /:id/read | `authMiddleware` | markAsRead |

Mounted at `/notifications`.

---

### src/services/adminAnnouncement.service.ts (new)

#### composePlatformAnnouncement

| Field | Detail |
|---|---|
| Signature | `composePlatformAnnouncement(title: string, body: string, audienceRoles: Role[], createdById: string): Promise<AnnouncementDTO>` |
| Purpose | Sends a platform-wide announcement — writes one `Notification` row per `User` whose `role` is in `audienceRoles`, through `notification.service.ts`'s `dispatchNotification`, per API spec §1.2. |
| Side effects | `prisma.user.findMany({ where: { role: { in: audienceRoles } } })`, then one `dispatchNotification` call per matched user. For a large user base this should be batched (e.g. chunks of a few hundred) rather than one `Promise.all` firing thousands of concurrent sends — a note for implementation, not a hard requirement from the source docs. |

Test file: `tests/services/adminAnnouncement.service.test.ts`

#### adjustNotificationRules

| Field | Detail |
|---|---|
| Signature | `listAnnouncements(page, limit): Promise<PaginatedAnnouncementsDTO>` |
| Purpose | List previously sent announcements (API spec §1.2 `GET /admin/announcements`) — despite the file-structure doc's naming this function `adjustNotificationRules`, the endpoint it actually serves per the API spec is a plain paginated list read; there is no separate "notification rules" configuration surface documented anywhere in Docs 01–04, so this is implemented as the list read the API contract requires. Worth flagging to whoever wrote Doc 05a rather than silently renaming it. |

Test file: `tests/services/adminAnnouncement.service.test.ts`

### src/controllers/adminAnnouncement.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| createAnnouncement | `adminAnnouncementService.composePlatformAnnouncement(req.body.title, req.body.body, req.body.audienceRoles, req.user.id)` | 201 |
| listAnnouncements | `adminAnnouncementService.adjustNotificationRules(req.query.page, req.query.limit)` | 200 |

### src/routes/adminAnnouncement.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | / | `authMiddleware, requireRole('ADMIN')` | createAnnouncement |
| GET | / | `authMiddleware, requireRole('ADMIN')` | listAnnouncements |

Mounted at `/admin/announcements`.

---

### src/schemas/policy.schema.ts (new)

| Schema | Shape |
|---|---|
| publishPolicySchema | `z.object({ body: z.object({ type: z.enum(['PRIVACY','TERMS','SAFETY','REFUND','RULES']), content: z.string().min(1) }) })` |

### src/services/policy.service.ts (new)

#### getCurrentPolicy

| Field | Detail |
|---|---|
| Signature | `getCurrentPolicy(type: PolicyType): Promise<PolicyDocumentDTO>` |
| Purpose | Returns the highest `version` row for the given `type` (Doc 04 §4.2.9). |
| Throws | `ApiError(400, "Invalid policy type")` — caught at the schema/route param layer in practice, but re-validated here defensively since `type` arrives as a raw path param, not a body field. `ApiError(404, "Policy not yet published")` — zero versions exist yet for this `type`. |

Test file: `tests/services/policy.service.test.ts`

#### publishNewVersion

| Field | Detail |
|---|---|
| Signature | `publishNewVersion(type: PolicyType, content: string, createdById: string): Promise<{ type, version, publishedAt }>` |
| Purpose | Creates a new versioned row rather than editing in place — `version = (current max version for type) + 1`. |
| Side effects | Insert-only; never updates or deletes a prior version, preserving full history. |

Test file: `tests/services/policy.service.test.ts` — includes a versioning-increments-correctly case across repeated publishes.

### src/controllers/policy.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getPolicy | `policyService.getCurrentPolicy(req.params.type)` | 200 |
| publishPolicy | `policyService.publishNewVersion(req.body.type, req.body.content, req.user.id)` | 201 |

### src/routes/policy.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /policies/:type | — | getPolicy |
| POST | /admin/policies | `authMiddleware, requireRole('ADMIN'), validate(publishPolicySchema)` | publishPolicy |

This router mounts at both `/policies` (public) and `/admin/policies` (admin) — the one router file spans two mount points, matching `00-api-conventions.md` §0.7's deliberate public/admin path split for the same underlying entity.

---

### src/jobs/notificationRetry.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Fixed interval (e.g. every 5 minutes) — a config constant to set at implementation time; not specified in Docs 01–04. |
| Effect | Calls `notification.service.ts → retryFailed()`. |
| Idempotency | Safe to run concurrently with itself missing (relies on the service's own `status = FAILED` filter — a row already retried and moved to `SENT` before this run started is simply not picked up). |

Test file: `tests/services/notification.service.test.ts` covers the underlying `retryFailed` logic directly; the job wrapper itself (interval registration) is not unit-tested per the standing convention that only `src/services/*` gets a mirrored test file.

---

**Next:** proceed to → [8-2. Backend: Accounts & Guardianship]
