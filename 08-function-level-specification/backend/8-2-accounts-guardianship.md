## Project: AKEWTutor — Backend Function-Level Spec: Accounts & Guardianship
**Conventions:** see `00-api-conventions.md` §0.1–0.7. **API reference:** `02-accounts-guardianship-api.md`. **Folder/file reference:** `05a-backend-structure.md` §2.

**Owns:** StudentProfile, ParentProfile, TutorProfile, ParentStudentRelationship, Subject, TutorSubjectRanking, AvailabilitySlot. **Depends on:** `shared-config` (hard).

**Links back to:** [06-api/02-accounts-guardianship-api.md], [05a. Backend Folder & File Structure §2]
**Links forward to:** [9-2. Backend Test Spec: Accounts & Guardianship]

---

### src/schemas/studentProfile.schema.ts (new)

| Schema | Shape |
|---|---|
| updateAcademicProfileSchema | `z.object({ body: z.object({ studentId: z.string().uuid().optional(), grade: z.number().int().min(1).max(12).optional(), school: z.string().optional(), subjectsOfInterest: z.array(z.string().uuid()).optional(), academicLevel: z.string().optional(), learningGoals: z.string().optional(), preferredLanguage: z.string().optional(), learningSchedulePreference: z.object({}).passthrough().optional(), teachingStylePreference: z.string().optional(), budgetPreference: z.string().optional(), formatPreference: z.enum(['ONE_TO_ONE','ONE_TO_THREE','ONE_TO_FIVE']).optional() }) })` — every field optional, since the endpoint accepts partial updates (API spec §2.2). |

### src/services/studentProfile.service.ts (new)

#### assertAccountStatusAllowsAccess

> ✨ **Gap closed (Pre-Implementation Hardening).** `04-database-and-data-model.md §4.2` (StudentProfile) previously stated the `GUARDIAN_REQUIRED_HOLD` hold "is enforced at the application layer against every booking/class-access check" without naming the function that does it — leaving `matching-cohorts` and `class-delivery-library` each free to implement (or skip) the check independently. This is the single named enforcement point every other feature calls.

| Field | Detail |
|---|---|
| Signature | `assertAccountStatusAllowsAccess(studentId: string): Promise<void>` |
| Purpose | The one platform-wide gate for FR-AC-008's "guardian required" hold. Called by any other feature's function *before* it performs a booking- or class-access-relevant action on behalf of a student — it is not itself a route handler and has no corresponding endpoint. |
| Throws | `ApiError(403, "This student's account is on hold pending a guardian — booking and class access are unavailable until a guardian is linked")` — `StudentProfile.accountStatus !== 'ACTIVE'` (covers both `PENDING_ACTIVATION` and `GUARDIAN_REQUIRED_HOLD`; a not-yet-activated Grades 1–5 student and a hold-state student get the identical error, since neither should be able to trigger a booking or session-join action). |
| Side effects | Read-only — a single `StudentProfile.findUnique` on `accountStatus`. Resolves with no return value when the account is `ACTIVE`. |
| Callers (cross-feature, soft dependency — service call, no FK, per `feature-decomposition.md §1.1`) | `matching-cohorts`: `matching.service.ts → selectTutor`, `triggerNoExactMatch`, `requestGroupFormat` (all three booking-entry points, called first, before any `MatchRequest`/`Cohort` row is created). `class-delivery-library`: `session.service.ts → assertSessionAccessAllowed` (see `8-4-class-delivery-library.md`), called before `GET /sessions` / `GET /sessions/:sessionId` return session data to a Student/Parent caller. |
| Not called by | Admin-facing reads/actions (Admin must always be able to see and manage a held account) and the guardianship endpoints themselves (a parent must be able to view/manage the hold state and send a new invite while the student is on hold — that is the intended way out of the state, so gating it here would create a deadlock). |

Test file: `tests/services/studentProfile.service.test.ts`

#### getProfile

| Field | Detail |
|---|---|
| Signature | `getProfile(callerId: string, callerRole: 'STUDENT'\|'PARENT', studentId?: string): Promise<StudentProfileDTO>` |
| Purpose | Retrieve the caller's own profile, or — for a Parent — a linked student's, scoped by an `ACTIVE` `ParentStudentRelationship`. |
| Throws | `ApiError(403, "Not authorized to view this student's profile")` — Parent supplies a `studentId` with no `ACTIVE` relationship to it. |
| Side effects | Read-only. For `PARENT`, joins through `ParentStudentRelationship` to confirm `status: ACTIVE` before returning the profile. |

