## Project: AKEWTutor — Backend Function-Level Spec: Class Delivery, Recording & Library
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.4 (recording-missing escalation and payment-pause reschedule are job/event-driven, exposed here only as read state) and §0.5 (Cloudflare R2, Jitsi integration shapes). **API reference:** `04-class-delivery-library-api.md`. **Folder/file reference:** `05a-backend-structure.md` §4.

**Owns:** ScheduledSession, RescheduleRequest, SessionMiss, RecordingConsent, Recording, LibraryMaterial, WeeklyAssessment. **Depends on:** `matching-cohorts` (hard) — every session belongs to a confirmed `Cohort`.

**Links back to:** [06-api/04-class-delivery-library-api.md], [05a. Backend Folder & File Structure §4]
**Links forward to:** [9-4. Backend Test Spec: Class Delivery, Recording & Library]

---

### src/schemas/session.schema.ts (new)

| Schema | Shape |
|---|---|
| provideJitsiLinkSchema | `z.object({ body: z.object({ jitsiLinkUrl: z.string().url() }) })` |

### src/services/session.service.ts (new)

#### generateSessionsForCohort

| Field | Detail |
|---|---|
| Signature | `generateSessionsForCohort(cohortId: string): Promise<{ created: number }>` |
| Purpose | Not a client-facing endpoint — called once a `Cohort` reaches `PENDING_PAYMENT`/is confirmed. First sets `Cohort.sessionsPerWeek` from the count of distinct recurring `AvailabilitySlot` rows matched into this Cohort's schedule (Doc 02 §7 v3.2 callout, Doc 04 `Cohort.sessionsPerWeek`), then generates `sessionsPerWeek × 4` recurring `ScheduledSession` rows (one 28-day billing cycle) from those same matched slots. |
| Side effects | Sets `Cohort.sessionsPerWeek` (once, if not already set). Bulk `createMany` of `ScheduledSession(status: SCHEDULED)` rows for the cohort's 28-day billing cycle — `sessionsPerWeek × 4` rows per cycle, one call per cycle rollover. |
| Edge cases | Must not double-generate if called twice for the same cohort/cycle — guarded by checking for existing sessions in the target window before inserting. Must not overwrite an already-set `sessionsPerWeek` on a re-run (e.g. a retried call after a partial failure) — cadence is frozen at first successful confirmation per Doc 04. |

Test file: `tests/services/session.service.test.ts`

#### provideJitsiLink

| Field | Detail |
|---|---|
| Signature | `provideJitsiLink(tutorId: string, sessionId: string, jitsiLinkUrl: string): Promise<{ id, jitsiLinkUrl, jitsiLinkSentAt, providedLateNotice }>` |
| Purpose | Tutor submits the session's video link — AKEWTutor stores/delivers the URL but never calls a Jitsi API itself (§0.5). |
| Throws | `ApiError(403, "Not authorized to provide a link for this session")` — caller is not the session's assigned tutor. |
| Side effects | Sets `jitsiLinkUrl`, `jitsiLinkSentAt`. |
| Edge cases | `providedLateNotice: true` when submitted with under 30 minutes remaining before `scheduledStart` — this does **not** block submission (the student still needs the link), but the flag is retained for `sessionMiss.service.ts → recordTutorCausedMiss` to reference if the class ends up disrupted as a result (API spec §4.2). |

Test file: `tests/services/session.service.test.ts` — includes the ≥30-min-before boundary case, both sides.

#### markCompleted

| Field | Detail |
|---|---|
| Signature | `markCompleted(tutorId: string, sessionId: string): Promise<{ id, status: 'COMPLETED' }>` |
| Throws | `ApiError(409, "This session's status cannot be changed")` — already `COMPLETED`, `MISSED`, or `CANCELLED`. |

Test file: `tests/services/session.service.test.ts`

