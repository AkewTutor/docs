## Project: AKEWTutor — Backend Test Documentation: Class Delivery, Recording & Library
**Links back to:** [05a. Backend Folder & File Structure §4], [8-4. Function-Level Spec: Class Delivery, Recording & Library]
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.4 (recording-missing/payment-pause-reschedule are job/event-driven) and §0.5 (Cloudflare R2, Jitsi shapes).

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** ScheduledSession, RescheduleRequest, SessionMiss, RecordingConsent, Recording, LibraryMaterial, WeeklyAssessment. **Depends on:** `matching-cohorts` (hard).

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

**Tier note — Integration (HTTP contract):** this tier tests routing/middleware/controller wiring with the service layer mocked. It does not test persistence — see the Integration (persistence) tier (same file, below, or in a sibling `9-N-module-persistence.md`) for that.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| session.service.ts | FR-CD-001–003, FR-CD-006, FR-AC-008 (`assertSessionAccessAllowed` — gap closed), Section 7 v3.2 cadence/billing | — |
| recordingConsent.service.ts | FR-SC-008–009 | — |
| storage.client.ts | FR-SP-035, FR-CD-005 | NFR-007, NFR-012 |
| recording.service.ts | FR-SP-035–037, FR-CD-004–008, Section 5.8 (90-day retention, 720p) | NFR-007, NFR-009, NFR-012 |
| library.service.ts | FR-CD-009, FR-AD-014 | — |
| reschedule.service.ts | FR-MK-004, FR-MK-006–008 | — |
| sessionMiss.service.ts | FR-MK-001–003, FR-MK-009 | — |
| weeklyAssessment.service.ts | FR-SP-038, FR-TU-017 | — |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/session.schema.ts | tests/schemas/session.schema.test.ts | Unit | ☐ |
| src/services/session.service.ts | tests/services/session.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/session.controller.ts | tests/controllers/session.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/session.routes.ts | tests/routes/session.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/schemas/recordingConsent.schema.ts | tests/schemas/recordingConsent.schema.test.ts | Unit | ☐ |
| src/services/recordingConsent.service.ts | tests/services/recordingConsent.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/recordingConsent.controller.ts | tests/controllers/recordingConsent.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/recordingConsent.routes.ts | tests/routes/recordingConsent.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/utils/providers/storage.client.ts | tests/utils/providers/storage.client.test.ts | Unit (mocked R2 SDK) | ☐ |
| src/services/recording.service.ts | tests/services/recording.service.test.ts | Unit (mocked Prisma, mocked `recordingConsent.service`, mocked `storage.client`) | ☐ |
| src/controllers/recording.controller.ts | tests/controllers/recording.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/recording.routes.ts | tests/routes/recording.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/schemas/library.schema.ts | tests/schemas/library.schema.test.ts | Unit | ☐ |
| src/services/library.service.ts | tests/services/library.service.test.ts | Unit (mocked Prisma, mocked `storage.client`) | ☐ |
| src/controllers/library.controller.ts | tests/controllers/library.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/library.routes.ts | tests/routes/library.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/schemas/reschedule.schema.ts | tests/schemas/reschedule.schema.test.ts | Unit | ☐ |
| src/services/reschedule.service.ts | tests/services/reschedule.service.test.ts | Unit (mocked Prisma, mocked `sessionMiss.service`) | ☐ |
| src/controllers/reschedule.controller.ts | tests/controllers/reschedule.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/reschedule.routes.ts | tests/routes/reschedule.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/services/sessionMiss.service.ts | tests/services/sessionMiss.service.test.ts | Unit (mocked Prisma, mocked `session.service`, `notification.service`) | ☐ |
| src/controllers/sessionMiss.controller.ts | tests/controllers/sessionMiss.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/sessionMiss.routes.ts | tests/routes/sessionMiss.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/schemas/weeklyAssessment.schema.ts | tests/schemas/weeklyAssessment.schema.test.ts | Unit | ☐ |
| src/services/weeklyAssessment.service.ts | tests/services/weeklyAssessment.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/weeklyAssessment.controller.ts | tests/controllers/weeklyAssessment.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/weeklyAssessment.routes.ts | tests/routes/weeklyAssessment.routes.test.ts | Integration (HTTP contract, supertest) | ☐ |
| src/jobs/classReminder.job.ts | — | Underlying logic covered via `notification.service.test.ts` (shared-config) call assertions; interval wrapper excluded | — |
| src/jobs/recordingMissingCheck.job.ts | — | Underlying logic covered via `recording.service.test.ts`'s `flagMissing`/`escalateMissing` cases; interval wrapper excluded | — |

---

### 9.2 Test Case Detail — session.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| provideJitsiLinkSchema requires a valid URL | — | parse `{ body: { jitsiLinkUrl: "not-a-url" } }` | fails |
| provideJitsiLinkSchema accepts a valid Jitsi URL | — | parse `{ body: { jitsiLinkUrl: "https://meet.jit.si/abc123" } }` | passes |

---

### 9.3 Test Case Detail — session.service.test.ts

FRs: FR-CD-001–003, Section 7 v3.2 cadence/billing. **OWASP: A01:2021 – Broken Access Control.**