Test file: `tests/services/studentProfile.service.test.ts`

#### updateBasicProfile

| Field | Detail |
|---|---|
| Signature | `updateBasicProfile(callerId, callerRole, studentId, input: { profilePictureUrl? }): Promise<{ id, profilePictureUrl }>` |
| Throws | `ApiError(400, "Unsupported image format or file too large")` — invalid image, enforced at this layer since it's a business-rule check beyond simple field presence. Same ownership check as `getProfile`. |

Test file: `tests/services/studentProfile.service.test.ts`

#### updateAcademicProfile

| Field | Detail |
|---|---|
| Signature | `updateAcademicProfile(callerId, callerRole, studentId, input): Promise<StudentAcademicProfileDTO>` |
| Purpose | Sets the fields the matching engine depends on (API spec §2.2). |
| Side effects | Partial `prisma.studentProfile.update` — only supplied fields are touched. |
| Edge cases | Setting `formatPreference` here does **not** create a `MatchRequest` — that is a separate, explicit call into `matching-cohorts`' own endpoints (API spec §2.2 note). The "must have grade + at least one subject + formatPreference before matching" rule is enforced at the matching endpoints, not here — this function accepts a genuinely incomplete profile without error. |

Test file: `tests/services/studentProfile.service.test.ts`

### src/controllers/studentProfile.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getMyProfile | `studentProfileService.getProfile(req.user.id, req.user.role, req.query.studentId)` | 200 |
| updateBasicProfile | `studentProfileService.updateBasicProfile(req.user.id, req.user.role, req.body.studentId, req.body)` | 200 |
| updateAcademicProfile | `studentProfileService.updateAcademicProfile(req.user.id, req.user.role, req.body.studentId, req.body)` | 200 |

### src/routes/studentProfile.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /students/me/profile | `authMiddleware` | getMyProfile |
| PATCH | /students/me/profile | `authMiddleware` | updateBasicProfile |
| PATCH | /students/me/academic-profile | `authMiddleware, validate(updateAcademicProfileSchema)` | updateAcademicProfile |

Mounted at `/students/me`. Ownership scoping (Student vs. Parent-with-ACTIVE-relationship) is enforced in the service layer per `00-api-conventions.md` §0.1, not in route middleware.

---

### src/schemas/guardianship.schema.ts (new)

| Schema | Shape |
|---|---|
| addStudentSchema | `z.object({ body: z.object({ grade: z.number().int().min(1).max(12), inviteContact: z.string().min(1) }) })` |
| inviteGuardianSchema | `z.object({ body: z.object({ inviteContact: z.string().min(1) }) })` |
| revokeRelationshipSchema | `z.object({ body: z.object({ permissions: z.object({}).passthrough().optional(), revoke: z.boolean().default(false) }) })` |

### src/services/guardianship.service.ts (new)

#### addStudentAndInvite

| Field | Detail |
|---|---|
| Signature | `addStudentAndInvite(parentId: string, grade: number, inviteContact: string): Promise<GuardianshipInviteDTO>` |
| Purpose | Grades 1–5 parent-initiated student creation + activation invite (UC-04). |
| Throws | `ApiError(400, "Grades 6–12 students register independently — see /auth/register/student")` — `grade` is 6–12. |
| Side effects | Creates a placeholder `StudentProfile` (not yet linked to a `User`) plus a `ParentStudentRelationship(status: INVITED, relationshipType: MANDATORY_GUARDIAN)`, sets `inviteExpiresAt` to +14 days, dispatches the invite via `notification.service.ts`. |

Test file: `tests/services/guardianship.service.test.ts`

#### resendOrRegenerateInvite

| Field | Detail |
|---|---|
| Signature | `resendOrRegenerateInvite(parentId: string, relationshipId: string): Promise<{ relationshipId, inviteExpiresAt }>` |
| Throws | `ApiError(403, "Not authorized to manage this invite")` — relationship not owned by caller. `ApiError(409, "This student has already activated their account")` — `status` already `ACTIVE`. |
| Side effects | Regenerates the invite token, resets `inviteExpiresAt` to +14 days from now. |

Test file: `tests/services/guardianship.service.test.ts` — includes the 14-day expiry reset case.

#### activateInvite

