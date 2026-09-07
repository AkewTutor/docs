## Project: AKEWTutor — Backend Test Documentation: Matching & Cohorts
**Links back to:** [05a. Backend Folder & File Structure §3], [8-3. Function-Level Spec: Matching & Cohorts]
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.4 (system-driven state — no trigger endpoint for job-driven transitions).

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** MatchRequest, TutorExclusion, Cohort, CohortMembership, FormatSwitchRequest. **Depends on:** `accounts-guardianship` (hard).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered |
|---|---|
| matching.service.ts | FR-MA-001–002, FR-MA-007, FR-MA-012, FR-MA-016–018, FR-SP-025 (uniqueStudentsTaught read), FR-TU-008 (automatic secondary-subject fallback), Section 8 v3.2 match-percentage formula |
| cohort.service.ts | FR-MA-006, FR-MA-009, FR-MA-011, FR-MA-016, Section 7 v3.0 partial-group-formation, Section 8 tutor-exit continuity + M3 group-splitting worked example, FR-SP-030 (profile-visibility split) |
| adminMatching.service.ts | FR-MA-003–004, FR-MA-008, FR-MA-010, FR-MA-013–015, FR-MA-017, FR-AD-005–008, Section 11.3 stale-booking-approval rules |
| formatSwitch.service.ts | FR-SP-045–049 |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/matching.schema.ts | tests/schemas/matching.schema.test.ts | Unit | ☐ |
| src/services/matching.service.ts | tests/services/matching.service.test.ts | Unit (mocked Prisma, mocked `cohort.service`) | ☐ |
| src/controllers/matching.controller.ts | tests/controllers/matching.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/matching.routes.ts | tests/routes/matching.routes.test.ts | Integration (supertest) | ☐ |
| src/services/cohort.service.ts | tests/services/cohort.service.test.ts | Unit (mocked Prisma, mocked `adminMatching.service` for handoff calls) | ☐ |
| src/controllers/cohort.controller.ts | tests/controllers/cohort.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/cohort.routes.ts | tests/routes/cohort.routes.test.ts | Integration (supertest) | ☐ |
| src/services/adminMatching.service.ts | tests/services/adminMatching.service.test.ts | Unit (mocked Prisma, mocked `cohort.service`, `notification.service`) | ☐ |
| src/controllers/adminMatching.controller.ts | tests/controllers/adminMatching.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminMatching.routes.ts | tests/routes/adminMatching.routes.test.ts | Integration (supertest) | ☐ |
| src/schemas/formatSwitch.schema.ts | tests/schemas/formatSwitch.schema.test.ts | Unit | ☐ |
| src/services/formatSwitch.service.ts | tests/services/formatSwitch.service.test.ts | Unit (mocked `cohort.service`, `matching.service`, `refund.service`) | ☐ |
| src/controllers/formatSwitch.controller.ts | tests/controllers/formatSwitch.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/formatSwitch.routes.ts | tests/routes/formatSwitch.routes.test.ts | Integration (supertest) | ☐ |
| src/jobs/groupFormationWindow.job.ts | — | Underlying close-and-form logic covered via `cohort.service.test.ts`'s partial-formation case; interval wrapper excluded | — |
| src/jobs/zeroMatchEscalation.job.ts | — | Underlying escalation logic covered via `matching.service.test.ts`; interval wrapper excluded | — |
| src/jobs/staleApproval.job.ts | — | Underlying flagging logic covered via `adminMatching.service.test.ts`; interval wrapper excluded | — |

---

### 9.2 Test Case Detail — matching.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| searchTutorsQuerySchema requires subjectId and grade | — | parse `{ query: {} }` | fails on both missing fields |
| searchTutorsQuerySchema rejects grade outside 1–12 | — | parse `{ query: { subjectId: uuid, grade: '13' } }` | fails (coerced then range-checked) |
| searchTutorsQuerySchema accepts an optional studentId (parent-on-behalf-of case) | — | parse `{ query: { subjectId: uuid, grade: '9', studentId: uuid } }` | passes |
| selectTutorSchema requires tutorId | — | parse `{ body: {} }` | fails |
| noExactMatchSchema accepts an empty body (student self case) | — | parse `{ body: {} }` | passes — `studentId` optional, defaults to caller |

---

### 9.3 Test Case Detail — matching.service.test.ts