#### generateSessionsForCohort

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sets sessionsPerWeek from matched recurring slots | mock a cohort with 2 distinct matched recurring `AvailabilitySlot`s, `sessionsPerWeek` not yet set | call `generateSessionsForCohort(cohortId)` | `Cohort.sessionsPerWeek` set to `2` |
| Generates sessionsPerWeek × 4 sessions for the cycle | same setup | call `generateSessionsForCohort(cohortId)` | exactly 8 `ScheduledSession(status: SCHEDULED)` rows created — the authoritative worked example from Doc 02 §7 v3.2 |
| Does not double-generate on a re-run for the same cycle | mock existing sessions already present in the target window | call `generateSessionsForCohort(cohortId)` a second time | no additional `ScheduledSession` rows created — guarded by checking for existing sessions first |
| Does not overwrite an already-set sessionsPerWeek | mock `Cohort.sessionsPerWeek` already `2`, but the tutor's current `AvailabilitySlot`s would now compute to `3` | call `generateSessionsForCohort(cohortId)` again (e.g. a retried call) | `sessionsPerWeek` remains `2` — cadence is frozen at first confirmation per Doc 04, a later availability edit must never retroactively change it |
| A tutor's later availability edits never affect an ACTIVE cohort's cadence | mock a cohort already `ACTIVE` with `sessionsPerWeek: 2`; mock the tutor removing/adding slots afterward | re-derive/re-check `sessionsPerWeek` for this cohort | unchanged at `2` — only a fresh `MatchRequest`/new `Cohort` picks up new cadence, per Doc 02 §7's explicit freeze rule |
| **[Phase 4 — Review §6.1] Weekly recurring sessions keep the same local wall-clock time across a DST transition** | mock a recurring `AvailabilitySlot` expressed as a UTC-offset window belonging to a user whose client reports a DST-observing zone (e.g. a diaspora parent in `America/New_York`); generate the cycle's 4 weekly `ScheduledSession` occurrences spanning that zone's DST transition date | call `generateSessionsForCohort(cohortId)` | each of the 4 sessions' *local wall-clock* time in that zone is identical week to week (e.g. always "4:00 PM Eastern") even though the corresponding UTC instant shifts by an hour across the transition — sessions are generated from the stored recurrence rule's local time semantics, not by adding a fixed UTC duration per week, which would silently produce a session an hour off from what the tutor/student actually agreed to. `Africa/Addis_Ababa` itself never observes DST, so this case only matters for a traveling or diaspora user's client-side interpretation of the same underlying UTC instants — the stored instants themselves are unaffected either way |

#### provideJitsiLink

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Tutor provides link ≥30 minutes before start | mock `scheduledStart` 45 minutes away | call `provideJitsiLink(tutorId, sessionId, url)` | resolves `{ id, jitsiLinkUrl, jitsiLinkSentAt, providedLateNotice: false }` |
| Tutor provides link at exactly the 30-minute boundary | mock `scheduledStart` exactly 30 minutes away | call `provideJitsiLink(...)` | resolves `providedLateNotice: false` — flagged for implementer to confirm inclusive/exclusive boundary choice is applied consistently; this doc assumes ≥30 min is on-time per FR-CD-003's "at least 30 minutes" wording |
| Tutor provides link under 30 minutes before start | mock `scheduledStart` 10 minutes away | call `provideJitsiLink(...)` | resolves successfully (submission is **not** blocked) with `providedLateNotice: true` |
| Submission still succeeds even if extremely late (mid-class or after) | mock `scheduledStart` in the past | call `provideJitsiLink(...)` | still resolves (not rejected outright) with `providedLateNotice: true` — the student still needs the link; this flag only feeds `sessionMiss.service.ts` if the class ends up disrupted |
| Non-assigned tutor rejected (IDOR) | mock caller is not this session's assigned tutor | call `provideJitsiLink(otherTutorId, sessionId, url)` | throws `ApiError(403, "Not authorized to provide a link for this session")` |

#### assertSessionAccessAllowed (**gap closed** — see `04-database-and-data-model.md §4.2`, `8-4-class-delivery-library.md`)