| Field | Detail |
|---|---|
| Signature | `activateInvite(token: string, password: string): Promise<{ accessToken, studentId, relationshipStatus }>` |
| Throws | `ApiError(400, "This invite is no longer valid — ask your parent/guardian to resend it")` — expired. `ApiError(404, "Invite not found")` — token unknown. |
| Side effects | Creates the `User` row (role `STUDENT`) tied to the placeholder `StudentProfile`, hashes the password, sets `ParentStudentRelationship.status: ACTIVE, activatedAt`, issues an access token. |

Test file: `tests/services/guardianship.service.test.ts`

#### inviteOptionalGuardian

| Field | Detail |
|---|---|
| Signature | `inviteOptionalGuardian(studentId: string, inviteContact: string): Promise<GuardianshipInviteDTO>` |
| Purpose | A Grade 6–12 student invites an optional guardian (UC-07) — purely additive, never gates the student's own access. |
| Throws | `ApiError(403, "This action is only available to Grade 6–12 students")` — caller's `StudentProfile.grade` is 1–5. |
| Side effects | Creates `ParentStudentRelationship(relationshipType: OPTIONAL_GUARDIAN, status: INVITED)`. |

Test file: `tests/services/guardianship.service.test.ts`

#### revokeOrModifyRelationship / handleSoleGuardianRemoval

| Field | Detail |
|---|---|
| Signature | `revokeOrModifyRelationship(callerId, callerRole, relationshipId, input: { permissions?, revoke? }): Promise<RelationshipResultDTO>` |
| Purpose | Revoke or edit permissions on a relationship; when the revoked relationship is the student's **sole** `MANDATORY_GUARDIAN` link, delegates to `handleSoleGuardianRemoval` to place the student into `GUARDIAN_REQUIRED_HOLD`. |
| Throws | `ApiError(403, "Only a guardian or Admin can remove this relationship")` — a Grade 1–5 student attempting to revoke their own mandatory guardian. `ApiError(403, "Not authorized to modify this relationship")` — a Grade 6–12 student attempting to revoke a relationship they didn't initiate. |
| Side effects | (sole-guardian case) Sets `StudentProfile.accountStatus: GUARDIAN_REQUIRED_HOLD` — explicitly does **not** delete data, progress, XP, or recordings (FR-AC-008). |
| Edge cases | `studentAccountStatus` is only present in the response when this was the sole mandatory guardian being removed — every other revoke/modify omits the field entirely (API spec §2.2), which the service enforces by conditionally including the key, not by returning it as `null`. |

Test file: `tests/services/guardianship.service.test.ts` — includes the sole-guardian-hold case explicitly.

### src/controllers/guardianship.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| addStudent | `guardianshipService.addStudentAndInvite(req.user.id, req.body.grade, req.body.inviteContact)` | 201 |
| resendInvite | `guardianshipService.resendOrRegenerateInvite(req.user.id, req.params.relationshipId)` | 200 |
| activateInvite | `guardianshipService.activateInvite(req.params.token, req.body.password)` | 200 |
| inviteGuardian | `guardianshipService.inviteOptionalGuardian(req.user.id, req.body.inviteContact)` | 201 |
| listRelationships | `guardianshipService.listRelationships(req.user.id, req.user.role, req.query.status)` — co-located, not separately tabled in Doc 05a but required by API spec §2.1's `GET /guardianship/relationships` | 200 |
| revokeRelationship | `guardianshipService.revokeOrModifyRelationship(req.user.id, req.user.role, req.params.id, req.body)` | 200 |

### src/routes/guardianship.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | /students | `authMiddleware, validate(addStudentSchema)` | addStudent |
| POST | /invites/:relationshipId/resend | `authMiddleware` | resendInvite |
| POST | /invites/:token/activate | — (public; token is the credential) | activateInvite |
| POST | /guardian-invites | `authMiddleware, validate(inviteGuardianSchema)` | inviteGuardian |
| GET | /relationships | `authMiddleware` | listRelationships |
| PATCH | /relationships/:id/revoke | `authMiddleware, validate(revokeRelationshipSchema)` | revokeRelationship |

Mounted at `/guardianship`.

### src/jobs/inviteReminder.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Daily interval, checks `ParentStudentRelationship(status: INVITED)` rows created exactly 7 days ago and not yet reminded. |
| Effect | Sends a day-7 activation reminder via `notification.service.ts`. |
| Idempotency | Should mark a `reminderSentAt` field (or equivalent) so a job re-run the same day doesn't double-send — flagged as an implementation detail since Doc 04 doesn't explicitly name this column. |

---

### src/schemas/tutorProfile.schema.ts (new)