FRs: FR-MA-001–002, FR-MA-007, FR-MA-012, FR-MA-016–018, FR-SP-025, FR-TU-008, Section 8 v3.2 match-percentage formula. **OWASP: A01:2021 – Broken Access Control (parent-on-behalf-of studentId scoping mirrors 9-2's pattern), A04:2021 – Insecure Design (matching correctness has direct fairness/business-integrity implications).**

#### searchOneToOneTutors

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns matches for a 1-to-1 student | mock caller `formatPreference: ONE_TO_ONE`; mock qualifying tutors | call `searchOneToOneTutors(callerId, 'STUDENT', undefined, filters)` | resolves `tutors: [...]` |
| Rejects a non-1-to-1 caller | mock caller `formatPreference: ONE_TO_THREE` | call `searchOneToOneTutors(...)` | throws `ApiError(400, "Search is only available for the 1-to-1 format — see /matching/group-format for 1-to-3/1-to-5")` |
| Budget is a hard filter — over-budget tutors excluded entirely | mock 3 candidate tutors, one priced above the student's `budgetPreference` | call `searchOneToOneTutors(..., { budget: "300" })` | the over-budget tutor is **absent** from results, not merely sorted last |
| Language is a hard filter across formats | mock a tutor whose teaching language ≠ student's `preferredLanguage` | call `searchOneToOneTutors(..., { language: "Amharic" })` | the mismatched-language tutor is absent |
| Only VERIFIED tutors appear | mock a `PENDING` tutor who otherwise matches every filter | call `searchOneToOneTutors(...)` | the unverified tutor is absent — confirms `adminTutorVerification`'s approval gate is actually enforced in the query, not just assumed |
| Zero results is a 200, not a 404 | mock zero qualifying tutors | call `searchOneToOneTutors(...)` | resolves `{ tutors: [] }`, does not throw — this is what surfaces "No Exact Match" client-side, per §0.3 |
| Grade is always a hard match | mock a tutor whose ranked subject applies (any grade per FR-TU-006) but query grade doesn't overlap the student's actual need — **clarify**: since a tutor's ranked subject spans all of Grades 1–12, this case instead verifies the *student's stated* `grade` param is passed through to the query filter, not silently ignored | call `searchOneToOneTutors(..., { grade: 4 })` twice with different grade values | assert the Prisma query filter reflects the passed grade each time — grade always participates in the query even though it never disqualifies a tutor by ranked-subject grade-range (there is none) |
| Primary-subject search with results never falls back to secondary-subject tutors (FR-TU-008) | mock ≥1 tutor with `rank = 1` for the student's primary subject/grade, plus a separate tutor who only qualifies via `rank = 2` (secondary) for that same subject/grade | call `searchOneToOneTutors(callerId, 'STUDENT', undefined, filters)` | resolves only the `rank = 1` tutor(s) — the secondary-ranked tutor is **absent**; a primary-subject match must never trigger the secondary-subject fallback |
| Primary-subject search with zero results automatically falls back to secondary-subject tutors (FR-TU-008) | mock zero tutors with `rank = 1` for the student's subject/grade, and ≥1 tutor with `rank = 2` for the same subject/grade | call `searchOneToOneTutors(callerId, 'STUDENT', undefined, filters)` | resolves `{ tutors: [...] }` including the `rank = 2` tutor(s) — the system automatically re-queries against secondary-subject rankings in the same call/response, with no separate Admin- or client-triggered action required |

#### recommendTutorsWithMatchPercent

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Hard filters applied before any scoring | mock a tutor who fails the language hard filter | call `recommendTutorsWithMatchPercent(...)` | that tutor never appears in `recommendations` at all — **never scores `0%`**, per the v3.2 formula note that a hard-filter failure means the tutor never enters scoring |
| Primary-subject match scores 40% of the subject-rank factor at its max | mock a qualifying tutor with `TutorSubjectRanking.rank = 1` for the student's subject, a teaching-style match, and 100% schedule overlap | call `recommendTutorsWithMatchPercent(...)` | `matchPercentage = round(0.40×100 + 0.35×100 + 0.25×100) = 100` |
| Secondary-subject match scores 70 on the subject-rank factor | mock `rank = 2` for the student's subject, teaching-style match, 100% overlap | call `recommendTutorsWithMatchPercent(...)` | `matchPercentage = round(0.40×70 + 0.35×100 + 0.25×100) = round(28+35+25) = 88` |
| Teaching-style mismatch scores 40 on that factor | mock `rank = 1`, tutor style ≠ student's stated preference, 100% overlap | call `recommendTutorsWithMatchPercent(...)` | `matchPercentage = round(0.40×100 + 0.35×40 + 0.25×100) = round(40+14+25) = 79` |
| No stated teaching-style preference is neutral (scores 100, never penalized) | mock `rank = 1`, student has no `teachingStylePreference` set, 100% overlap | call `recommendTutorsWithMatchPercent(...)` | `matchPercentage = round(0.40×100 + 0.35×100 + 0.25×100) = 100` — confirms "no preference" is scored identically to "matched," per the v3.2 formula's explicit neutral-scoring rule, not defaulted to the 40-point mismatch score |
| Schedule overlap is proportional, capped at 100 | mock student has 4 preferred slots, tutor overlaps 2 of them | call `recommendTutorsWithMatchPercent(...)` | scheduleOverlapScore = `(2/4)×100 = 50`; full formula = `round(0.40×100 + 0.35×100 + 0.25×50) = round(40+35+12.5) = round(87.5) = 88` (round-half-up) |
| Round-half-up applied at exactly .5 | construct inputs whose weighted sum is exactly `94.5` (e.g. subjectRank=100, style=100, overlap=78 → 0.40×100+0.35×100+0.25×78 = 40+35+19.5 = 94.5) | call `recommendTutorsWithMatchPercent(...)` | `matchPercentage = 95`, not `94` — the exact worked example from Doc 02 §8's v3.2 callout |
| MATCH_WEIGHTS are not Admin-configurable | inspect the module for any Prisma/config read of the weight constants | call `recommendTutorsWithMatchPercent(...)` | assert no database/config lookup occurs for the 40/35/25 weights — they are hardcoded named constants per Doc 02 §8, and a test that mocks them as DB-driven would be testing the wrong design |
| Creates a SEARCHING MatchRequest on first call | mock no existing `MatchRequest` | call `recommendTutorsWithMatchPercent(...)` | a new `MatchRequest(status: SEARCHING)` is created |
| Reuses an existing MatchRequest on a subsequent call | mock an existing `SEARCHING` `MatchRequest` for the caller | call `recommendTutorsWithMatchPercent(...)` again | no duplicate `MatchRequest` row is created — the existing one is reused/updated |
| Zero recommendations reports zeroMatchSince from the MatchRequest row | mock zero qualifying tutors; mock `MatchRequest.zeroMatchSince` already set by the job | call `recommendTutorsWithMatchPercent(...)` | resolves `{ recommendations: [], zeroMatchSince: <timestamp> }` — this function reads, but never itself computes, the 48-hour threshold |

#### selectTutor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful selection | mock tutor not excluded, no existing active match | call `selectTutor(callerId, 'STUDENT', studentId, tutorId)` | resolves `{ cohortId, status: 'PENDING_ADMIN_APPROVAL', tutorId }`; a `Cohort(format: ONE_TO_ONE)` + single `CohortMembership` created via `cohort.service.ts` |
| Excluded tutor rejected | mock a `TutorExclusion` row for `(studentId, tutorId)` | call `selectTutor(...)` | throws `ApiError(400, "This tutor is not available — please choose from your current recommendations")` — this is the mechanism that prevents re-selecting a tutor who previously rejected/was rejected for this student (Section 8 Cross-Path Rules) |
| Already has a pending or active match | mock an existing non-terminal `MatchRequest`/`Cohort` for the caller | call `selectTutor(...)` | throws `ApiError(409, "You already have a pending or active match")` |
| Delegates cohort creation, does not duplicate it | spy on `cohort.service` import | call `selectTutor(...)` | assert `cohort.service.ts`'s shared cohort-creation path was invoked rather than a second, independent `Cohort.create` call living in this function |

#### triggerNoExactMatch

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Manual trigger produces the standard Path B state | — | call `triggerNoExactMatch(callerId, 'STUDENT', studentId)` | resolves `{ matchRequestId, status: 'PENDING_ADMIN_ASSIGNMENT' }` |
| Resulting state is identical to the automatic 48h escalation's state | mock a `MatchRequest` that was auto-escalated by `zeroMatchEscalation.job.ts` (status already `PENDING_ADMIN_ASSIGNMENT`/`ZERO_MATCH_PENDING` per the read-path note) alongside a manually-triggered one | compare both `MatchRequest` shapes | identical downstream shape — Admin's queue handling must not need to distinguish which path produced the row, per Doc 8-3's explicit "all three paths converge" note |

#### requestGroupFormat

| Case | Setup | Action | Expected result |
|---|---|---|---|
| 1-to-3/1-to-5 caller succeeds | mock caller `formatPreference: ONE_TO_THREE` | call `requestGroupFormat(callerId, 'STUDENT', studentId, subjectId)` | resolves `{ matchRequestId, status: 'SEARCHING' }` |
| 1-to-1 caller rejected | mock caller `formatPreference: ONE_TO_ONE` | call `requestGroupFormat(...)` | throws `ApiError(400, "Use /matching/select-tutor or /matching/no-exact-match for the 1-to-1 format")` |
| Response never includes tutorId, matchPercentage, or any profile field | mock a successful call | call `requestGroupFormat(...)` | assert the resolved object has **only** `matchRequestId`/`status` keys — a deliberately smaller DTO shape, not a full object with hidden fields the client is merely told not to render (API spec §3.2's no-match-information rule for group formats — this must be enforced server-side) |
| Actual grouping does not happen synchronously in this call | spy on `cohort.service.formOrJoinCohort` | call `requestGroupFormat(...)` | assert `formOrJoinCohort` is **not** called directly from within this function — Doc 8-3 states grouping is invoked by a separate matching/scheduling process, not synchronously here; a test asserting it IS called here would be testing an incorrect coupling |

#### Status read path (getMyRequestStatus / getTutorDetail)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No active request | mock no `MatchRequest` found | call the requests/me read | throws `ApiError(404, "No match request in progress")` |
| ZERO_MATCH_PENDING covers both manual and automatic triggers indistinguishably | mock a row with `status: ZERO_MATCH_PENDING` regardless of how it got there | call the read | resolves the same shape either way — client never sees a "how did this happen" field |
| Tutor not found | mock no `TutorProfile` found | call the tutor-detail read | throws `ApiError(404, "Tutor not found")` |
| Unverified tutor treated as not found | mock a `TutorProfile` with `verificationStatus: PENDING` | call the tutor-detail read for that id | throws the same `ApiError(404, "Tutor not found")` — an unverified tutor's profile must not be viewable via a guessed/leaked id even though the row technically exists |
| uniqueStudentsTaught is a distinct count, not a session tally | mock a tutor who has taught the same student across 5 separate completed `CohortMembership`/session rows, plus 2 other distinct students | call the tutor-detail read | `uniqueStudentsTaught === 3`, not `7` — direct regression test for FR-SP-025's amended wording |

---

### 9.4 Test Case Detail — matching.controller.test.ts / matching.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 7 routes require auth | no Authorization header | request each of the 7 endpoints | all `401` |
| searchTutors validates required query params | mock controller layer | request `GET /matching/tutors/search` with no `subjectId`/`grade` | rejected by `validate(searchTutorsQuerySchema)` |
| selectTutor passes req.user through, studentId optional | mock service | call controller as a Student with no `studentId` in body | `selectTutor` called with `(req.user.id, 'STUDENT', undefined, req.body.tutorId)` |
| selectTutor as a Parent passes the target studentId | mock service | call controller as a Parent with `req.body.studentId` set | `selectTutor` called with `(parentId, 'PARENT', studentId, tutorId)` — the ownership check itself lives at the service layer (9.3), this test only confirms pass-through |
| requestGroupFormat response never leaks extra fields even if the mocked service accidentally returns more | mock service to resolve `{ matchRequestId, status, tutorId: 'leaked' }` (a deliberately over-permissive mock) | call controller | **flag, not a pass/fail on the controller alone:** this scenario exists to make explicit that the no-match-information guarantee must be enforced in the *service* DTO shape (9.3), since the controller is a thin pass-through and will faithfully forward whatever the service returns — a controller-only test cannot substitute for the service-level shape assertion |

---

### 9.5 Test Case Detail — cohort.service.test.ts

FRs: FR-MA-006, FR-MA-009, FR-MA-011, FR-MA-016, Section 7 partial-formation, Section 8 tutor-exit + M3 group-splitting, FR-SP-030. **OWASP: A01:2021 – Broken Access Control (profile-visibility split enforcement).**

#### formOrJoinCohort

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Joins an existing compatible FORMING cohort | mock a `Cohort(status: FORMING)` matching subject/grade/format/schedule with room remaining | call `formOrJoinCohort(matchRequestId)` | adds a `CohortMembership`; resolves `{ cohortId, status: 'FORMING' }` (or `'FULL'` if this membership fills the target size) |
| Creates a new cohort when none compatible exists | mock no compatible `FORMING` cohort | call `formOrJoinCohort(matchRequestId)` | creates a new `Cohort`, sets `groupFormationWindowExpiresAt` = now + 48h (Admin-configurable default) |
| Never sets or touches any pricing field | spy on the `Cohort`/`CohortMembership` create/update calls | call `formOrJoinCohort(matchRequestId)` for a 1-to-5 request that ends up at partial size | assert no `pricePerStudentPerHour`/`amount`-shaped field appears anywhere in this function's writes — pricing is a `payments-earnings` concern entirely, confirming the boundary Doc 8-3 explicitly draws |
| Cohort reaching target size resolves FULL | mock a cohort at `groupSize - 1`, this call is the final join | call `formOrJoinCohort(matchRequestId)` | resolves `status: 'FULL'` |

#### approveCohort / rejectCohort

| Case | Setup | Action | Expected result |
|---|---|---|---|
| approveCohort moves state forward | mock a pending-approval cohort | call `approveCohort(cohortId, adminId)` | resolves `CohortResultDTO` reflecting the approved state |
| rejectCohort on a Path A (1-to-1) cohort writes a TutorExclusion and a fresh MatchRequest | mock a `ONE_TO_ONE` cohort pending approval | call `rejectCohort(cohortId, adminId, "reason")` | a `TutorExclusion(studentId, tutorId)` row is created; a new `MatchRequest(status: SEARCHING)` is spawned for the student — this is what makes the excluded tutor disappear from the student's next `recommendTutorsWithMatchPercent` call (cross-referenced with 9.3) |
| rejectCohort on a Path C (group) cohort re-queues without exclusion | mock a `ONE_TO_THREE`/`ONE_TO_FIVE` cohort pending approval | call `rejectCohort(cohortId, adminId, "reason")` | **no** `TutorExclusion` row is written; the affected students' requests simply re-enter the auto-match queue (`status: SEARCHING` again) — per Section 8's v3.0 Admin-rejection-handling distinction between Path A and Path C |
| Internal rejection reason never surfaces to the student | mock a rejection with `internalReason: "Tutor's availability conflicted with a higher-priority booking"` | call `rejectCohort(...)`; inspect the student-facing notification payload | the notification contains only a generic "assignment could not be confirmed" message, never the internal reason string |

#### tutorExitContinuity

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Ends the outgoing cohort and memberships | mock an active `ONE_TO_FIVE` cohort, 5 active `CohortMembership` rows, `reason: 'DROPOUT'` | call `tutorExitContinuity(cohortId, 'DROPOUT')` | `Cohort.status: ENDED, endedReason: TUTOR_DROPOUT`; all 5 `CohortMembership` rows set to `status: ENDED, endReason: DROPPED_BY_ADMIN` |
| Suspension reason recorded distinctly from dropout | same setup, `reason: 'SUSPENDED'` | call `tutorExitContinuity(cohortId, 'SUSPENDED')` | `Cohort.endedReason: TUTOR_SUSPENDED`, distinct value from the dropout case |
| Spawns exactly one fresh MatchRequest per affected membership | mock 5 active memberships | call `tutorExitContinuity(cohortId, reason)` | exactly 5 new `MatchRequest` rows created, each `path: PATH_C, status: PENDING_ADMIN_ASSIGNMENT` (not `SEARCHING`) — the specific gap Doc 8-3 flags as previously missing in prior drafts |
| Single eligible replacement tutor keeps the group together | mock a tutor with capacity/eligibility for all 5 affected students | call `tutorExitContinuity(cohortId, reason)` | hands off to `adminMatching.service.manuallyAssignTutor` with all 5 new `MatchRequest` ids as one call — the group is not split |
| No single tutor can take the full group — flagged for manual assembly | mock no single eligible tutor for all 5 | call `tutorExitContinuity(cohortId, reason)` | the 5 new `MatchRequest` rows are flagged/left for `manuallyAssembleGroup`, per the M3 worked example — this function itself does not attempt the 3-and-2 (or any) split; it hands off the *possibility* of a split to Admin |
| Group-splitting produces independent cohorts with independent cadence (verified at the adminMatching layer, cross-referenced) | — | see `adminMatching.service.test.ts`'s `manuallyAssembleGroup` cases below for the actual split-execution assertions; this file only verifies the fresh-`MatchRequest`-per-membership spawn and single-vs-multi-tutor branch decision | — |
| Refund for the transition gap is calculated against the OLD cohort's totalSessionsBilled | spy on any refund-triggering call this function makes (directly or via a downstream feature) | call `tutorExitContinuity(cohortId, reason)` | if this function itself triggers the refund flow, assert it references the OLD cohort's `totalSessionsBilled`, not a new cohort's — per the M3 worked example's explicit "old cohort's totalSessionsBilled, new cohort starts fresh" rule; if this triggering instead lives in `payments-earnings`, this test asserts only that the ended `Cohort`'s id/`totalSessionsBilled` snapshot is what gets handed off, and the actual refund-calculation test lives in `9-7-payments-earnings.md` |

#### endCohort

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sets the timestamp messaging keys off of | — | call `endCohort(cohortId, "reason string")` | `Cohort.status: ENDED, endedAt` set to the call time, `endedReason` stored — this exact field is what `archiveMessageThreads.job.ts` (messaging feature) reads for its 90-day window, so a wrong/missing `endedAt` here would silently break that feature's archival timing |

#### getMyCohort / getCohortMembers (co-located reads)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No cohort yet | mock no `Cohort`/`CohortMembership` for caller | call the getMyCohort read | resolves `{ cohorts: [] }`, not an error, per §0.3 |
| Non-member requests cohort details (IDOR) | mock caller has no `CohortMembership` in the target cohort and is not its assigned tutor | call `getCohortMembers(cohortId, callerId)` | throws `ApiError(403, "Not authorized to view this cohort")` |
| Student in a 1-to-3/1-to-5 cohort sees only name+photo per member | mock a `ONE_TO_FIVE` cohort, caller is a current student member | call `getCohortMembers(cohortId, studentCallerId)` | the tutor entry in the response contains only `name`/`photo` — **no** `education`, `totalStudentsCount`, or `matchPercentage` field present at all (not present-but-null) |
| Student in a 1-to-1 cohort sees the full tutor profile | mock a `ONE_TO_ONE` cohort | call `getCohortMembers(cohortId, studentCallerId)` | the tutor entry includes `education`, `totalStudentsCount`, etc. — the full-profile shape is format-conditional |
| Tutor sees only studentId/firstName/grade per member, for any format | mock the caller is the assigned tutor of a `ONE_TO_THREE` cohort | call `getCohortMembers(cohortId, tutorCallerId)` | each member entry contains only `studentId, firstName, grade` — never full academic profile, budget, or contact info |

---

### 9.6 Test Case Detail — cohort.controller.test.ts / cohort.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Both routes require auth | no Authorization header | request `GET /cohorts/me`, `GET /cohorts/:id/members` | both `401` |
| getCohortMembers propagates 403 unchanged | mock service to throw | call controller | error passed through |

---

### 9.7 Test Case Detail — adminMatching.service.test.ts

FRs: FR-MA-003–004, FR-MA-008, FR-MA-010, FR-MA-013–015, FR-MA-017, FR-AD-005–008, Section 11.3. **OWASP: A01:2021 – Broken Access Control.**

#### listPendingApprovals

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Overdue items always sort to top regardless of filter | mock a mix of overdue and non-overdue items | call `listPendingApprovals(false, undefined, 1, 20)` (not overdue-only) | overdue items appear first in the returned order |
| overdueOnly filter narrows the set | mock the same mix | call `listPendingApprovals(true, undefined, 1, 20)` | only overdue items returned |
| Non-overdue items explicitly report isOverdue: false | mock a case under 48 hours | call `listPendingApprovals(...)` | that item's `isOverdue === false` in the response — the field is never omitted, per Doc 8-3's explicit note |
| Path filter narrows to Path A/B/C | mock mixed-path items | call `listPendingApprovals(false, 'PATH_C', 1, 20)` | only Path C items returned |
| This function never computes staleness itself | spy on any date-math performed inside this function beyond reading existing fields | call `listPendingApprovals(...)` | assert the function reads `isOverdue`/`adminOverdueNotifiedAt` as stored, performing no independent 48-hour calculation of its own — that logic belongs to `staleApproval.job.ts` only |

#### approveBooking / rejectBooking

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Approve moves cohort to PENDING_PAYMENT | mock a cohort in a pending-approval state | call `approveBooking(cohortId, adminId)` | `Cohort.status: PENDING_PAYMENT` |
| Approve does not itself confirm the schedule | spy on any scheduling/session-generation call | call `approveBooking(cohortId, adminId)` | assert no `class-delivery-library` session-generation call is made from here — schedule confirmation requires payment completion first, per Doc 8-3's explicit boundary |
| Approving a cohort not awaiting approval | mock a cohort already `PENDING_PAYMENT` or `ACTIVE` | call `approveBooking(cohortId, adminId)` | throws `ApiError(409, "This case is not awaiting approval")` |
| Rejecting a cohort not awaiting approval | same pattern | call `rejectBooking(cohortId, adminId, "reason")` | throws the identical `ApiError(409, ...)` |
| rejectBooking delegates the state mutation to cohort.service, not duplicated here | spy on `cohort.service.rejectCohort` | call `rejectBooking(cohortId, adminId, "reason")` | assert `cohort.service.rejectCohort` was called — this service's own role is authorization/orchestration only, per Doc 8-3's explicit split |

#### manuallyAssignTutor / manuallyAssembleGroup

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Assign a single MatchRequest (Path B) | mock an eligible, VERIFIED tutor with matching subject/grade | call `manuallyAssignTutor([matchRequestId], tutorId, adminId)` | resolves a `CohortAssignmentDTO`; `Cohort.status: PENDING_PAYMENT` directly — no separate approve step |
| Ineligible tutor rejected — not verified | mock tutor `verificationStatus: PENDING` | call `manuallyAssignTutor([id], tutorId, adminId)` | throws `ApiError(400, "Selected tutor is not eligible for this assignment")` |
| Ineligible tutor rejected — subject/grade mismatch for one of several requests | mock a tutor eligible for 4 of 5 `matchRequestIds` but not the 5th (e.g. wrong ranked subject) | call `manuallyAssignTutor([5 ids], tutorId, adminId)` | throws the same `ApiError(400, ...)` — group assembly must validate mutual compatibility across *every* request in the array, not just the majority |
| M3 worked example — single tutor takes the full group of 5, no split | mock 5 `MatchRequest` rows (all `path: PATH_C`, from a `tutorExitContinuity` spawn) and one tutor eligible for all 5 with sufficient `AvailabilitySlot` capacity for the group's cadence | call `manuallyAssignTutor([5 ids], tutorId, adminId)` | one new `Cohort` created, inheriting the same `sessionsPerWeek` as the original ended cohort; all 5 students become members of this one new cohort |
| M3 worked example — group split into two new cohorts | mock the same 5 `MatchRequest` rows; Admin makes two separate calls: `manuallyAssembleGroup([3 ids], tutorX, adminId)` then `manuallyAssembleGroup([2 ids], tutorY, adminId)` | call both in sequence | two independent `Cohort` rows result, each with its own `sessionsPerWeek` (derived independently from each new tutor's matched `AvailabilitySlot`s, **not** required to match the original cohort's cadence), its own `MessageThread`, and its own billing cycle |
| Split cohorts' sessionsPerWeek are independently derived, not copied | mock tutorX's matched slots yield 3 sessions/week, tutorY's yield 1 session/week, original cohort had `sessionsPerWeek: 2` | call both `manuallyAssembleGroup` calls | resulting cohorts have `sessionsPerWeek: 3` and `sessionsPerWeek: 1` respectively — neither equals the original `2`, confirming no copy-through occurs |
| Illustrative split ratio is not a hard rule | mock a 5-student split into groups of 4 and 1 (not the illustrative 3-and-2) | call `manuallyAssembleGroup` with a 4/1 split across two calls | both succeed — Doc 02 §8 explicitly states the 3-and-2 example is illustrative, not a required grouping; a test asserting a specific split ratio is required would be over-constraining the spec |
| Double-fail group assembly (both primary and secondary subject searches failed) reaches this function with no student-facing trigger | mock a `Path C` `MatchRequest` in `PENDING_ADMIN_ASSIGNMENT` from a double-fail (FR-MA-017), not from tutor-exit | call `manuallyAssembleGroup([id], tutorId, adminId)` | resolves the same `CohortAssignmentDTO` shape — this function handles both origins (double-fail and tutor-exit-split) identically, since both converge on the same `PENDING_ADMIN_ASSIGNMENT` `MatchRequest` state |

---

### 9.8 Test Case Detail — adminMatching.controller.test.ts / adminMatching.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 4 routes require Admin | no Authorization header, then a Tutor token | request `GET /admin/matching/queue`, `POST /admin/matching/:id/approve`, `POST /admin/matching/:id/reject`, `POST /admin/matching/manual-assign` | `401` then `403` for each |
| approve/reject pass req.user.id as adminId | mock service | call controller | correct `adminId` propagation, never client-suppliable |
| manualAssign dispatches to manuallyAssembleGroup for multi-id requests | mock service; spy on which function is called | call controller with `req.body.matchRequestIds.length > 1` | `manuallyAssembleGroup` invoked, not `manuallyAssignTutor` |
| manualAssign dispatches to manuallyAssignTutor for a single-id request | same | call controller with a single-element array | `manuallyAssignTutor` invoked |

---

### 9.9 Test Case Detail — formatSwitch.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requires a valid toFormat enum value | — | parse `{ body: { toFormat: 'ONE_TO_TWO' } }` | fails |
| Accepts an optional studentId | — | parse `{ body: { toFormat: 'ONE_TO_ONE', studentId: uuid } }` | passes |

---

### 9.10 Test Case Detail — formatSwitch.service.test.ts

FRs: FR-SP-045–049.

#### requestSwitch

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful switch from 1-to-1 to 1-to-3 | mock an active `ONE_TO_ONE` `CohortMembership`, `toFormat: ONE_TO_THREE` | call `requestSwitch(callerId, 'STUDENT', studentId, 'ONE_TO_THREE')` | current membership ended; a new `MatchRequest` created under the new format; `refundId` populated if remaining sessions existed |
| No active assignment to switch from | mock no active `CohortMembership` | call `requestSwitch(...)` | throws `ApiError(409, "No active assignment to switch from")` |
| Already in the requested format | mock an active `ONE_TO_ONE` membership, `toFormat: 'ONE_TO_ONE'` | call `requestSwitch(...)` | throws `ApiError(400, "You are already in this format")` |
| Old assignment cancelled immediately | mock a valid switch | call `requestSwitch(...)` | assert the old `CohortMembership` end call happens synchronously within this function, before/alongside the new `MatchRequest` creation — not deferred |
| Re-enters matching via the correct path for the new format | mock switching to `ONE_TO_ONE` | call `requestSwitch(...)` | the new `MatchRequest` is created via the Path A/B entry point in `matching.service.ts` |
| Re-enters matching via Path C for a group format | mock switching to `ONE_TO_FIVE` | call `requestSwitch(...)` | the new `MatchRequest` is created via the Path C entry point (`requestGroupFormat`'s underlying creation logic) |
| Refund is null when there were zero remaining paid sessions | mock the current billing cycle already fully consumed (0 sessions remaining) | call `requestSwitch(...)` | resolves `refundId: null` |
| Cohort-mates unaffected — group format switch | mock the caller is one of 4 students in a `ONE_TO_FIVE` cohort; the other 3 have separate `CohortMembership` rows | call `requestSwitch(callerId, ...)` | only the caller's own `CohortMembership` is ended; the other 3 members' rows and the cohort itself (if it still has ≥1 member) are untouched — this is the case Doc 8-3 explicitly calls out |
| Outgoing tutor notified with the correct, non-punitive framing | mock a valid switch; spy on the notification dispatched to the outgoing tutor | call `requestSwitch(...)` | the notification indicates a student-initiated format switch, not a rating/performance/complaint-driven removal (FR-SP-049) |
| Refund amount uses the sessions-delivered proration formula (cross-referenced) — **I1 fix** | spy on the call into `refund.service.ts` | call `requestSwitch(...)` | assert `refund.service.createPendingRefund(paymentId, 'FORMAT_SWITCH')` is invoked (which internally calls `calculateProration` with the current cohort's `totalSessionsBilled` and sessions-delivered-so-far) — the actual proration-math assertions live in `9-7-payments-earnings.md`; this test only confirms the correct handoff occurs from this function, and that the resulting `Refund` is left `PENDING` for Admin review rather than auto-approved |

---

### 9.11 Test Case Detail — formatSwitch.controller.test.ts / formatSwitch.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Route requires auth | no Authorization header | request `POST /format-switch` | `401` |
| Validates toFormat enum at the route layer | mock controller layer | request with `{ toFormat: 'BOGUS' }` | rejected by `validate(requestFormatSwitchSchema)` |
| Controller passes req.user through | mock service | call controller | `requestSwitch` called with `(req.user.id, req.user.role, req.body.studentId, req.body.toFormat)` |

---

### 9.12 Coverage Honesty Check (per PR Steward, at review time)

- [ ] Every branch of the match-percentage formula (subject-rank 40%, teaching-style 35% including the "no preference = neutral" case, schedule-overlap 25% with capping, and the round-half-up boundary at exactly `.5`) has its own isolated test with hand-computed expected output — not one end-to-end test that happens to produce a plausible-looking number.
- [ ] The "hard filter fails → never enters scoring, never scores 0%" rule is tested by confirming the tutor is **absent from the array**, not present with a `matchPercentage: 0` — these are different bugs and only the array-membership assertion catches the second one.
- [ ] The Path A vs. Path C rejection re-routing (`TutorExclusion` written vs. not) is tested as two genuinely separate cases with separate setup, not inferred from one generic "rejection re-routes" test.
- [ ] `tutorExitContinuity`'s fresh-`MatchRequest`-per-membership spawn is asserted by counting the actual created rows (5 for a 5-student cohort), not just asserting "some match requests were created."
- [ ] The M3 group-split test explicitly asserts the two resulting cohorts have **independently-derived** `sessionsPerWeek` values that do **not** equal the original cohort's cadence — a test that only checks "two cohorts were created" would miss a regression where cadence was incorrectly copied through.
- [ ] `getCohortMembers`'s profile-visibility split is tested via `not.toHaveProperty` for the omitted fields (`education`, `totalStudentsCount`, `matchPercentage`), not merely checking the fields aren't rendered by a client that was never under test.
- [ ] `requestGroupFormat`'s no-match-information guarantee is tested against the actual resolved object's key set, not against documentation describing what the client is supposed to hide.

---

### 9.13 Out of Scope for Automated Testing (and why)

- **Real concurrent group-formation races** (two students joining the same near-full cohort simultaneously) — unit tests exercise `formOrJoinCohort`'s branching logic against mocked, sequential Prisma calls; genuine concurrency/locking behavior against a live database needs a separate load/concurrency test, not a Vitest unit suite.
- **`groupFormationWindow.job.ts`, `zeroMatchEscalation.job.ts`, `staleApproval.job.ts` interval scheduling** — the *effects* of each job are covered via the relevant service's test file (`cohort.service.test.ts`, `matching.service.test.ts`, `adminMatching.service.test.ts` respectively); the cron/interval registration itself is excluded per the standing convention.
- **Admin-configurable 48-hour group-formation window value** — tests assume the documented 48-hour default; if/when Admin-configuration of this value ships, a dedicated test for the configurable-value path should be added at that time rather than guessed at here.
- **Real notification delivery for rejection/format-switch/tutor-exit notices** — covered at the `dispatchNotification` call-site level (asserting it was called with the right non-disclosing payload); actual SMS/email/push delivery is `shared-config`'s concern (see `9-1-shared-config.md`).

---

**Next:** proceed to → [9-4. Backend Test Documentation: Class Delivery, Recording & Library]
