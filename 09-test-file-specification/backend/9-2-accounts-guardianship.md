## Project: AKEWTutor — Backend Test Documentation: Accounts & Guardianship
**Links back to:** [05a. Backend Folder & File Structure §2], [8-2. Function-Level Spec: Accounts & Guardianship]
**Conventions:** see `00-api-conventions.md` §0.1–0.7.

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** StudentProfile, ParentProfile, TutorProfile, ParentStudentRelationship, Subject, TutorSubjectRanking, AvailabilitySlot. **Depends on:** `shared-config` (hard).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| studentProfile.service.ts | FR-SP-006–010 | NFR-009 (parent/student scoping) |
| guardianship.service.ts | FR-AC-002–008, FR-SP-001 (invite path) | NFR-009 |
| tutorProfile.service.ts | FR-TU-003, FR-TU-006 | — |
| availability.service.ts | FR-TU-009 | — |
| subject.service.ts | FR-AD-013, NFR-011 (extensible catalog) | — |
| adminTutorVerification.service.ts | FR-TU-004, FR-AD-002 | — |
| adminPeople.service.ts | FR-AD-001 (incl. relationship mgmt), FR-AD-003, FR-MK-003 (escalation intake) | NFR-009, NFR-010 |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/studentProfile.schema.ts | tests/schemas/studentProfile.schema.test.ts | Unit | ☐ |
| src/services/studentProfile.service.ts | tests/services/studentProfile.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/studentProfile.controller.ts | tests/controllers/studentProfile.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/studentProfile.routes.ts | tests/routes/studentProfile.routes.test.ts | Integration (supertest) | ☐ |
| src/schemas/guardianship.schema.ts | tests/schemas/guardianship.schema.test.ts | Unit | ☐ |
| src/services/guardianship.service.ts | tests/services/guardianship.service.test.ts | Unit (mocked Prisma, notification.service) | ☐ |
| src/controllers/guardianship.controller.ts | tests/controllers/guardianship.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/guardianship.routes.ts | tests/routes/guardianship.routes.test.ts | Integration (supertest) | ☐ |
| src/jobs/inviteReminder.job.ts | — | Underlying logic covered by `guardianship.service.test.ts`; interval wrapper excluded per standing convention | — |
| src/schemas/tutorProfile.schema.ts | tests/schemas/tutorProfile.schema.test.ts | Unit | ☐ |
| src/services/tutorProfile.service.ts | tests/services/tutorProfile.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/tutorProfile.controller.ts | tests/controllers/tutorProfile.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/tutorProfile.routes.ts | tests/routes/tutorProfile.routes.test.ts | Integration (supertest) | ☐ |
| src/schemas/availability.schema.ts | tests/schemas/availability.schema.test.ts | Unit | ☐ |
| src/services/availability.service.ts | tests/services/availability.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/availability.controller.ts | tests/controllers/availability.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/availability.routes.ts | tests/routes/availability.routes.test.ts | Integration (supertest) | ☐ |
| src/services/subject.service.ts | tests/services/subject.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/subject.controller.ts | tests/controllers/subject.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/subject.routes.ts | tests/routes/subject.routes.test.ts | Integration (supertest) | ☐ |
| src/services/adminTutorVerification.service.ts | tests/services/adminTutorVerification.service.test.ts | Unit (mocked Prisma, notification.service) | ☐ |
| src/controllers/adminTutorVerification.controller.ts | tests/controllers/adminTutorVerification.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminTutorVerification.routes.ts | tests/routes/adminTutorVerification.routes.test.ts | Integration (supertest) | ☐ |
| src/services/adminPeople.service.ts | tests/services/adminPeople.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/adminPeople.controller.ts | tests/controllers/adminPeople.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminPeople.routes.ts | tests/routes/adminPeople.routes.test.ts | Integration (supertest) | ☐ |

---

### 9.2 Test Case Detail — studentProfile.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Accepts a fully-empty partial update | — | parse `{ body: {} }` | passes — every field optional, per API spec §2.2's partial-update semantics |
| Rejects grade outside 1–12 | — | parse `{ body: { grade: 13 } }` | validation fails |
| Rejects grade 0 | — | parse `{ body: { grade: 0 } }` | validation fails |
| Accepts a valid formatPreference enum value | — | parse `{ body: { formatPreference: 'ONE_TO_THREE' } }` | passes |
| Rejects an invalid formatPreference value | — | parse `{ body: { formatPreference: 'ONE_TO_TWO' } }` | validation fails — only the three documented formats (Section 7) are accepted |
| Rejects a subjectsOfInterest entry that isn't a UUID | — | parse `{ body: { subjectsOfInterest: ['not-a-uuid'] } }` | validation fails |
| Accepts arbitrary keys inside learningSchedulePreference (passthrough) | — | parse `{ body: { learningSchedulePreference: { days: ['MON','WED'] } } }` | passes — schema is intentionally loose here per Doc 8-2's `.passthrough()` |