| Schema | Shape |
|---|---|
| updateTutorProfileSchema | `z.object({ body: z.object({ profilePictureUrl: z.string().url().optional(), bio: z.string().optional(), experienceDescription: z.string().optional(), educationInstitution: z.string().optional(), degree: z.string().optional() }) })` |
| rankSubjectsSchema | `z.object({ body: z.object({ subjects: z.array(z.object({ subjectId: z.string().uuid(), rank: z.union([z.literal(1), z.literal(2)]) })).min(1).max(2).refine(arr => new Set(arr.map(s => s.subjectId)).size === arr.length, "Each subject may be ranked once").refine(arr => new Set(arr.map(s => s.rank)).size === arr.length, "Ranks must be unique") }) })` |

### src/services/tutorProfile.service.ts (new)

#### getProfile / updateProfile

| Field | Detail |
|---|---|
| Signature | `getProfile(tutorId: string): Promise<TutorProfileDTO>` · `updateProfile(tutorId: string, input): Promise<Partial<TutorProfileDTO>>` |
| Edge cases | `updateProfile` never allows self-service edits to `verificationStatus` — that field is owned exclusively by `adminTutorVerification.service.ts`, with the sole exception of `resubmitVerification` below (a narrow, single-purpose transition, not a general re-opening of the field). |

Test file: `tests/services/tutorProfile.service.test.ts`

#### resubmitVerification

| Field | Detail |
|---|---|
| Signature | `resubmitVerification(tutorId: string): Promise<{ id: string; verificationStatus: 'PENDING' }>` |
| Purpose | Issue 2 fix — names the mechanism `10-e2e-specification.md §10.8` (E2E-6) and `09-test-file-specification/phase7-review-signoff.md` flagged as missing: a tutor-initiated way to re-queue a `REJECTED` application after correcting their profile via the existing `updateProfile`. |
| Throws | `ApiError(409, "Only a rejected application can be resubmitted")` — `verificationStatus` is not currently `REJECTED`. |
| Side effects | Sets `verificationStatus: PENDING`, and resets `verifiedAt: null, verifiedById: null` — the previous rejection's Admin/timestamp no longer describes the current (re-pending) state. This is the one narrow case where `tutorProfile.service.ts` is allowed to write `verificationStatus`; every other transition still belongs exclusively to `adminTutorVerification.service.ts`. Does not itself touch any other profile field — the tutor corrects those separately via `PATCH /tutors/me/profile` first. |
| Edge cases | Calling this while `verificationStatus` is `PENDING` or `VERIFIED` throws the same 409 — resubmission is only ever a `REJECTED → PENDING` transition. |

Test file: `tests/services/tutorProfile.service.test.ts`

#### rankSubjects