FRs: FR-AC-008, FR-CD-006. **OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Student caller with ACTIVE account and real membership | mock `assertAccountStatusAllowsAccess` (accounts-guardianship) to resolve silently; mock caller has an `ACTIVE`/historical `CohortMembership` on the session's `Cohort` | call `assertSessionAccessAllowed(callerId, 'STUDENT', sessionId)` | resolves the `ScheduledSession` row |
| **Student/Parent blocked while on the guardian-hold** | mock `assertAccountStatusAllowsAccess` to throw `ApiError(403, ...)` | call `assertSessionAccessAllowed(callerId, 'STUDENT', sessionId)` | the same `ApiError(403, ...)` propagates unmodified, and is thrown before the membership/ownership check below is even reached |
| **Parent caller is gated on the target student's status, not their own** | mock `assertAccountStatusAllowsAccess` to throw for the linked student | call `assertSessionAccessAllowed(parentId, 'PARENT', sessionId)` | throws the same `ApiError(403, ...)` — a parent has no independent `accountStatus` of their own; the gate always resolves against the `StudentProfile` |
| **Tutor caller is never gated by the guardian-hold check** | mock `assertAccountStatusAllowsAccess` to throw if it were called (it must not be) | call `assertSessionAccessAllowed(tutorId, 'TUTOR', sessionId)` where the tutor is genuinely assigned to the cohort | resolves the session normally — `assertAccountStatusAllowsAccess` is asserted **not called** for a `TUTOR`/`ADMIN` caller, confirming the tutor/Admin exemption documented in Doc 8-4 |
| **Admin caller is never gated by the guardian-hold check** | same pattern as above, `callerRole: 'ADMIN'` | call `assertSessionAccessAllowed(adminId, 'ADMIN', sessionId)` | resolves the session; `assertAccountStatusAllowsAccess` asserted not called |
| Caller with no relation to the cohort at all (IDOR, independent of hold status) | mock a real, existing session/cohort the caller was never a member or tutor of; `assertAccountStatusAllowsAccess` resolves silently (account is `ACTIVE`) | call `assertSessionAccessAllowed(callerId, 'STUDENT', otherSessionId)` | throws `ApiError(403, "Not authorized to view this session")` — this is the ordinary membership check, distinct from the hold check above, and must still fire even when the caller's own account is in good standing |
| Non-existent sessionId | mock `ScheduledSession.findUnique` → `null` | call `assertSessionAccessAllowed(callerId, 'STUDENT', randomUUID())` | throws `ApiError(404, ...)` |

#### markCompleted

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Marks a SCHEDULED session completed | mock session `status: SCHEDULED` | call `markCompleted(tutorId, sessionId)` | resolves `{ id, status: 'COMPLETED' }` |
| Already-completed session rejected | mock `status: COMPLETED` | call `markCompleted(tutorId, sessionId)` | throws `ApiError(409, "This session's status cannot be changed")` |
| Missed session rejected | mock `status: MISSED` | call `markCompleted(...)` | throws the same `ApiError(409, ...)` |
| Cancelled session rejected | mock `status: CANCELLED` | call `markCompleted(...)` | throws the same `ApiError(409, ...)` |

#### listMySessions / getSession (co-located reads)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No sessions yet | mock empty result for a student still matching | call the list read | resolves `{ sessions: [] }`, not an error |
| getSession — non-member rejected (IDOR) | mock caller has no `CohortMembership`/tutor assignment on this session's cohort | call `getSession(callerId, sessionId)` | throws `ApiError(403, "Not authorized to view this session")` |
| getSession reports PAYMENT_PAUSE_RESCHEDULED as a valid read-only state | mock a session with `status: 'PAYMENT_PAUSE_RESCHEDULED'` | call `getSession(callerId, sessionId)` | resolves that status verbatim — this endpoint never sets it, only reports it (§0.4) |
| listMySessions scoped strictly to the caller | mock sessions belonging to other cohorts entirely | call `listMySessions(callerId)` | assert the query filter is scoped to cohorts the caller actually belongs to — a missing scope here would leak other students'/tutors' schedules |

---

### 9.4 Test Case Detail — session.controller.test.ts / session.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 4 routes require auth | no Authorization header | request each | `401` |
| provideLink validates URL shape | mock controller layer | request with `{ jitsiLinkUrl: "not-a-url" }` | rejected by `validate(provideJitsiLinkSchema)` |
| getSession propagates 403 unchanged | mock service to throw | call controller | passed through |

---

### 9.5 Test Case Detail — recordingConsent.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| acknowledgeConsentSchema requires both tutorId and studentId as uuids | — | parse `{ body: { tutorId: "x" } }` | fails |

---

### 9.6 Test Case Detail — recordingConsent.service.test.ts

FRs: FR-SC-008–009.

#### getConsentStatus

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Neither side acknowledged | mock no `RecordingConsent` row | call `getConsentStatus(tutorId, studentId)` | resolves `{ tutorId, studentId, tutorAcknowledgedAt: null, studentOrParentAcknowledgedAt: null, consentComplete: false }` |
| One side acknowledged | mock a row with only `tutorAcknowledgedAt` set | call `getConsentStatus(...)` | resolves `consentComplete: false` |
| Both sides acknowledged | mock both timestamps set | call `getConsentStatus(...)` | resolves `consentComplete: true` |

#### acknowledgeAsStudentOrParent / acknowledgeAsTutor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| First acknowledgment creates the row | mock no existing row | call `acknowledgeAsStudentOrParent(callerId, tutorId, studentId)` | upserts `RecordingConsent` with `studentOrParentAcknowledgedAt` set, `consentComplete: false` (tutor side still pending) |
| Second acknowledgment completes the pairing | mock a row with the student side already set | call `acknowledgeAsTutor(callerId, tutorId, studentId)` | `tutorAcknowledgedAt` set; `consentComplete` flips to `true` |
| Acknowledgment is one-time per pairing, not per session | mock a pairing already fully consented | call `acknowledgeAsStudentOrParent(...)` again for the same pairing | resolves idempotently (no duplicate row, no error) — a re-acknowledgment does not need to be blocked, but must not create a second `RecordingConsent` row for the same pairing |
| Consent for one pairing does not affect another pairing in the same cohort | mock a 1-to-3 cohort, one student's pairing consented, another's not | call `getConsentStatus` for each pairing independently | each pairing's `consentComplete` reflects only its own two timestamps, never bleeding across pairings |