---

### 9.3 Test Case Detail — studentProfile.service.test.ts

FRs: FR-SP-006–010. **OWASP: A01:2021 – Broken Access Control (parent/student ownership scoping is the central risk in this file).**

#### getProfile

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Student retrieves their own profile | `callerRole: 'STUDENT'`, no `studentId` supplied | call `getProfile(callerId, 'STUDENT')` | resolves the caller's own `StudentProfileDTO`; the caller's own `id` is used as the lookup key, `studentId` param ignored/irrelevant for this role |
| Parent retrieves a linked, ACTIVE student's profile | mock `ParentStudentRelationship.findFirst` → `{ status: 'ACTIVE' }` for `(parentId, studentId)` | call `getProfile(parentId, 'PARENT', studentId)` | resolves the target student's profile |
| Parent attempts to view an unlinked student's profile | mock relationship lookup → `null` | call `getProfile(parentId, 'PARENT', otherStudentId)` | throws `ApiError(403, "Not authorized to view this student's profile")` — canonical IDOR/BOLA test: a parent must not view an arbitrary `studentId` by guessing |
| Parent attempts to view a student via a revoked/non-ACTIVE relationship | mock relationship lookup → `{ status: 'REVOKED' }` | call `getProfile(parentId, 'PARENT', studentId)` | throws the same `ApiError(403, ...)` — a relationship existing historically is not sufficient, it must be `ACTIVE` |
| Parent attempts to view a student under a `GUARDIAN_REQUIRED_HOLD`/`INVITED` (not yet ACTIVE) relationship | mock relationship lookup → `{ status: 'INVITED' }` | call `getProfile(parentId, 'PARENT', studentId)` | throws the same `ApiError(403, ...)` — invited-but-not-activated is not sufficient access either, per FR-AC-003's "invite-management access only" restriction |

#### updateBasicProfile

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid image update | mock ownership check passes | call `updateBasicProfile(callerId, 'STUDENT', callerId, { profilePictureUrl: 'https://.../a.jpg' })` | resolves `{ id, profilePictureUrl }` |
| Invalid image format/size | — | call with a malformed/oversized image reference | throws `ApiError(400, "Unsupported image format or file too large")` |
| Same ownership check as getProfile applies | mock relationship lookup → `null` for an unrelated parent/student pair | call `updateBasicProfile(parentId, 'PARENT', otherStudentId, {...})` | throws `ApiError(403, ...)` — a parent must not be able to *edit* a student's profile picture any more than they can *view* it via a guessed id |

#### updateAcademicProfile

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Partial update touches only supplied fields | mock `prisma.studentProfile.update` | call `updateAcademicProfile(callerId, 'STUDENT', callerId, { grade: 10 })` | assert the Prisma `update` call's `data` object contains only `grade`, not every field reset to `undefined`/default |
| Setting formatPreference does not create a MatchRequest | mock update; spy on any matching-related call (none should exist in this service) | call `updateAcademicProfile(..., { formatPreference: 'ONE_TO_ONE' })` | no `MatchRequest`/matching-service call is made — this function's effect is scoped to the profile row only, per Doc 8-2's explicit note |
| Incomplete profile is accepted without error | mock update with only `preferredLanguage` supplied, no grade/subjects/format yet | call `updateAcademicProfile` | resolves successfully — the "must be complete before matching" rule is enforced elsewhere (`matching-cohorts`), not here |
| Parent updates a linked ACTIVE student's academic profile | mock relationship lookup → ACTIVE | call `updateAcademicProfile(parentId, 'PARENT', studentId, {...})` | resolves successfully |
| Parent updates an unlinked student's academic profile (IDOR) | mock relationship lookup → `null` | call `updateAcademicProfile(parentId, 'PARENT', otherStudentId, {...})` | throws `ApiError(403, ...)` |

---

### 9.4 Test Case Detail — studentProfile.controller.test.ts / studentProfile.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getMyProfile delegates with req.user identity, not a client-suppliable override | mock service | call controller with `req.user = { id, role: 'STUDENT' }`, `req.query.studentId` absent | `getProfile` called with `(req.user.id, req.user.role, undefined)` |
| getMyProfile passes a parent's chosen studentId through for the service's own ownership check | mock service | call controller with `req.user = { id: parentId, role: 'PARENT' }`, `req.query.studentId = 'x'` | `getProfile` called with `(parentId, 'PARENT', 'x')` — confirms the controller does no authorization itself; it is a pure pass-through, and the 403 test lives at the service layer (9.3) |
| updateAcademicProfile propagates 403 | mock service to throw `ApiError(403, ...)` | call controller | error passed through unchanged |
| All three routes require auth | no Authorization header | request `GET /students/me/profile`, `PATCH /students/me/profile`, `PATCH /students/me/academic-profile` | all `401` |
| updateAcademicProfile validates body | mock controller layer, valid token | request with `{ grade: 99 }` | rejected by `validate(updateAcademicProfileSchema)` |