| Field | Detail |
|---|---|
| Signature | `rankSubjects(tutorId: string, subjects: { subjectId: string; rank: 1\|2 }[]): Promise<TutorSubjectRankingDTO[]>` |
| Purpose | Full replace (matches the `PUT` semantics of the endpoint) — rank order is meaningful and resolved atomically. |
| Throws | `ApiError(400, "A tutor may rank a maximum of two subjects")` — more than 2 submitted (also caught by the schema's `.max(2)`, but re-checked here as the authoritative business rule per Doc 02 §6.2/FR-TU-007). `ApiError(400, "Each subject may be ranked once, and ranks must be unique")` — duplicate `subjectId` or `rank` (also schema-enforced; service-level check exists so the hard two-subject cap and uniqueness rule are never bypassable even if the schema is relaxed later). |
| Side effects | `prisma.$transaction([deleteMany existing rankings for tutor, createMany new rankings])` — atomic full replace, never a partial update that could leave 3 active rankings mid-transaction. |
| Edge cases | Each ranked subject applies across the full Grade 1–12 span — there is no per-rank grade-range field to set (Doc 02 §6.2). |

Test file: `tests/services/tutorProfile.service.test.ts` — includes the third-subject-rejected case and the atomic-replace transaction case.

### src/controllers/tutorProfile.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getMyProfile | `tutorProfileService.getProfile(req.user.id)` | 200 |
| updateProfile | `tutorProfileService.updateProfile(req.user.id, req.body)` | 200 |
| resubmitVerification | `tutorProfileService.resubmitVerification(req.user.id)` | 200 |
| rankSubjects | `tutorProfileService.rankSubjects(req.user.id, req.body.subjects)` | 200 |

### src/routes/tutorProfile.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /me/profile | `authMiddleware` | getMyProfile |
| PATCH | /me/profile | `authMiddleware, validate(updateTutorProfileSchema)` | updateProfile |
| POST | /me/resubmit-verification | `authMiddleware` | resubmitVerification |
| PUT | /me/subjects | `authMiddleware, validate(rankSubjectsSchema)` | rankSubjects |

Mounted at `/tutors`.

---

### src/schemas/availability.schema.ts (new)

| Schema | Shape |
|---|---|
| createSlotSchema | `z.object({ body: z.object({ dayOfWeek: z.number().int().min(0).max(6).optional(), startTime: z.string().datetime(), endTime: z.string().datetime(), isRecurring: z.boolean() }).refine(b => !b.isRecurring \|\| b.dayOfWeek !== undefined, "dayOfWeek required for recurring slots").refine(b => new Date(b.endTime) > new Date(b.startTime), "End time must be after start time") })` |
| deleteSlotSchema | `z.object({ params: z.object({ slotId: z.string().uuid() }) })` |

### src/services/availability.service.ts (new)

#### setSlots / listSlots

| Field | Detail |
|---|---|
| Signature | `setSlots(tutorId: string, input): Promise<AvailabilitySlotDTO>` · `listSlots(tutorId: string): Promise<AvailabilitySlotDTO[]>` |
| Throws | (setSlots) `ApiError(400, "End time must be after start time")` — redundant with the schema check, retained as the authoritative business rule. |

Test file: `tests/services/availability.service.test.ts`

#### removeSlot

| Field | Detail |
|---|---|
| Signature | `removeSlot(tutorId: string, slotId: string): Promise<{ id, deleted: true }>` |
| Throws | `ApiError(403, "Not authorized to remove this slot")` — slot belongs to a different tutor. `ApiError(409, "This slot is in use by a confirmed session and cannot be removed until it is resolved")` — a confirmed `ScheduledSession` depends on this slot. |
| Side effects | Checks for a dependent confirmed session (`class-delivery-library`'s `ScheduledSession`, no FK to `AvailabilitySlot` directly per Doc 04 but resolved via the tutor/time-window overlap at query time) before deleting. |

Test file: `tests/services/availability.service.test.ts` — includes the slot-in-use-cannot-delete case.

### src/controllers/availability.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listMyAvailability | `availabilityService.listSlots(req.user.id)` | 200 |
| addSlot | `availabilityService.setSlots(req.user.id, req.body)` | 201 |
| removeSlot | `availabilityService.removeSlot(req.user.id, req.params.slotId)` | 200 |

### src/routes/availability.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /me/availability | `authMiddleware` | listMyAvailability |
| POST | /me/availability | `authMiddleware, validate(createSlotSchema)` | addSlot |
| DELETE | /me/availability/:slotId | `authMiddleware, validate(deleteSlotSchema)` | removeSlot |

Mounted at `/tutors` (sibling to `tutorProfile.routes.ts`, both under `/tutors/me/*`).

---

### src/services/subject.service.ts (new)

#### listSubjects / createSubject / deactivateSubject

| Field | Detail |
|---|---|
| Signature | `listSubjects(includeInactive: boolean): Promise<SubjectDTO[]>` · `createSubject(name: string): Promise<SubjectDTO>` · `deactivateSubject(id: string, isActive: boolean): Promise<{ id, isActive }>` |
| Throws | (createSubject) `ApiError(409, "A subject with this name already exists")` — unique constraint. |
| Side effects | `deactivateSubject` never deletes a `Subject` row — flips `isActive` only, preserving historical matching/teaching data referencing it (Doc 04 §4.4). |

Test file: `tests/services/subject.service.test.ts`

### src/controllers/subject.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listSubjects | `subjectService.listSubjects(req.query.includeInactive)` | 200 |
| createSubject | `subjectService.createSubject(req.body.name)` | 201 |
| deactivateSubject | `subjectService.deactivateSubject(req.params.id, req.body.isActive)` | 200 |

### src/routes/subject.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /subjects | — | listSubjects |
| POST | /admin/subjects | `authMiddleware, requireRole('ADMIN')` | createSubject |
| PATCH | /admin/subjects/:id | `authMiddleware, requireRole('ADMIN')` | deactivateSubject |

Same public/admin split pattern as `policy.routes.ts` — one router file, two mount points.

---

### src/services/adminTutorVerification.service.ts (new)

#### listPendingTutors / approveTutor / rejectTutor

| Field | Detail |
|---|---|
| Signature | `listPendingTutors(page, limit): Promise<PaginatedTutorDTO>` · `approveTutor(tutorId: string, adminId: string): Promise<TutorVerificationResultDTO>` · `rejectTutor(tutorId: string, adminId: string, reason?: string): Promise<{ id, verificationStatus }>` |
| Throws | (approve/reject) `ApiError(409, "This tutor has already been reviewed")` — `verificationStatus` already `VERIFIED` or `REJECTED`. |
| Side effects | (approve) Sets `verificationStatus: VERIFIED, verifiedAt, verifiedById` — this is what makes the tutor visible/matchable to `matching-cohorts`' search and recommendation queries. (reject) `reason` is stored as an internal note only, never surfaced to the tutor verbatim (API spec §2.2) — the tutor-facing notification uses a generic rejection message from `notification.service.ts`, not `reason` itself. |

Test file: `tests/services/adminTutorVerification.service.test.ts`

### src/controllers/adminTutorVerification.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listPending | `adminTutorVerificationService.listPendingTutors(req.query.page, req.query.limit)` | 200 |
| approve | `adminTutorVerificationService.approveTutor(req.params.tutorId, req.user.id)` | 200 |
| reject | `adminTutorVerificationService.rejectTutor(req.params.tutorId, req.user.id, req.body.reason)` | 200 |

### src/routes/adminTutorVerification.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /pending | `authMiddleware, requireRole('ADMIN')` | listPending |
| POST | /:tutorId/approve | `authMiddleware, requireRole('ADMIN')` | approve |
| POST | /:tutorId/reject | `authMiddleware, requireRole('ADMIN')` | reject |

Mounted at `/admin/tutors`.

---

### src/services/adminPeople.service.ts (new)

#### listUsers

| Field | Detail |
|---|---|
| Signature | `listUsers(role?: Role, search?: string, page?, limit?): Promise<PaginatedUserDTO>` |

#### manageRelationshipRecords

| Field | Detail |
|---|---|
| Signature | `manageRelationshipRecords(relationshipId: string, adminId: string, input: { status?, permissions? }): Promise<RelationshipResultDTO>` |
| Purpose | Admin's direct-edit path onto `ParentStudentRelationship`, including the Admin-initiated entry into the sole-guardian-removal flow — delegates to `guardianship.service.ts → handleSoleGuardianRemoval` when the edit revokes a sole `MANDATORY_GUARDIAN` relationship, rather than duplicating that logic here. |

#### suspendAccount / restrictAccount

| Field | Detail |
|---|---|
| Signature | `suspendAccount(userId: string, adminId: string, reason: string, restrictionType: 'SUSPENDED'\|'RESTRICTED'): Promise<SuspensionResultDTO>` |
| Purpose | Suspend/restrict any account (UC-78), including a tutor escalated via the `class-delivery-library` 2+/30-day miss rule (FR-MK-003). |
| Side effects | Sets `User.accountStatus`/tutor-equivalent restriction field. If the target is a Tutor with active `CohortMembership` rows, computes `affectedCohortIds` and returns them — this function does **not** itself perform re-matching; it only flags which cohorts `matching-cohorts`' `adminMatching.service.ts → manuallyAssignTutor` will need to process (API spec §2.2). `reason` is an internal note, never disclosed to affected students (UC-34). |
| Edge cases | `affectedCohortIds` is populated only when suspending a Tutor with active cohorts — omitted (not an empty array) for every other case, matching the same "field-presence-signals-meaning" convention used in `revokeOrModifyRelationship`. |

Test file: `tests/services/adminPeople.service.test.ts` — includes the tutor-suspension-flags-cohorts case.

### src/controllers/adminPeople.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listUsers | `adminPeopleService.listUsers(req.query.role, req.query.search, req.query.page, req.query.limit)` | 200 |
| editRelationship | `adminPeopleService.manageRelationshipRecords(req.params.id, req.user.id, req.body)` | 200 |
| suspendAccount | `adminPeopleService.suspendAccount(req.params.userId, req.user.id, req.body.reason, req.body.restrictionType)` | 200 |

### src/routes/adminPeople.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware, requireRole('ADMIN')` | listUsers |
| PATCH | /relationships/:id | `authMiddleware, requireRole('ADMIN')` | editRelationship |
| POST | /:userId/suspend | `authMiddleware, requireRole('ADMIN')` | suspendAccount |

Mounted at `/admin/people`.

---

**Next:** proceed to → [8-3. Backend: Matching & Cohorts]