### src/controllers/session.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listMySessions | direct paginated Prisma read, scoped to caller's cohort memberships/tutor assignments | 200 |
| provideLink | `sessionService.provideJitsiLink(req.user.id, req.params.sessionId, req.body.jitsiLinkUrl)` | 200 |
| getSession | direct read, ownership-checked | 200 |
| completeSession | `sessionService.markCompleted(req.user.id, req.params.sessionId)` (co-located, per API spec §4.1's `POST /sessions/:sessionId/complete`) | 200 |

| Field | Detail |
|---|---|
| listMySessions throws | none beyond common validation |
| listMySessions edge case | No sessions yet (student still matching) → `sessions: []`, not an error. |
| getSession throws | `ApiError(403, "Not authorized to view this session")` — caller not a member/tutor of the session's cohort. |
| getSession edge case | `status: "PAYMENT_PAUSE_RESCHEDULED"` is a valid, reportable value here (§0.4) — this endpoint never triggers that state, only reports it. |

Test file: covered by `tests/services/session.service.test.ts`.

### src/routes/session.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware` | listMySessions |
| GET | /:sessionId | `authMiddleware` | getSession |
| POST | /:sessionId/link | `authMiddleware, validate(provideJitsiLinkSchema)` | provideLink |
| POST | /:sessionId/complete | `authMiddleware` | completeSession |

Mounted at `/sessions`.

---

### src/schemas/recordingConsent.schema.ts (new)

| Schema | Shape |
|---|---|
| acknowledgeConsentSchema | `z.object({ body: z.object({ tutorId: z.string().uuid(), studentId: z.string().uuid() }) })` |

### src/services/recordingConsent.service.ts (new)

#### getConsentStatus

| Field | Detail |
|---|---|
| Signature | `getConsentStatus(tutorId: string, studentId: string): Promise<RecordingConsentDTO>` |
| Output | `{ tutorId, studentId, tutorAcknowledgedAt, studentOrParentAcknowledgedAt, consentComplete }` |

#### acknowledgeAsStudentOrParent / acknowledgeAsTutor

| Field | Detail |
|---|---|
| Signature | `acknowledgeAsStudentOrParent(callerId, tutorId, studentId): Promise<RecordingConsentDTO>` · `acknowledgeAsTutor(callerId, tutorId, studentId): Promise<RecordingConsentDTO>` |
| Purpose | One-time consent acknowledgment per tutor–student pairing (not per session) — the controller dispatches to whichever function matches `req.user.role`. |
| Side effects | Upserts the `RecordingConsent` row for the pairing, setting the caller-side timestamp. `consentComplete` flips `true` only once both sides are set. Blocks the first recorded session for a pairing until both sides acknowledge (enforced downstream in `recording.service.ts → uploadRecording`, not here — this function only records the acknowledgment). |

Test file: `tests/services/recordingConsent.service.test.ts` — includes the first-session-blocked-without-consent case (verified via the upload path, not this function directly).

### src/controllers/recordingConsent.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getStatus | `recordingConsentService.getConsentStatus(req.query.tutorId, req.query.studentId)` | 200 |
| acknowledge | `recordingConsentService.acknowledgeAsStudentOrParent` or `acknowledgeAsTutor`, branched on `req.user.role` | 200 |

### src/routes/recordingConsent.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /status | `authMiddleware` | getStatus |
| POST | /acknowledge | `authMiddleware, validate(acknowledgeConsentSchema)` | acknowledge |

Mounted at `/recording-consent`.

---

### src/utils/providers/storage.client.ts (new)

Thin wrapper around Cloudflare R2 (§0.5). Shared contract:

```typescript
interface StorageClient {
  upload(key: string, file: Buffer, contentType: string): Promise<{ storageKey: string }>;
  getSignedUrl(storageKey: string, expiresInSeconds: number): Promise<string>;
}
```

Used by `recording.service.ts` and `library.service.ts`. Test file mocks the underlying R2 SDK call.

### src/services/recording.service.ts (new)

#### uploadRecording

| Field | Detail |
|---|---|
| Signature | `uploadRecording(tutorId: string, sessionId: string, file: Buffer): Promise<RecordingDTO>` |
| Purpose | Tutor uploads a session recording; auto-routes it to the correct student's Library — no manual filing step. |
| Throws | `ApiError(403, "Not authorized to upload a recording for this session")` — caller not the session's assigned tutor. `ApiError(409, "Recording consent must be acknowledged by both parties for every active student before this session can be recorded")` — blocked unless `recordingConsent.service.ts → getConsentStatus` returns `consentComplete: true` for **every currently-`ACTIVE` `CohortMembership`'s pairing** on this session's Cohort, not just one. For a 1-to-1 Cohort this is the single pairing (only relevant for that pairing's first session); for a 1-to-3/1-to-5 Cohort this is every active member's individual pairing, re-checked at each upload (a mid-cycle new joiner whose consent isn't yet complete blocks future uploads for the whole class, not just their own access to past ones). |
| Side effects | Encodes/stores at 720p via `storage.client.ts`; sets `expiresAt` to +90 days from upload; `keepPermanently: false` by default. |

Test file: `tests/services/recording.service.test.ts` — includes the consent-gate case and a cross-student-access-denied case (tested via `getSignedUrl` below).

#### getSignedUrl