---

### 9.5 Test Case Detail — guardianship.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| addStudentSchema requires grade 1–12 | — | parse `{ body: { grade: 0, inviteContact: 'a@b.com' } }` | fails |
| addStudentSchema rejects missing inviteContact | — | parse `{ body: { grade: 3 } }` | fails |
| inviteGuardianSchema requires a non-empty contact | — | parse `{ body: { inviteContact: '' } }` | fails |
| revokeRelationshipSchema defaults revoke to false | — | parse `{ body: {} }` | passes; `revoke === false` |

---

### 9.6 Test Case Detail — guardianship.service.test.ts

FRs: FR-AC-001–008. **OWASP: A01:2021 – Broken Access Control, A04:2021 – Insecure Design (invite-token predictability/expiry is a security-relevant design choice here).**

#### addStudentAndInvite

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Grade 1–5 succeeds | — | call `addStudentAndInvite(parentId, 3, contact)` | creates a placeholder `StudentProfile` + `ParentStudentRelationship(status: INVITED, relationshipType: MANDATORY_GUARDIAN)`, `inviteExpiresAt` = now + 14 days; dispatch call made |
| Grade 6–12 rejected on this path | — | call `addStudentAndInvite(parentId, 9, contact)` | throws `ApiError(400, "Grades 6–12 students register independently — see /auth/register/student")` — confirms the routing rule is enforced on the parent-initiated side too, not just the student self-registration side (9-1's `registerUser` test) |
| Invite expiry is exactly 14 days from creation | freeze/mock the clock | call `addStudentAndInvite` | assert `inviteExpiresAt` equals `now + 14d`, not some other window |
| Invite dispatched to the contact method actually provided | — | call with an email-shaped `inviteContact`, then a phone-shaped one | dispatch routed via the corresponding channel each time |

#### resendOrRegenerateInvite

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Owner resends before activation | mock relationship found, owned by caller, `status: INVITED` | call `resendOrRegenerateInvite(parentId, relationshipId)` | `inviteExpiresAt` reset to now + 14 days (a fresh 14-day window, not an extension of the old one) |
| Non-owner attempts to resend (IDOR) | mock relationship found, `parentId` does not match caller | call `resendOrRegenerateInvite(otherParentId, relationshipId)` | throws `ApiError(403, "Not authorized to manage this invite")` |
| Already-activated relationship | mock relationship found, `status: ACTIVE` | call `resendOrRegenerateInvite(parentId, relationshipId)` | throws `ApiError(409, "This student has already activated their account")` |
| Repeated resend keeps resetting the window (no cap documented) | mock relationship found | call `resendOrRegenerateInvite` three times in a row | each call resets to a fresh +14 days — flagged as intentionally unlimited per Doc 8-2/FR-AC-003, not a bug; if abuse-rate-limiting is desired later it is an explicit open item, not silently assumed here |

#### activateInvite

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid, unexpired token | mock a matching `ParentStudentRelationship` with `inviteExpiresAt` in the future | call `activateInvite(token, password)` | creates the `User` row (role `STUDENT`), hashes the password, sets `status: ACTIVE, activatedAt`, resolves `{ accessToken, studentId, relationshipStatus: 'ACTIVE' }` |
| Expired token | mock a match with `inviteExpiresAt` in the past | call `activateInvite(expiredToken, password)` | throws `ApiError(400, "This invite is no longer valid — ask your parent/guardian to resend it")` |
| Unknown token | mock no match found | call `activateInvite(bogusToken, password)` | throws `ApiError(404, "Invite not found")` |
| Token cannot be reused after activation | mock a token already `ACTIVE` | call `activateInvite(sameToken, password)` again | rejected — either the 404 (token consumed/rotated on activation) or a dedicated already-activated message; whichever the implementer chooses, the test asserts a second activation attempt with the same token never succeeds a second time (prevents a leaked/observed invite link from being replayed) |
| Password is hashed before persistence | mock a valid activation | call `activateInvite(token, "plainpass123")` | assert the `User.create` call's password field is a bcrypt hash, not the literal string |
| Unlocks full parent functionality (relationship status observable) | mock a valid activation | call `activateInvite` | resolves `relationshipStatus: 'ACTIVE'` — the value the parent-side UI/other endpoints key off of to unlock full functionality (FR-AC-004) |

#### inviteOptionalGuardian

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Grade 6–12 student invites a guardian | mock caller's `StudentProfile.grade = 9` | call `inviteOptionalGuardian(studentId, contact)` | creates `ParentStudentRelationship(relationshipType: OPTIONAL_GUARDIAN, status: INVITED)` |
| Grade 1–5 student attempts to invite (should be structurally impossible, but service defends anyway) | mock caller's `StudentProfile.grade = 3` | call `inviteOptionalGuardian(studentId, contact)` | throws `ApiError(403, "This action is only available to Grade 6–12 students")` — defense in depth, since a Grade 1–5 student normally has no independent account to call this from in the first place |

#### revokeOrModifyRelationship / handleSoleGuardianRemoval

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Guardian revokes their own relationship (not sole mandatory) | mock student has 2 `MANDATORY_GUARDIAN` relationships, caller revokes one | call `revokeOrModifyRelationship(parentId, 'PARENT', relationshipId, { revoke: true })` | relationship set to a revoked/inactive state; `studentAccountStatus` key **omitted** from the response entirely (not `null`) — this was not the sole guardian |
| Guardian revokes the sole mandatory relationship — triggers hold | mock student has exactly 1 `MANDATORY_GUARDIAN` relationship, caller revokes it | call `revokeOrModifyRelationship(parentId, 'PARENT', relationshipId, { revoke: true })` | delegates to `handleSoleGuardianRemoval`; `StudentProfile.accountStatus` set to `GUARDIAN_REQUIRED_HOLD`; response includes `studentAccountStatus: 'GUARDIAN_REQUIRED_HOLD'` |
| Sole-guardian removal preserves all student data | mock the above | call `revokeOrModifyRelationship` | assert no delete call is made against `StudentProfile`, XP ledger, recordings, or any other student-owned data — FR-AC-008's explicit "no data deleted" guarantee |
| Grade 1–5 student attempts to revoke their own mandatory guardian | mock caller is the student (role STUDENT), grade 1–5 | call `revokeOrModifyRelationship(studentId, 'STUDENT', relationshipId, { revoke: true })` | throws `ApiError(403, "Only a guardian or Admin can remove this relationship")` |
| Grade 6–12 student revokes a relationship they did not initiate | mock the relationship's `initiatedBy` is the guardian, not the student | call `revokeOrModifyRelationship(studentId, 'STUDENT', relationshipId, { revoke: true })` | throws `ApiError(403, "Not authorized to modify this relationship")` |
| Grade 6–12 student revokes a relationship they *did* initiate | mock `initiatedBy === studentId` | call `revokeOrModifyRelationship(studentId, 'STUDENT', relationshipId, { revoke: true })` | succeeds — FR-AC-007's explicit student-initiated exception |
| Unrelated caller attempts to revoke a relationship entirely unconnected to them (IDOR) | mock caller has no relation to the target relationship at all | call `revokeOrModifyRelationship(randomUserId, 'PARENT', someOtherRelationshipId, {...})` | throws `ApiError(403, ...)` — a parent must not modify a relationship by guessing another family's relationship id |
| Modify-permissions-only (no revoke) never triggers the hold flow | mock `{ revoke: false, permissions: {...} }` on any relationship, including a sole-guardian one | call `revokeOrModifyRelationship` | `handleSoleGuardianRemoval` is **not** invoked; only `permissions` is updated |

---

### 9.7 Test Case Detail — guardianship.controller.test.ts / guardianship.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| addStudent, resendInvite, inviteGuardian, revokeRelationship all require auth | no Authorization header | request each | all `401` |
| activateInvite is deliberately public (token is the credential) | mock controller layer, no Authorization header | request `POST /guardianship/invites/:token/activate` with a valid body | reaches the controller mock, not `401` — confirmed intentional per Doc 8-2 |
| activateInvite validates password shape | mock controller layer | request with `{ password: "short" }` | rejected if a min-length rule applies (cross-check against `auth.schema.ts`'s `min(8)` pattern — flag to implementer if `activateInvite`'s own schema doesn't enforce this, since Doc 8-2 doesn't explicitly list a schema for it) |
| revokeRelationship passes req.user through, never trusts a client-supplied caller id | mock service | call controller | `revokeOrModifyRelationship` called with `(req.user.id, req.user.role, req.params.id, req.body)` — `req.params.id` is the *relationship* id, never confused with a user id |

---

### 9.8 Test Case Detail — tutorProfile.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| rankSubjectsSchema accepts exactly 2 uniquely-ranked subjects | — | parse `{ body: { subjects: [{subjectId: A, rank: 1}, {subjectId: B, rank: 2}] } }` | passes |
| rankSubjectsSchema accepts exactly 1 subject | — | parse `{ body: { subjects: [{subjectId: A, rank: 1}] } }` | passes |
| rankSubjectsSchema rejects 3 subjects (schema-level cap) | — | parse an array of 3 | fails — `.max(2)` |
| rankSubjectsSchema rejects duplicate subjectId | — | parse `[{subjectId: A, rank: 1}, {subjectId: A, rank: 2}]` | fails — the "each subject may be ranked once" refine |
| rankSubjectsSchema rejects duplicate rank | — | parse `[{subjectId: A, rank: 1}, {subjectId: B, rank: 1}]` | fails — the "ranks must be unique" refine |
| rankSubjectsSchema rejects rank values outside {1,2} | — | parse `[{subjectId: A, rank: 3}]` | fails |
| updateTutorProfileSchema does not accept a verificationStatus field (mass-assignment guard) | — | parse `{ body: { bio: "...", verificationStatus: "VERIFIED" } }` | the unknown `verificationStatus` key is stripped (default Zod object behavior) or rejected if `.strict()` — this is the schema-level companion to the service-level guard in 9.9 below (OWASP A08 — mass assignment / privilege field smuggling) |

---

### 9.9 Test Case Detail — tutorProfile.service.test.ts

FRs: FR-TU-003, FR-TU-006. **OWASP: A01:2021 – Broken Access Control, A08:2021 – Software and Data Integrity Failures (mass-assignment onto verificationStatus).**

#### getProfile / updateProfile

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns the tutor's own profile | mock `prisma.tutorProfile.findUnique` | call `getProfile(tutorId)` | resolves `TutorProfileDTO` |
| updateProfile applies only editable fields | mock update; spy on call args | call `updateProfile(tutorId, { bio: "new bio", verificationStatus: "VERIFIED" })` | assert the persisted update excludes `verificationStatus` entirely — a tutor must never self-approve their own verification by smuggling the field into a profile-edit payload, even if it slipped past schema stripping (defense in depth alongside 9.8's schema case) |

#### rankSubjects

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Ranks exactly 2 subjects successfully | mock `$transaction([deleteMany, createMany])` | call `rankSubjects(tutorId, [{subjectId:A,rank:1},{subjectId:B,rank:2}])` | resolves both rankings; transaction called with delete-then-create in one array |
| Third subject rejected at the service layer even if schema were bypassed | call the service function directly with a 3-item array (simulating a relaxed/future schema) | call `rankSubjects(tutorId, [3 items])` | throws `ApiError(400, "A tutor may rank a maximum of two subjects")` — the authoritative business rule Doc 8-2 says must survive independent of the schema |
| Duplicate subjectId/rank rejected at the service layer | same bypass scenario | call `rankSubjects(tutorId, [dup subjectId or dup rank])` | throws `ApiError(400, "Each subject may be ranked once, and ranks must be unique")` |
| Full replace is atomic | spy on `$transaction` | call `rankSubjects(tutorId, [...])` for a tutor who already has 2 existing rankings | assert both the delete of old rankings and create of new ones occur inside a single `$transaction` call — never two separate, non-atomic Prisma calls that could leave a partial/duplicate state on failure |
| Transaction failure leaves no partial state (mocked rejection) | mock `$transaction` to reject | call `rankSubjects(tutorId, [...])` | the promise rejects; assert no follow-up "cleanup" call was made — the atomicity guarantee is the DB transaction itself, not application-level compensation |
| Ranked subject requires no grade-range input | — | call `rankSubjects` with a normal 2-subject payload | resolves without any grade-related field being read/required — confirms there is no per-rank grade-range concept anywhere in this function, per Doc 02 §6.2/FR-TU-006 |

---

### 9.10 Test Case Detail — tutorProfile.controller.test.ts / tutorProfile.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 3 routes require auth | no Authorization header | request `GET/PATCH /tutors/me/profile`, `PUT /tutors/me/subjects` | all `401` |
| rankSubjects validates the 2-max/unique-rank rule at the route layer | mock controller layer | request `PUT /tutors/me/subjects` with 3 subjects | rejected by `validate(rankSubjectsSchema)`, controller never called |
| updateProfile controller passes only req.body through, id from req.user | mock service | call controller | `updateProfile` called with `(req.user.id, req.body)` — never a client-suppliable tutor id from params/body that could target another tutor's profile |

---

### 9.11 Test Case Detail — availability.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| createSlotSchema requires dayOfWeek when isRecurring is true | — | parse `{ body: { isRecurring: true, startTime: t1, endTime: t2 } }` (no dayOfWeek) | fails |
| createSlotSchema allows omitted dayOfWeek when isRecurring is false | — | parse `{ body: { isRecurring: false, dayOfWeek: undefined, startTime: t1, endTime: t2 } }` | passes |
| createSlotSchema rejects endTime before/equal to startTime | — | parse `{ body: { startTime: "2026-01-01T10:00:00Z", endTime: "2026-01-01T09:00:00Z" } }` | fails |
| createSlotSchema rejects dayOfWeek outside 0–6 | — | parse `{ body: { dayOfWeek: 7, isRecurring: true, startTime: t1, endTime: t2 } }` | fails |
| deleteSlotSchema requires a uuid slotId param | — | parse `{ params: { slotId: "not-a-uuid" } }` | fails |

---

### 9.12 Test Case Detail — availability.service.test.ts

FRs: FR-TU-009. **OWASP: A01:2021 – Broken Access Control.**

#### setSlots / listSlots

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates a valid slot | — | call `setSlots(tutorId, { dayOfWeek: 1, startTime, endTime, isRecurring: true })` | resolves the created `AvailabilitySlotDTO` |
| Rejects endTime before startTime at the service layer too | — | call `setSlots(tutorId, { startTime: later, endTime: earlier, ... })` | throws `ApiError(400, "End time must be after start time")` — authoritative re-check independent of schema |
| listSlots returns only the calling tutor's own slots | mock `findMany` scoped by `tutorId` | call `listSlots(tutorId)` | assert the Prisma filter includes `tutorId` — a query without it would leak every tutor's schedule |

#### removeSlot

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Owner removes their own unused slot | mock slot found, owned by caller, no dependent confirmed session | call `removeSlot(tutorId, slotId)` | resolves `{ id, deleted: true }` |
| Non-owner attempts removal (IDOR) | mock slot found, `tutorId` on the slot ≠ caller | call `removeSlot(otherTutorId, slotId)` | throws `ApiError(403, "Not authorized to remove this slot")` |
| Slot in use by a confirmed session | mock a dependent confirmed `ScheduledSession` overlapping this slot's tutor/time-window | call `removeSlot(tutorId, slotId)` | throws `ApiError(409, "This slot is in use by a confirmed session and cannot be removed until it is resolved")` — the slot-in-use case Doc 8-2 explicitly flags |
| Slot with no dependent session removes cleanly even if other unrelated slots exist | mock other, unrelated confirmed sessions for the same tutor on different slots | call `removeSlot(tutorId, targetSlotId)` | succeeds — confirms the in-use check is scoped to *this* slot's overlap, not "does this tutor have any confirmed sessions at all" |

---

### 9.13 Test Case Detail — availability.controller.test.ts / availability.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 3 routes require auth | no Authorization header | request each | `401` |
| removeSlot propagates 409 unchanged | mock service to throw `ApiError(409, ...)` | call controller | error passed through, not rewritten as a generic 400 |
| addSlot validates body | mock controller layer | request with `endTime` before `startTime` | rejected by `validate(createSlotSchema)` |

---

### 9.14 Test Case Detail — subject.service.test.ts

FRs: FR-AD-013. NFRs: NFR-011.

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listSubjects returns only active subjects by default | mock `findMany({ where: { isActive: true } })` | call `listSubjects(false)` | resolves only active subjects |
| listSubjects with includeInactive returns all | mock `findMany` unfiltered by `isActive` | call `listSubjects(true)` | resolves both active and inactive subjects — for Admin use |
| createSubject succeeds for a new name | mock `create` | call `createSubject("Biology")` | resolves the new `SubjectDTO` |
| createSubject rejects a duplicate name | mock `create` to reject with a unique-constraint error | call `createSubject("Mathematics")` (already exists) | throws `ApiError(409, "A subject with this name already exists")` |
| deactivateSubject flips isActive without deleting | spy on Prisma calls | call `deactivateSubject(id, false)` | assert an `update` call was made (`isActive: false`), and no `delete` call was ever issued — historical matching/teaching data referencing this subject must remain intact (Doc 04 §4.4) |
| Reactivating a previously deactivated subject | mock update | call `deactivateSubject(id, true)` | resolves `{ id, isActive: true }` |

---

### 9.15 Test Case Detail — subject.controller.test.ts / subject.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| GET /subjects is public | mock controller layer, no Authorization header | request `GET /subjects` | `200` |
| POST /admin/subjects requires Admin | no Authorization header, then a Student token | request `POST /admin/subjects` | `401` then `403` |
| PATCH /admin/subjects/:id requires Admin | same pattern | request `PATCH /admin/subjects/x` | `401` then `403` |
| Public GET never exposes an admin-only mutate path by accident | mock controller layer | request `POST /subjects` (no `/admin` prefix) | `404`/method-not-allowed — the mutate routes exist only under `/admin/subjects`, confirming the router mounts are genuinely split, not just documentation |

---

### 9.16 Test Case Detail — adminTutorVerification.service.test.ts

FRs: FR-TU-004, FR-AD-002.

#### listPendingTutors

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Lists only PENDING tutors, paginated | mock `findMany({ where: { verificationStatus: 'PENDING' } })` | call `listPendingTutors(1, 20)` | resolves paginated pending tutors only — a `VERIFIED` or `REJECTED` tutor never appears in this queue |

#### approveTutor / rejectTutor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Approve a pending tutor | mock tutor found, `verificationStatus: PENDING` | call `approveTutor(tutorId, adminId)` | sets `verificationStatus: VERIFIED, verifiedAt, verifiedById: adminId`; dispatch notification sent to the tutor |
| Reject a pending tutor with a reason | mock tutor found, `PENDING` | call `rejectTutor(tutorId, adminId, "Incomplete credentials")` | sets `verificationStatus: REJECTED`; the reason is persisted as an internal note field only |
| Rejected reason never leaks to the tutor-facing notification | mock reject; spy on `dispatchNotification`'s payload | call `rejectTutor(tutorId, adminId, "Suspicious ID document")` | assert the dispatched notification payload does **not** contain the literal reason string — only a generic rejection message, per API spec §2.2's explicit non-disclosure rule |
| Approving an already-reviewed tutor | mock tutor found, `verificationStatus: VERIFIED` already | call `approveTutor(tutorId, adminId)` | throws `ApiError(409, "This tutor has already been reviewed")` |
| Rejecting an already-rejected tutor | mock tutor found, `verificationStatus: REJECTED` already | call `rejectTutor(tutorId, adminId, "...")` | throws the same `ApiError(409, ...)` |
| Approval is what makes a tutor matchable (integration note, tested at unit level via the flag) | mock approve | call `approveTutor(tutorId, adminId)` | assert the persisted `verificationStatus: VERIFIED` — the actual matchability behavior is verified in `matching-cohorts`' own test suite (9-3), this test only confirms the flag this feature is responsible for setting |

---

### 9.17 Test Case Detail — adminTutorVerification.controller.test.ts / adminTutorVerification.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 3 routes require Admin | no Authorization header, then a Tutor token | request `GET /admin/tutors/pending`, `POST /admin/tutors/:id/approve`, `POST /admin/tutors/:id/reject` | `401` then `403` for each |
| approve/reject pass req.user.id as adminId, never client-suppliable | mock service | call controller | `approveTutor`/`rejectTutor` called with `req.user.id` as the admin actor |

---

### 9.18 Test Case Detail — adminPeople.service.test.ts

FRs: FR-AD-001 (incl. relationship mgmt), FR-AD-003, FR-MK-003 (intake). **OWASP: A01:2021 – Broken Access Control.**

#### listUsers

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Filters by role | mock `findMany({ where: { role: 'TUTOR' } })` | call `listUsers('TUTOR', undefined, 1, 20)` | resolves only tutors |
| Search term filters across name/email/phone | mock `findMany` with a search `OR` clause | call `listUsers(undefined, "amanuel", 1, 20)` | resolves matching users |
| No filters returns all users, paginated | mock `findMany` unfiltered | call `listUsers(undefined, undefined, 1, 20)` | resolves a paginated full list |

#### manageRelationshipRecords

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Admin edits a relationship directly | mock relationship found | call `manageRelationshipRecords(relationshipId, adminId, { permissions: {...} })` | resolves the updated `RelationshipResultDTO` |
| Admin-initiated sole-guardian revocation delegates correctly | mock a sole `MANDATORY_GUARDIAN` relationship, `input.status` set to a revoked value | call `manageRelationshipRecords(relationshipId, adminId, { status: 'REVOKED' })` | delegates to `guardianship.service.handleSoleGuardianRemoval` rather than duplicating the hold-state logic locally — assert the shared function was actually called, not reimplemented inline (a maintenance/consistency check, not just a behavior check) |

#### suspendAccount / restrictAccount

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Suspend a Student/Parent (no cohort side effects) | mock target user has no active `CohortMembership`/`TutorProfile` | call `suspendAccount(userId, adminId, "Policy violation", 'SUSPENDED')` | sets `accountStatus`; response **omits** `affectedCohortIds` entirely (not an empty array) |
| Suspend a Tutor with active cohorts | mock target is a Tutor with 2 active `CohortMembership`-bearing cohorts | call `suspendAccount(tutorId, adminId, reason, 'SUSPENDED')` | sets the tutor's restriction field; response includes `affectedCohortIds: [id1, id2]` |
| Suspend a Tutor with zero active cohorts | mock target is a Tutor, no active cohorts | call `suspendAccount(tutorId, adminId, reason, 'SUSPENDED')` | response **omits** `affectedCohortIds` — same field-presence convention as the sole-guardian case elsewhere in this doc |
| This function never itself re-matches students | mock a tutor suspension with affected cohorts; spy on any matching-service import/call | call `suspendAccount` | assert no `cohort.service`/`adminMatching.service` function was called directly — this function only *flags* `affectedCohortIds` for `matching-cohorts` to consume, per Doc 8-2's explicit boundary |
| Suspension reason never leaks to affected students | mock a suspension; spy on any notification dispatched to the affected students (via a hypothetical downstream call, or confirm none is made from this function directly) | call `suspendAccount(tutorId, adminId, "Repeated no-shows and a complaint on file", 'SUSPENDED')` | the internal `reason` string is not present in anything this function itself dispatches to non-Admin recipients |
| restrictAccount (lesser action) is distinguishable from suspendAccount | mock a `RESTRICTED` call | call `suspendAccount(userId, adminId, reason, 'RESTRICTED')` | persisted status reflects `RESTRICTED`, not conflated with a full `SUSPENDED` state |

---

### 9.19 Test Case Detail — adminPeople.controller.test.ts / adminPeople.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 3 routes require Admin | no Authorization header, then a Parent token | request `GET /admin/people`, `PATCH /admin/people/relationships/:id`, `POST /admin/people/:userId/suspend` | `401` then `403` for each |
| suspendAccount passes req.user.id as adminId | mock service | call controller | `suspendAccount` called with `(req.params.userId, req.user.id, req.body.reason, req.body.restrictionType)` |

---

### 9.20 Coverage Honesty Check (per PR Steward, at review time)

- [ ] Every ownership-scoped read/write in this feature (`studentProfile.getProfile`/`updateBasicProfile`/`updateAcademicProfile`, `guardianship.resendOrRegenerateInvite`/`revokeOrModifyRelationship`, `availability.removeSlot`) has an explicit IDOR test where the caller is *not* the owner and *not* an authorized parent — not just a happy-path "owner succeeds" test.
- [ ] The sole-guardian-hold case asserts both the state transition (`GUARDIAN_REQUIRED_HOLD`) **and** the absence of any delete call — a test that only checks the status field would miss a regression that also (incorrectly) purges data.
- [ ] The "field omitted vs. present" convention (`studentAccountStatus`, `affectedCohortIds`) is asserted via `expect(result).not.toHaveProperty(...)`, not `expect(result.field).toBeUndefined()` — the latter passes even if the key exists with an explicit `undefined` value, which is a different wire shape once JSON-serialized (a `JSON.stringify` of `{ x: undefined }` drops the key, but this distinction is worth testing explicitly rather than assuming).
- [ ] `rejectTutor`'s "reason never leaks" case actually inspects the mocked `dispatchNotification` call's payload contents, not just that a notification was sent.
- [ ] `tutorProfile.updateProfile`'s mass-assignment guard against `verificationStatus` is tested by including that field in the input and asserting it's absent from the persisted update — not by only testing legitimate fields and assuming the illegitimate one is naturally excluded.
- [ ] `rankSubjects`'s atomic-replace is tested against the actual `$transaction` call shape, not just the end result — a non-atomic two-call implementation could produce the same final state in tests while being unsafe under concurrent/partial-failure conditions in production.

---

### 9.21 Out of Scope for Automated Testing (and why)

- **Real image upload/validation for `profilePictureUrl`** (`updateBasicProfile`) — unit tests exercise the business-rule branch (valid vs. invalid format/size) against mocked inputs; actual file-upload handling, storage, and virus/content scanning (if any) is a separate concern not specified in Docs 01–08 for this feature.
- **`inviteReminder.job.ts`'s cron scheduling itself** — the day-7 reminder *logic* is covered via `guardianship.service.test.ts`, but the interval registration and idempotency column (`reminderSentAt`, flagged as an implementation detail in Doc 8-2) needs a manual/staging check once that column is actually named and added to the schema.
- **Real Prisma referential-integrity behavior** for `removeSlot`'s in-use check — the overlap detection between `AvailabilitySlot` and `ScheduledSession` has no direct FK per Doc 04, so the unit test mocks the resolved overlap result; a live-database query-correctness pass is tracked separately.
- **Rate limiting on `resendOrRegenerateInvite`** — flagged, not tested: Doc 8-2 documents no cap on resend frequency; noted here as the same class of open item as 9-1's login/verification rate-limiting gap (OWASP A04).

---

**Next:** proceed to → [9-3. Backend Test Documentation: Matching & Cohorts]