---

### 9.7 Test Case Detail — recordingConsent.controller.test.ts / recordingConsent.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Both routes require auth | no Authorization header | request `GET /recording-consent/status`, `POST /recording-consent/acknowledge` | `401` |
| acknowledge branches correctly by role | mock both service functions | call controller as a Student, then as a Tutor, for the same pairing | `acknowledgeAsStudentOrParent` called for the Student caller, `acknowledgeAsTutor` called for the Tutor caller — never the wrong one |
| acknowledge as a Parent also routes to the student-side function | mock service | call controller as a Parent (on behalf of a linked student) | `acknowledgeAsStudentOrParent` called — Parent is treated as the student-side party, per FR-SC-008's "students/parents" wording |

---

### 9.8 Test Case Detail — storage.client.test.ts

**OWASP: A01:2021 – Broken Access Control (signed URL scope/expiry), A02:2021 – Cryptographic Failures (credential handling), A05:2021 – Security Misconfiguration (bucket/object exposure).**

#### upload

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful upload | mock the R2 SDK `PutObject` call to resolve | call `upload(key, fileBuffer, 'video/mp4')` | resolves `{ storageKey }` |
| Upload failure propagates a normalized error | mock the SDK call to reject | call `upload(...)` | rejects — callers (`recording.service.ts`) are expected to handle this as a genuine failure, not silently swallowed like a notification dispatch |
| Storage key is not derived from unsanitized user input | inspect the key passed to the SDK call, given a title/filename containing path-traversal-like characters (e.g. `../../etc/passwd`) | call `upload(maliciously-crafted-key, file, contentType)` | assert the actual key sent to R2 is sanitized/namespaced (e.g. prefixed with a generated UUID, not the raw user-supplied filename) — a raw pass-through here would be a path-traversal-adjacent object-storage risk (OWASP A05) |