| Field | Detail |
|---|---|
| Signature | `getSignedUrl(callerId: string, recordingId: string): Promise<{ signedUrl: string; expiresIn: number }>` |
| Throws | `ApiError(403, "Not authorized to access this recording")` — caller was never a member of this recording's cohort. `ApiError(404, "Recording no longer available")` — past retention and not `keepPermanently`, or `deletedAt` set. This **is** treated as not-found, distinct from an empty list, since the object is genuinely gone (§0.3). |
| Side effects | `storage.client.ts → getSignedUrl` — short-lived URL (900s), never a direct client-to-R2 connection. |

Test file: `tests/services/recording.service.test.ts` — includes the cross-student-access-denied case explicitly (a student who was never a member of the recording's cohort, even with a guessed valid UUID).

#### keepPermanently

| Field | Detail |
|---|---|
| Signature | `keepPermanently(callerId: string, recordingId: string): Promise<{ id, keepPermanently: true }>` |
| Throws | `ApiError(403, "Not authorized to modify this recording")` — same ownership rule as `getSignedUrl`. |

Test file: `tests/services/recording.service.test.ts`

#### flagMissing / escalateMissing

| Field | Detail |
|---|---|
| Signature | `flagMissing(sessionId: string): Promise<void>` · `escalateMissing(sessionId: string): Promise<void>` |
| Purpose | Called by `recordingMissingCheck.job.ts` — not a client-facing endpoint. |
| Side effects | Sets `ScheduledSession.recordingStatus: MISSING` at 2h post-`scheduledEnd`, `ESCALATED` at 24h. |

Test file: `tests/services/recording.service.test.ts`

### src/controllers/recording.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| upload | `recordingService.uploadRecording(req.user.id, req.body.sessionId, req.file)` | 201 |
| getMyRecordings | direct paginated read, excludes expired-and-not-kept rows | 200 |
| getSignedUrl | `recordingService.getSignedUrl(req.user.id, req.params.recordingId)` | 200 |
| keepPermanently | `recordingService.keepPermanently(req.user.id, req.params.recordingId)` | 200 |

| Field | Detail |
|---|---|
| getMyRecordings edge case | A recording past `expiresAt` and not `keepPermanently` is excluded from the list entirely — expected expiry, not an error (UC-47 alternate flow, API spec §4.2). |

### src/routes/recording.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | / | `authMiddleware` (multipart) | upload |
| GET | /me | `authMiddleware` | getMyRecordings |
| GET | /:recordingId/signed-url | `authMiddleware` | getSignedUrl |
| POST | /:recordingId/keep-permanently | `authMiddleware` | keepPermanently |

Mounted at `/recordings`. Access is scoped to the caller's own cohort membership at the service layer, per `00-api-conventions.md` §0.1.

---

### src/schemas/library.schema.ts (new)

| Schema | Shape |
|---|---|
| uploadMaterialSchema | `z.object({ body: z.object({ cohortId: z.string().uuid(), title: z.string().min(1), fileType: z.enum(['PDF','NOTE','BOOK']) }) })` |

### src/services/library.service.ts (new)

#### uploadMaterial

| Field | Detail |
|---|---|
| Signature | `uploadMaterial(tutorId: string, cohortId: string, title: string, fileType: string, file: Buffer): Promise<LibraryMaterialDTO>` |
| Throws | `ApiError(400, "Unsupported file type")` — beyond the schema's enum, this also validates the actual file's MIME type matches the declared `fileType`. `ApiError(403, "Not authorized to upload to this cohort")` — caller isn't the cohort's assigned tutor. |
| Side effects | Stores via `storage.client.ts`. |

Test file: `tests/services/library.service.test.ts`

#### listCohortMaterials

| Field | Detail |
|---|---|
| Signature | `listCohortMaterials(callerId: string, cohortId: string): Promise<LibraryMaterialDTO[]>` |
| Throws | `ApiError(403, "Not authorized to view this cohort's materials")` — caller not a member/tutor of the cohort. |

Test file: `tests/services/library.service.test.ts`

#### adminManageLibrary

| Field | Detail |
|---|---|
| Signature | `adminManageLibrary(materialId: string, adminId: string, input: { title?, remove? }): Promise<{ id, title }>` |
| Purpose | Retention/access override — also backs the Admin recording-compliance queue read (`GET /admin/library/recording-compliance`), which is a co-located read in this same controller per Doc 05a rather than a separate exported function (it delegates to `recording.service.ts`'s `flagMissing`/`escalateMissing` state, read-only here). |

Test file: `tests/services/library.service.test.ts`

### src/controllers/library.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| upload | `libraryService.uploadMaterial(req.user.id, req.body.cohortId, req.body.title, req.body.fileType, req.file)` | 201 |
| listForCohort | `libraryService.listCohortMaterials(req.user.id, req.params.cohortId)` | 200 |
| adminOverride | `libraryService.adminManageLibrary(req.params.id, req.user.id, req.body)` | 200 |
| adminRecordingCompliance | read via `recording.service.ts` state (`MISSING`/`ESCALATED` sessions), co-located here per Doc 05a | 200 |

### src/routes/library.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | /library/materials | `authMiddleware, validate(uploadMaterialSchema)` | upload |
| GET | /library/cohorts/:cohortId/materials | `authMiddleware` | listForCohort |
| PATCH | /admin/library/materials/:id | `authMiddleware, requireRole('ADMIN')` | adminOverride |
| GET | /admin/library/recording-compliance | `authMiddleware, requireRole('ADMIN')` | adminRecordingCompliance |

Mounted at both `/library` and `/admin/library` — same one-router-two-mount-points pattern as `policy.routes.ts`/`subject.routes.ts`.

---

### src/schemas/reschedule.schema.ts (new)

| Schema | Shape |
|---|---|
| requestRescheduleSchema | `z.object({ body: z.object({ sessionId: z.string().uuid(), requestedNewStart: z.string().datetime() }) })` |

### src/services/reschedule.service.ts (new)

#### requestReschedule

| Field | Detail |
|---|---|
| Signature | `requestReschedule(callerId: string, sessionId: string, requestedNewStart: string): Promise<RescheduleResultDTO>` |
| Purpose | Classifies server-side as `FREE_RESCHEDULE` (≥12h notice) or `SAME_DAY_MISS` (<12h notice) — `noticeHours = (session.scheduledStart - now) in hours`. |
| Throws | `ApiError(400, "Requested time is outside the tutor's availability")` — `requestedNewStart` doesn't fall within an existing `AvailabilitySlot`. |
| Side effects | (`FREE_RESCHEDULE`) Creates a `RescheduleRequest`, moves the session, increments the caller's monthly free-reschedule counter (via `enforceMonthlyCap`). (`SAME_DAY_MISS`) Immediately writes a `SessionMiss` row via `sessionMiss.service.ts`, attributed to whichever party requested it — evaluated under the same tutor-caused/student-caused rules as any other miss. |
| Edge cases | The 12-hour boundary is computed at request time using `session.scheduledStart`, not `requestedNewStart` — a request submitted at exactly 12.0 hours' notice should be defined as inclusive or exclusive at implementation time (Docs 01–04 don't pin this down to the second; document whichever is chosen as a code comment). |

Test file: `tests/services/reschedule.service.test.ts` — includes the 12h-boundary and monthly-cap cases.

#### enforceMonthlyCap

| Field | Detail |
|---|---|
| Signature | `enforceMonthlyCap(callerId: string): Promise<{ freeReschedulesUsedThisMonth: number }>` |
| Throws | `ApiError(409, "Free reschedule limit reached for this month — further changes require Admin review")` — caller has already used 2 free reschedules this month and this request would qualify as free. |
| Edge cases | The cap resets on a rolling calendar-month boundary, not a rolling 30-day window (distinct from `SessionMiss`'s 30-day rolling escalation check below) — worth flagging as a deliberate difference, not an inconsistency, since Doc 02 specifies "per month" for this cap and "rolling 30-day" for tutor-miss escalation separately. |

Test file: `tests/services/reschedule.service.test.ts`

### src/controllers/reschedule.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| requestReschedule | `rescheduleService.requestReschedule(req.user.id, req.body.sessionId, req.body.requestedNewStart)` | 200 |

### src/routes/reschedule.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | / | `authMiddleware, validate(requestRescheduleSchema)` | requestReschedule |

Mounted at `/reschedule`. Ownership check (caller must be a member/tutor of the session's cohort) enforced in the service layer.

---

### src/services/sessionMiss.service.ts (new)

#### recordTutorCausedMiss

| Field | Detail |
|---|---|
| Signature | `recordTutorCausedMiss(sessionId: string, missType: MissType): Promise<TutorCausedMissDTO>` |
| Purpose | Queues a mandatory free make-up session within 7 days; no refund/credit for the missed session itself; tutor is paid a flat 50% share for the make-up (FR-MK-009). |
| Side effects | Creates `SessionMiss(causedBy: TUTOR)`, generates a linked make-up `ScheduledSession` (via `session.service.ts`), sets `makeupDeadline` +7 days, flags `tutorEarningRateForMakeup: REDUCED_MAKEUP` for `earning.service.ts` (`payments-earnings`) to apply once the make-up is delivered. |

#### recordStudentCausedMiss

| Field | Detail |
|---|---|
| Signature | `recordStudentCausedMiss(sessionId: string, missType: MissType): Promise<StudentCausedMissDTO>` |
| Side effects | Creates `SessionMiss(causedBy: STUDENT)` — no make-up, no refund; the session counts as delivered against the month's billed sessions; tutor is paid their normal full share. |

#### checkTutorEscalation

| Field | Detail |
|---|---|
| Signature | `checkTutorEscalation(tutorId: string): Promise<boolean>` |
| Purpose | Computed at query time — `true` when the tutor has 2+ `causedBy: TUTOR` misses within a rolling 30-day window; never a stored counter (Doc 04 SessionMiss notes), to avoid drift between the counter and the underlying rows. |

Throws (both record functions): `ApiError(409, "A miss has already been recorded for this session")` — a `SessionMiss` row already exists for the `sessionId` (enforced via a unique constraint on `sessionId` at the DB level, checked here to return the friendly message rather than a raw constraint error).

Test file: `tests/services/sessionMiss.service.test.ts` — includes the reduced-rate-flag and escalation-threshold cases.

### src/controllers/sessionMiss.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| reportMiss | `sessionMissService.recordTutorCausedMiss` or `recordStudentCausedMiss`, branched on `req.body.causedBy` | 201 |
| listMisses | direct paginated read + `checkTutorEscalation` for the `escalationFlag` | 200 |

### src/routes/sessionMiss.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware` (Tutor sees own record only; Admin sees any, via `req.query.tutorId`) | listMisses |
| POST | / | `authMiddleware, requireRole('ADMIN')` | reportMiss |

Mounted at `/session-miss`. `POST` is Admin-only as the system-of-record entry point; the same effect also results automatically from a `SAME_DAY_MISS` reschedule classification (`POST /reschedule`), which calls `sessionMiss.service.ts` internally rather than through this route.

---

### src/schemas/weeklyAssessment.schema.ts (new)

| Schema | Shape |
|---|---|
| submitAssessmentSchema | `z.object({ body: z.object({ cohortMembershipId: z.string().uuid(), weekStartDate: z.string().date(), scoreSummary: z.string().optional(), tutorFeedback: z.string().min(1) }) })` |

### src/services/weeklyAssessment.service.ts (new)

#### submitAssessment

| Field | Detail |
|---|---|
| Signature | `submitAssessment(tutorId: string, input): Promise<WeeklyAssessmentDTO>` |
| Throws | `ApiError(409, "An assessment for this week has already been submitted")` — a row already exists for this `cohortMembershipId` + `weekStartDate`. |
| Side effects | Verifies caller is the submitting tutor for this membership before inserting. |

#### getAssessmentsForStudent

| Field | Detail |
|---|---|
| Signature | `getAssessmentsForStudent(callerId: string, cohortMembershipId: string): Promise<WeeklyAssessmentDTO[]>` |
| Throws | `ApiError(403, "Not authorized to view these assessments")` — caller is not party to this membership. |
| Edge cases | No assessment submitted yet for the current week returns the list without that week's entry — not an error (UC-62 alternate flow). |

Test file: `tests/services/weeklyAssessment.service.test.ts`

### src/controllers/weeklyAssessment.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| submit | `weeklyAssessmentService.submitAssessment(req.user.id, req.body)` | 201 |
| listForMembership | `weeklyAssessmentService.getAssessmentsForStudent(req.user.id, req.params.id)` | 200 |

### src/routes/weeklyAssessment.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | / | `authMiddleware, validate(submitAssessmentSchema)` | submit |
| GET | /cohort-membership/:id | `authMiddleware` | listForMembership |

Mounted at `/assessments`.

---

### src/jobs/classReminder.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan for `ScheduledSession(status: SCHEDULED)` rows 1 hour from `scheduledStart`. |
| Effect | Sends a reminder to tutor + student via `notification.service.ts` (`shared-config`). |
| Idempotency | A `reminderSentAt` marker (or equivalent) prevents double-sending on overlapping scan windows. |

### src/jobs/recordingMissingCheck.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan for `ScheduledSession(status: COMPLETED)` rows with no linked `Recording`. |
| Effect | Calls `recording.service.ts → flagMissing` at 2h past `scheduledEnd`, `escalateMissing` at 24h — surfaced read-only via `GET /sessions/:sessionId` and `GET /admin/library/recording-compliance` (§0.4). |

---

**Next:** proceed to → [8-5. Backend: Messaging]