#### getSignedUrl

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns a URL with the requested expiry | mock the SDK's presigned-URL generator | call `getSignedUrl(storageKey, 900)` | resolves a URL string; the SDK call is asserted to have been invoked with `expiresIn: 900` |
| Signed URL is short-lived per the platform's 900s convention | — | call `getSignedUrl(storageKey, 900)` from `recording.service.ts`'s actual call site | assert `recording.service.ts` always passes `900` (15 minutes), never an unbounded/very long expiry, for recording access (Section 5.8's "temporary/signed URLs, never permanent public links" requirement) |
| Credentials never appear in the returned URL logged anywhere | spy on any logger | call `getSignedUrl(...)` | no log statement contains the raw `CLOUDFLARE_R2_SECRET_KEY` |

---

### 9.9 Test Case Detail — recording.service.test.ts

FRs: FR-SP-035–037, FR-CD-004–008. **OWASP: A01:2021 – Broken Access Control (this is the file's central risk — cross-student/cross-cohort recording access), A04:2021 – Insecure Design (consent-gating is a design-level safety control).**

#### uploadRecording

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful upload for a fully-consented 1-to-1 pairing | mock `getConsentStatus` → `consentComplete: true` for the single pairing | call `uploadRecording(tutorId, sessionId, file)` | resolves `RecordingDTO`; `expiresAt` = upload time + 90 days; `keepPermanently: false` |
| Blocked — 1-to-1 pairing consent incomplete | mock `getConsentStatus` → `consentComplete: false` | call `uploadRecording(tutorId, sessionId, file)` | throws `ApiError(409, "Recording consent must be acknowledged by both parties for every active student before this session can be recorded")` |
| 1-to-3/1-to-5 — blocked if ANY active member's pairing is incomplete | mock 3 active members, 2 with `consentComplete: true`, 1 with `false` | call `uploadRecording(tutorId, sessionId, file)` | throws the same `ApiError(409, ...)` — a single incomplete pairing blocks the whole class's upload, not just that student's access |
| 1-to-3/1-to-5 — succeeds once every active member's pairing is complete | mock all active members `consentComplete: true` | call `uploadRecording(...)` | resolves successfully |
| Mid-cycle new joiner re-blocks future uploads until their consent completes | mock a cohort that previously uploaded successfully, then a new member joins with `consentComplete: false` | call `uploadRecording` for the *next* session after the join | throws the consent-gate `ApiError(409, ...)` again — re-checked at each upload, not cached from a prior successful upload |
| Non-assigned tutor rejected | mock caller is not the session's assigned tutor | call `uploadRecording(otherTutorId, sessionId, file)` | throws `ApiError(403, "Not authorized to upload a recording for this session")` |
| Stored at 720p (encoding parameter asserted, not pixel-counted in a unit test) | spy on the encoding/storage call | call `uploadRecording(...)` | assert the call to `storage.client.upload` (or an encoding step preceding it) specifies the 720p compressed target, per Section 5.8 |

#### getSignedUrl

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Cohort member retrieves their own recording | mock caller was a member of the recording's cohort | call `getSignedUrl(callerId, recordingId)` | resolves `{ signedUrl, expiresIn: 900 }` |
| Cross-student access denied — never a member of this cohort (IDOR/BOLA) | mock caller has zero `CohortMembership` rows for this recording's cohort, ever | call `getSignedUrl(otherStudentId, recordingId)` | throws `ApiError(403, "Not authorized to access this recording")` — this is the canonical case Doc 8-4 explicitly calls out: a guessed-but-valid recording UUID must not be openable by an unrelated student |
| A student in the same cohort as a *different* recording cannot cross into this one via id-guessing | mock caller is a genuine member of Cohort A; `recordingId` belongs to Cohort B | call `getSignedUrl(callerId, cohortBRecordingId)` | throws the same `ApiError(403, ...)` — being a legitimate student *somewhere* on the platform is not sufficient; membership must be on *this* recording's specific cohort |
| Past retention, not kept permanently | mock `expiresAt` in the past, `keepPermanently: false` | call `getSignedUrl(callerId, recordingId)` | throws `ApiError(404, "Recording no longer available")` — treated as genuinely not-found, per §0.3 |
| Past retention but kept permanently | mock `expiresAt` in the past, `keepPermanently: true` | call `getSignedUrl(callerId, recordingId)` | resolves successfully — retention does not apply once flagged |
| Soft-deleted recording | mock `deletedAt` set | call `getSignedUrl(callerId, recordingId)` | throws the same `ApiError(404, ...)` |
| Tutor retrieves a recording for their own past student | mock caller is the assigned tutor for this recording's cohort | call `getSignedUrl(tutorId, recordingId)` | resolves successfully — FR-CD-007's tutor access, subject to the same cohort-membership-style check |

#### keepPermanently

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Owner marks a recording to keep permanently | mock caller was a cohort member | call `keepPermanently(callerId, recordingId)` | resolves `{ id, keepPermanently: true }` |
| Non-member attempts to mark another student's recording (IDOR) | mock caller has no membership on this cohort | call `keepPermanently(otherStudentId, recordingId)` | throws `ApiError(403, "Not authorized to modify this recording")` |

#### flagMissing / escalateMissing

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Flags MISSING at 2h with no recording | mock a `COMPLETED` session, no linked `Recording`, 2+ hours past `scheduledEnd` | call `flagMissing(sessionId)` | `ScheduledSession.recordingStatus: MISSING` |
| Escalates at 24h if still missing | mock the same session, now 24+ hours past `scheduledEnd`, still `MISSING` | call `escalateMissing(sessionId)` | `recordingStatus: ESCALATED` |
| Does not flag a session that already has a recording | mock a session with a linked `Recording` | call `flagMissing(sessionId)` | no state change — the job's own query should already exclude this, but the function itself is tested defensively too |
| Missing recording alone does not trigger a refund or make-up | spy on any refund/make-up-triggering call | call `flagMissing(sessionId)` then `escalateMissing(sessionId)` | assert neither function calls into `sessionMiss.service.ts` or `refund.service.ts` — per Doc 09.1's explicit "does not, by itself, trigger a refund or make-up... unless the student separately reports" rule |

---

### 9.10 Test Case Detail — recording.controller.test.ts / recording.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 4 routes require auth | no Authorization header | request each | `401` |
| getMyRecordings excludes expired-and-not-kept rows | mock a mix of active, expired-kept, and expired-not-kept recordings | call the list read | the expired-not-kept recording is absent from the returned list — expected expiry, not an error |
| getSignedUrl propagates 403 and 404 distinctly | mock service to throw each in turn | call controller | both propagate unchanged — a client must be able to distinguish "not yours" from "gone," even though both are non-2xx |

---

### 9.11 Test Case Detail — library.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| uploadMaterialSchema requires a valid fileType enum | — | parse `{ body: { cohortId: uuid, title: "Notes", fileType: "VIDEO" } }` | fails — only `PDF`/`NOTE`/`BOOK` allowed |
| uploadMaterialSchema requires a non-empty title | — | parse `{ body: { cohortId: uuid, title: "", fileType: "PDF" } }` | fails |

---

### 9.12 Test Case Detail — library.service.test.ts

FRs: FR-CD-009, FR-AD-014.

#### uploadMaterial

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Assigned tutor uploads successfully | mock caller is the cohort's assigned tutor, file MIME matches declared type | call `uploadMaterial(tutorId, cohortId, "Chapter 3 Notes", "PDF", file)` | resolves `LibraryMaterialDTO` |
| Non-assigned tutor rejected | mock caller is not this cohort's tutor | call `uploadMaterial(otherTutorId, cohortId, ...)` | throws `ApiError(403, "Not authorized to upload to this cohort")` |
| Declared fileType doesn't match actual file MIME type | mock a `.exe`/mismatched MIME file declared as `PDF` | call `uploadMaterial(tutorId, cohortId, title, "PDF", file)` | throws `ApiError(400, "Unsupported file type")` — this is a business-rule check beyond the schema enum, guarding against a client lying about content type (OWASP A08 — content-type/integrity spoofing relevant to file-upload endpoints) |

#### listCohortMaterials

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Member retrieves cohort materials | mock caller is a current member/tutor | call `listCohortMaterials(callerId, cohortId)` | resolves the material list |
| Non-member rejected (IDOR) | mock caller has no membership/assignment on this cohort | call `listCohortMaterials(otherId, cohortId)` | throws `ApiError(403, "Not authorized to view this cohort's materials")` |

#### adminManageLibrary

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Admin edits a material's title | mock material found | call `adminManageLibrary(materialId, adminId, { title: "Updated title" })` | resolves `{ id, title }` |
| Admin removes a material | mock material found | call `adminManageLibrary(materialId, adminId, { remove: true })` | material removed/flagged removed |
| Recording-compliance read reflects recording.service.ts's flagged state, not its own | spy on any independent recompute of MISSING/ESCALATED | call the compliance read path | assert it reads `ScheduledSession.recordingStatus` as already set by `recording.service.ts`, performing no independent staleness calculation — same read-only boundary pattern as `adminMatching.listPendingApprovals` in 9-3 |

---

### 9.13 Test Case Detail — library.controller.test.ts / library.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| upload and listForCohort require auth | no Authorization header | request `POST /library/materials`, `GET /library/cohorts/:id/materials` | both `401` |
| adminOverride and adminRecordingCompliance require Admin | no Authorization header, then a Tutor token | request `PATCH /admin/library/materials/:id`, `GET /admin/library/recording-compliance` | `401` then `403` for each |

---

### 9.14 Test Case Detail — reschedule.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| requestRescheduleSchema requires a valid uuid sessionId and ISO datetime | — | parse `{ body: { sessionId: "x", requestedNewStart: "not-a-date" } }` | fails on both |

---

### 9.15 Test Case Detail — reschedule.service.test.ts

FRs: FR-MK-004, FR-MK-006–008.

#### requestReschedule

| Case | Setup | Action | Expected result |
|---|---|---|---|
| ≥12h notice classified as FREE_RESCHEDULE | mock `scheduledStart` 24 hours away, requestedNewStart within tutor availability | call `requestReschedule(callerId, sessionId, requestedNewStart)` | creates a `RescheduleRequest`, moves the session, increments the caller's monthly counter; no `SessionMiss` created |
| Exactly at the 12-hour boundary | mock `scheduledStart` exactly 12.0 hours away | call `requestReschedule(...)` | classified per the implementer's documented inclusive/exclusive choice (flagged as an open item in Doc 8-4) — this test asserts whichever choice was coded is applied **consistently**, and that the choice is documented in a code comment; it does not itself mandate which side of the boundary is "free" |
| <12h notice classified as SAME_DAY_MISS | mock `scheduledStart` 6 hours away | call `requestReschedule(...)` | delegates to `sessionMiss.service.ts` to write a `SessionMiss` row attributed to the requesting party; **no** `RescheduleRequest`/free-reschedule path taken |
| Requested time outside tutor availability | mock `requestedNewStart` falling outside every `AvailabilitySlot` for the tutor | call `requestReschedule(...)` | throws `ApiError(400, "Requested time is outside the tutor's availability")` |
| Monthly cap enforced — 3rd free reschedule in the same month rejected | mock caller has already used 2 free reschedules this calendar month, this request is ≥12h notice | call `requestReschedule(...)` | throws `ApiError(409, "Free reschedule limit reached for this month — further changes require Admin review")` via `enforceMonthlyCap` |
| Monthly cap resets at the calendar-month boundary, not a rolling 30 days | mock 2 free reschedules used in the prior calendar month, 0 so far in the current one | call `requestReschedule(...)` (≥12h notice) in the new month | succeeds — confirms the cap is calendar-month-scoped, distinct from `SessionMiss`'s rolling-30-day escalation window |
| A reschedule never consumes a make-up session or has a billing impact | mock a valid FREE_RESCHEDULE | call `requestReschedule(...)` | assert no `SessionMiss`/make-up/earning-rate flag is touched — confirms FR-MK-008's "carries no billing impact and does not consume a make-up session" |
| **[Phase 4 — Review §6.1] Notice-hours computation is unaffected by a DST transition between now and the session** | mock `scheduledStart` such that the wall-clock difference for a DST-observing client (e.g. `America/New_York`) would appear as either 11 or 13 hours depending on whether a naive `(scheduledStart - now) / 3600000` calculation is done against local time vs. UTC instants, straddling a DST transition date | call `requestReschedule(...)` for a case where the true elapsed time is exactly 12 real (UTC) hours | classified `FREE_RESCHEDULE` — the elapsed-time calculation is performed on the two `Date`/UTC instants directly (millisecond difference), never by re-deriving each side's local calendar/clock representation first, which is exactly the kind of arithmetic a DST-observing zone can silently corrupt by an hour |

#### enforceMonthlyCap

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Under the cap | mock 1 used this month | call `enforceMonthlyCap(callerId)` | resolves `{ freeReschedulesUsedThisMonth: 1 }`, does not throw |
| At the cap | mock 2 used this month, a 3rd free-eligible request incoming | call `enforceMonthlyCap(callerId)` | throws `ApiError(409, ...)` |

---

### 9.16 Test Case Detail — reschedule.controller.test.ts / reschedule.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Route requires auth | no Authorization header | request `POST /reschedule` | `401` |
| Validates body shape | mock controller layer | request with a missing `requestedNewStart` | rejected by `validate(requestRescheduleSchema)` |
| Ownership check happens at the service layer, controller is pass-through | mock service | call controller with a caller who is not on the session's cohort | error propagates from the service (see 9.15's implied ownership check — flagged: Doc 8-4 doesn't explicitly table this ApiError message; the test should confirm *some* 403 occurs, and flag the exact message to the implementer if not otherwise specified) |

---

### 9.17 Test Case Detail — sessionMiss.service.test.ts

FRs: FR-MK-001–003, FR-MK-009.

#### recordTutorCausedMiss

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Records the miss and queues a free make-up | mock a valid session, no existing `SessionMiss` for it | call `recordTutorCausedMiss(sessionId, 'NO_SHOW')` | creates `SessionMiss(causedBy: TUTOR)`; generates a linked make-up `ScheduledSession` via `session.service.ts`; `makeupDeadline` = now + 7 days |
| Make-up is flagged for the reduced tutor rate | mock a valid tutor-caused miss | call `recordTutorCausedMiss(...)` | the created make-up session (or the `SessionMiss` row) carries `tutorEarningRateForMakeup: REDUCED_MAKEUP` for `earning.service.ts` to apply later — this test asserts the *flag* is set, not the actual payout math (that lives in `9-7-payments-earnings.md`) |
| No refund/credit issued for the missed session itself | spy on any refund-triggering call | call `recordTutorCausedMiss(...)` | assert no `refund.service` call is made — FR-MK-001's explicit "no refund/credit issued for that session" |
| Duplicate miss for the same session rejected | mock a `SessionMiss` row already exists for this `sessionId` | call `recordTutorCausedMiss(sessionId, ...)` again | throws `ApiError(409, "A miss has already been recorded for this session")` |
| Escalation check fires after the 2nd tutor-caused miss in 30 days | mock this is the tutor's 2nd `causedBy: TUTOR` miss within a rolling 30-day window | call `recordTutorCausedMiss(...)`, then `checkTutorEscalation(tutorId)` | `checkTutorEscalation` resolves `true` |

#### recordStudentCausedMiss

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Records the miss with no make-up | mock a valid session | call `recordStudentCausedMiss(sessionId, 'NO_SHOW')` | creates `SessionMiss(causedBy: STUDENT)`; no make-up `ScheduledSession` created; session still counts as delivered against the month's billed sessions |
| Tutor paid full share for a student-caused miss | spy on any earning-rate flag | call `recordStudentCausedMiss(...)` | no `REDUCED_MAKEUP` flag set anywhere — the tutor's normal full-rate earning applies since they were available and ready, per FR-MK-009's explicit exclusion |
| Duplicate miss rejected | mock existing `SessionMiss` for the session | call `recordStudentCausedMiss(sessionId, ...)` again | throws the identical `ApiError(409, "A miss has already been recorded for this session")` |

#### checkTutorEscalation

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Below threshold | mock 1 tutor-caused miss in the last 30 days | call `checkTutorEscalation(tutorId)` | resolves `false` |
| At threshold | mock exactly 2 tutor-caused misses within the last 30 days | call `checkTutorEscalation(tutorId)` | resolves `true` |
| Misses outside the 30-day window don't count | mock 2 tutor-caused misses, one 31+ days ago, one recent | call `checkTutorEscalation(tutorId)` | resolves `false` — only 1 falls inside the rolling window |
| Computed at query time, never a stored/cached counter | spy on the query | call `checkTutorEscalation(tutorId)` twice, with a new miss inserted between calls (mocked) | the second call reflects the new miss immediately — confirms no stale counter field is being read (Doc 04's explicit design choice to avoid drift) |
| Student-caused misses never count toward tutor escalation | mock 5 `causedBy: STUDENT` misses for sessions this tutor taught | call `checkTutorEscalation(tutorId)` | resolves `false` — only `causedBy: TUTOR` rows count |

---

### 9.18 Test Case Detail — sessionMiss.controller.test.ts / sessionMiss.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| POST / requires Admin | no Authorization header, then a Tutor token | request `POST /session-miss` | `401` then `403` |
| GET / allows a Tutor to see only their own record | mock service; caller is a Tutor with no `tutorId` query override | call controller | the underlying read is scoped to `req.user.id`, ignoring any client-supplied `tutorId` for a non-Admin caller |
| GET / allows Admin to view any tutor via query param | mock service; caller is Admin with `req.query.tutorId` set | call controller | the read is scoped to the queried `tutorId` |
| reportMiss branches correctly by causedBy | mock both service functions | call controller with `causedBy: 'TUTOR'`, then `causedBy: 'STUDENT'` | the corresponding function is called each time, never the wrong one |

---

### 9.19 Test Case Detail — weeklyAssessment.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requires tutorFeedback | — | parse `{ body: { cohortMembershipId: uuid, weekStartDate: "2026-06-01" } }` | fails — `tutorFeedback` required, `min(1)` |
| scoreSummary is optional | — | parse a body with `tutorFeedback` set but no `scoreSummary` | passes |

---

### 9.20 Test Case Detail — weeklyAssessment.service.test.ts

FRs: FR-SP-038, FR-TU-017.

#### submitAssessment

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Tutor submits a new weekly assessment | mock caller is the membership's tutor, no existing row for this week | call `submitAssessment(tutorId, input)` | resolves `WeeklyAssessmentDTO` |
| Duplicate for the same week rejected | mock a row already exists for `(cohortMembershipId, weekStartDate)` | call `submitAssessment(tutorId, input)` again | throws `ApiError(409, "An assessment for this week has already been submitted")` |
| Non-assigned tutor rejected | mock caller is not the tutor for this membership | call `submitAssessment(otherTutorId, input)` | rejected — Doc 8-4 says the caller is verified before inserting; assert an `ApiError` (403-class) is thrown, flagging to the implementer that Doc 8-4 doesn't table the exact message for this case |

#### getAssessmentsForStudent

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Party to the membership retrieves assessments | mock caller is the student/parent/tutor on this membership | call `getAssessmentsForStudent(callerId, cohortMembershipId)` | resolves the assessment list |
| Non-party rejected (IDOR) | mock caller has no relation to this membership | call `getAssessmentsForStudent(otherId, cohortMembershipId)` | throws `ApiError(403, "Not authorized to view these assessments")` |
| No assessment yet for the current week | mock a list missing the current week's entry | call `getAssessmentsForStudent(...)` | resolves the list without that week — not an error, per UC-62 |

---

### 9.21 Test Case Detail — weeklyAssessment.controller.test.ts / weeklyAssessment.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Both routes require auth | no Authorization header | request `POST /assessments`, `GET /assessments/cohort-membership/:id` | both `401` |
| submit validates body | mock controller layer | request missing `tutorFeedback` | rejected by `validate(submitAssessmentSchema)` |

---

### 9.22 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `uploadRecording`'s group-format consent gate is tested with the **any-one-incomplete-blocks-all** case explicitly (2 of 3 complete, upload still blocked) — not just the all-complete and all-incomplete extremes, which would miss a bug where the check only looks at the *first* member.
- [ ] `getSignedUrl`'s cross-student-access-denied case is tested against a recording belonging to a *different, real* cohort the caller has no relation to — not just an entirely nonexistent id (which would only exercise the 404 path, not the 403/IDOR path).
- [ ] **[Phase 4]** Both DST-transition cases (`generateSessionsForCohort`, `requestReschedule`) are run with the test environment/mocked clock set to a genuinely DST-observing zone — see the identical note in `9-3-matching-cohorts.md`'s Coverage Honesty Check; the same false-pass risk applies here.
- [ ] The 90-day-retention-vs-kept-permanently distinction is tested as two separate branches on the same expired timestamp, not inferred from one case.
- [ ] `checkTutorEscalation`'s rolling-30-day window is tested with a miss just outside the window (e.g. day 31) to confirm the boundary excludes it, not just with misses clearly inside or clearly far outside.
- [ ] The reschedule monthly cap and the tutor-escalation 30-day window are tested as genuinely different time-scoping rules (calendar-month vs. rolling-30-day) — a shared/copy-pasted date-math helper between the two would be a real bug this doc's explicit callout is meant to catch.
- [ ] `flagMissing`/`escalateMissing`'s "does not itself trigger refund/make-up" is tested by spying on those downstream calls and asserting they were never made — not merely checking the return value.
- [ ] `library.service.uploadMaterial`'s MIME-vs-declared-type check is tested with an actual mismatch case, not only the schema-level enum validation (which is a separate, weaker check already covered in 9.11).

---

### 9.23 Out of Scope for Automated Testing (and why)

- **Real video encoding to 720p** — `recording.service.ts` is tested for whether it *calls* the storage/encoding step with the right parameters; actual transcoding correctness and output quality is a manual/infrastructure verification, not a unit test concern.
- **Real Cloudflare R2 behavior** (bucket policy, actual signed-URL redemption, network latency) — `storage.client.ts` is unit-tested against a mocked SDK only.
- **Jitsi link validity/liveness** — the backend stores and delivers a URL string; it never calls a Jitsi API (§0.5), so there is nothing to test beyond URL-shape validation, already covered in 9.2.
- **`classReminder.job.ts` / `recordingMissingCheck.job.ts` interval scheduling** — covered indirectly via the services they call; the cron registration itself is excluded per the standing convention.
- **Exact inclusive/exclusive behavior at the 12-hour reschedule boundary and the ≥30-minute Jitsi-link boundary** — flagged in both 9.9 and 9.15 as decisions Docs 01–04 leave to implementation-time judgment; this doc requires the chosen behavior be applied *consistently* and documented in code, rather than guessing a specific side of the boundary to hard-assert here.

---

**Next:** proceed to → [9-5. Backend Test Documentation: Messaging]
