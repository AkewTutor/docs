## Project: AKEWTutor — Backend Function-Level Spec: Matching & Cohorts
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.4 (system-driven state — group-formation window close, zero-match escalation, stale-approval flags have no trigger endpoint). **API reference:** `03-matching-cohorts-api.md`. **Folder/file reference:** `05a-backend-structure.md` §3.

**Owns:** MatchRequest, TutorExclusion, Cohort, CohortMembership, FormatSwitchRequest. **Depends on:** `accounts-guardianship` (hard) — matching cannot run without a student's academic profile or a tutor's ranked subjects/availability.

**Links back to:** [06-api/03-matching-cohorts-api.md], [05a. Backend Folder & File Structure §3]
**Links forward to:** [9-3. Backend Test Spec: Matching & Cohorts]

---

### src/schemas/matching.schema.ts (new)

| Schema | Shape |
|---|---|
| searchTutorsQuerySchema | `z.object({ query: z.object({ studentId: z.string().uuid().optional(), subjectId: z.string().uuid(), grade: z.coerce.number().int().min(1).max(12), scheduleAvailability: z.object({}).passthrough().optional(), budget: z.string().optional(), language: z.string().optional(), priceMax: z.string().optional() }) })` |
| selectTutorSchema | `z.object({ body: z.object({ studentId: z.string().uuid().optional(), tutorId: z.string().uuid() }) })` |
| noExactMatchSchema | `z.object({ body: z.object({ studentId: z.string().uuid().optional() }) })` |

### src/services/matching.service.ts (new)

#### searchOneToOneTutors

| Field | Detail |
|---|---|
| Signature | `searchOneToOneTutors(callerId, callerRole, studentId, filters): Promise<TutorSearchResultDTO[]>` |
| Purpose | Path A entry point — 1-to-1 tutor search by subject, grade, and hard filters. |
| Throws | `ApiError(400, "Search is only available for the 1-to-1 format — see /matching/group-format for 1-to-3/1-to-5")` — caller's `formatPreference` isn't `ONE_TO_ONE`. |
| Side effects | Read-only query against `TutorProfile` (verified only) joined to `TutorSubjectRanking` (subject match) and `AvailabilitySlot`; `budget` and `language` are hard filters (excluded entirely if unmet), `teachingStylePreference` is a soft factor used only in `recommendTutorsWithMatchPercent`, not here. |
| Edge cases | Zero results → `tutors: []`, `200` — not an error; this is what surfaces the "No Exact Match" option client-side (§0.3). |

Test file: `tests/services/matching.service.test.ts` — includes budget/language hard-filter cases.

#### recommendTutorsWithMatchPercent

| Field | Detail |
|---|---|
| Signature | `recommendTutorsWithMatchPercent(callerId, callerRole, studentId): Promise<{ recommendations: TutorRecommendationDTO[]; matchRequestId: string; zeroMatchSince: string \| null }>` |
| Purpose | 1-to-1 recommendations with a calculated match percentage — the more complete flow behind Path A vs. raw search. |
| Side effects | Computes `matchPercentage` as `round(0.40 × subjectRankScore + 0.35 × teachingStyleScore + 0.25 × scheduleOverlapScore)` — the exact weighted formula and per-factor scoring now defined in Doc 02 §8's v3.2 callout (M7 fix; previously an undocumented implementation detail). Creates or reuses a `MatchRequest(status: SEARCHING)` row for tracking. |
| Edge cases | Empty `recommendations` → `zeroMatchSince` reflects when the zero-match streak began, read from the `MatchRequest` row maintained by `zeroMatchEscalation.job.ts` (§0.4) — this function only reports that state, it does not itself decide when 48 hours have elapsed. |

Test file: `tests/services/matching.service.test.ts`

#### selectTutor

| Field | Detail |
|---|---|
| Signature | `selectTutor(callerId, callerRole, studentId, tutorId): Promise<{ cohortId, status: 'PENDING_ADMIN_APPROVAL', tutorId }>` |
| Purpose | Path A — student selects a preferred tutor, creating a booking request routed to Admin (UC-23). |
| Throws | `ApiError(400, "This tutor is not available — please choose from your current recommendations")` — `tutorId` is on the caller's `TutorExclusion` list. `ApiError(409, "You already have a pending or active match")` — an active `MatchRequest`/`Cohort` already exists for the caller. |
| Side effects | Creates a `Cohort(format: ONE_TO_ONE, status: PENDING_ADMIN_APPROVAL)` and a single `CohortMembership`; calls `cohort.service.ts` for the shared cohort-creation path rather than duplicating cohort-row construction here. |

Test file: `tests/services/matching.service.test.ts`

#### triggerNoExactMatch

| Field | Detail |
|---|---|
| Signature | `triggerNoExactMatch(callerId, callerRole, studentId): Promise<{ matchRequestId, status: 'PENDING_ADMIN_ASSIGNMENT' }>` |
| Purpose | Manual Path B entry — same resulting state as the automatic 48-hour zero-match escalation (`zeroMatchEscalation.job.ts`) and the automatic hand-off when a tutor's secondary-subject search also fails (FR-TU-008); all three paths converge on identical `MatchRequest.status`, so downstream Admin handling never needs to know which one fired. |

Test file: `tests/services/matching.service.test.ts`

#### requestGroupFormat

| Field | Detail |
|---|---|
| Signature | `requestGroupFormat(callerId, callerRole, studentId, subjectId): Promise<{ matchRequestId, status: 'SEARCHING' }>` |
| Purpose | Path C entry — system auto-match for 1-to-3/1-to-5; no search or selection UI. |
| Throws | `ApiError(400, "Use /matching/select-tutor or /matching/no-exact-match for the 1-to-1 format")` — caller's `formatPreference` is `ONE_TO_ONE`. |
| Side effects | Creates a `MatchRequest(status: SEARCHING)`; the actual grouping logic (finding/forming a compatible `Cohort`) lives in `cohort.service.ts → formOrJoinCohort`, invoked by a matching/scheduling process rather than synchronously inside this call, since group formation depends on other students' concurrent requests. |
| Edge cases | No `tutorId`, match percentage, or profile data is ever returned by this endpoint or any subsequent status check — enforced by returning a deliberately smaller DTO shape, not by the client hiding fields (API spec §3.2, no-match-information rule for group formats). |

Test file: `tests/services/matching.service.test.ts`

#### Status read path — `GET /matching/requests/me`, `GET /matching/tutors/:tutorId`

| Field | Detail |
|---|---|
| Purpose | Both are co-located read handlers in `matching.controller.ts` (per Doc 05a) backed by direct Prisma reads in this service rather than named exported functions of their own — a plain `MatchRequest.findFirst`/`TutorProfile.findUnique` respectively. |
| Throws | (requests/me) `ApiError(404, "No match request in progress")` — no active row. (tutor detail) `ApiError(404, "Tutor not found")` — doesn't exist or not yet `VERIFIED`. |
| Edge cases | `status: ZERO_MATCH_PENDING` on the requests/me read covers both a manual "No Exact Match" click and the automatic 48-hour escalation reaching the same state — the client does not distinguish which triggered it (Doc 04 MatchRequest notes). `uniqueStudentsTaught` on the tutor detail read is always a distinct-student count, computed via a `DISTINCT` aggregate over completed `CohortMembership` rows, never a raw session tally (FR-SP-025). |

Test file: `tests/services/matching.service.test.ts`

### src/controllers/matching.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| searchTutors | `matchingService.searchOneToOneTutors(...)` | 200 |
| getRecommendations | `matchingService.recommendTutorsWithMatchPercent(...)` | 200 |
| getTutorDetail | direct read, co-located | 200 |
| selectTutor | `matchingService.selectTutor(...)` | 201 |
| noExactMatch | `matchingService.triggerNoExactMatch(...)` | 200 |
| requestGroupFormat | `matchingService.requestGroupFormat(...)` | 201 |
| getMyRequestStatus | direct read, co-located | 200 |

### src/routes/matching.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /tutors/search | `authMiddleware, validate(searchTutorsQuerySchema)` | searchTutors |
| GET | /tutors/recommendations | `authMiddleware` | getRecommendations |
| GET | /tutors/:tutorId | `authMiddleware` | getTutorDetail |
| POST | /select-tutor | `authMiddleware, validate(selectTutorSchema)` | selectTutor |
| POST | /no-exact-match | `authMiddleware, validate(noExactMatchSchema)` | noExactMatch |
| POST | /group-format | `authMiddleware` | requestGroupFormat |
| GET | /requests/me | `authMiddleware` | getMyRequestStatus |

Mounted at `/matching`.

---

### src/services/cohort.service.ts (new)

#### formOrJoinCohort

| Field | Detail |
|---|---|
| Signature | `formOrJoinCohort(matchRequestId: string): Promise<{ cohortId: string; status: 'FORMING' \| 'FULL' }>` |
| Purpose | Path C grouping — finds a compatible `Cohort(status: FORMING)` matching subject/grade/format/schedule, or creates a new one, adding the student as a `CohortMembership`. |
| Side effects | Sets/refreshes `groupFormationWindowExpiresAt` on new cohort creation; `groupFormationWindow.job.ts` (§0.4) is what actually closes the window and forms the class at whatever size was reached — this function only manages membership up to that point. |
| Edge cases | Partial-formation rule (Section 7 of the SRS): a 1-to-5 cohort that closes at 3 students still bills at the standard per-student rate — pricing is a `payments-earnings` concern (`pricing.service.ts`), this function never touches price. |

Test file: `tests/services/cohort.service.test.ts` — includes the partial-group-fixed-price case (verifying no price field is set/altered here).

#### approveCohort / rejectCohort

| Field | Detail |
|---|---|
| Signature | `approveCohort(cohortId: string, adminId: string): Promise<CohortResultDTO>` · `rejectCohort(cohortId: string, adminId: string, internalReason?: string): Promise<CohortRejectionDTO>` |
| Purpose | Internal helpers invoked by `adminMatching.service.ts` (not exposed on their own route) — kept in `cohort.service.ts` since they mutate `Cohort` state directly, matching Doc 05a's ownership split (cohort data lives here; the admin-facing orchestration lives in `adminMatching.service.ts`). |
| Side effects | (reject) Re-routes per path: Path A writes a `TutorExclusion` row and spawns a fresh `MatchRequest`; Path C simply re-enters the auto-match queue with no exclusion recorded. |

Test file: `tests/services/cohort.service.test.ts`

#### tutorExitContinuity

| Field | Detail |
|---|---|
| Signature | `tutorExitContinuity(cohortId: string, reason: 'DROPOUT' \| 'SUSPENDED'): Promise<{ affectedStudentIds: string[] }>` |
| Purpose | Handles a tutor leaving an active group cohort — preserves the remaining students' cohort/group continuity where possible rather than dissolving the whole group (UC-33). |
| Side effects | 1. Ends the outgoing `Cohort` (`status: ENDED, endedReason: TUTOR_DROPOUT` or `TUTOR_SUSPENDED`, per Doc 04's Cohort notes on this flow), ending each affected `CohortMembership` (`status: ENDED, endReason: DROPPED_BY_ADMIN`). 2. **Spawns one fresh `MatchRequest` per affected `CohortMembership`** (same `studentId`, `subjectId`, `format`, `path: PATH_C`, `status: PENDING_ADMIN_ASSIGNMENT` — skipping `SEARCHING`, since this is already a re-match, not a fresh search) — this is the step that was missing in prior drafts: `manuallyAssignTutor`/`manuallyAssembleGroup` require `matchRequestIds`, and the affected students' original `MatchRequest` rows already resolved into the now-ending Cohort long ago, so fresh ones must exist before handoff is possible. 3. Collects the new `MatchRequest.id` values and hands off: if a single eligible replacement tutor can take the full remaining group, calls `adminMatching.service.ts → manuallyAssignTutor(newMatchRequestIds, tutorId, adminId)`; otherwise flags the same `newMatchRequestIds` for `manuallyAssembleGroup`, including the group-splitting case (Doc 02 §8 Cross-Path Rules, M3 worked example below). |
| Edge cases | If the automatic single-eligible-tutor check itself fails to find a candidate (not just "none assigned yet"), the freshly-spawned `MatchRequest` rows are left in `PENDING_ADMIN_ASSIGNMENT` for Admin's manual queue rather than silently retried — same visibility as any other Path C double-fail (FR-MA-017). |

Test file: `tests/services/cohort.service.test.ts` — includes the tutor-exit-group-continuity case.

#### endCohort

| Field | Detail |
|---|---|
| Signature | `endCohort(cohortId: string, reason: string): Promise<void>` |
| Side effects | Sets `Cohort.status: ENDED, endedAt, endedReason` — this is the timestamp `archiveMessageThreads.job.ts` (in `messaging`) keys its 90-day thread-archival window from. |

Test file: `tests/services/cohort.service.test.ts`

### src/controllers/cohort.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getMyCohort | `cohortService` read, co-located | 200 |
| getCohortMembers | `cohortService` read, co-located | 200 |

| Field | Detail |
|---|---|
| getMyCohort throws | none beyond common validation |
| getMyCohort edge case | No cohort yet (still `SEARCHING`) → `cohorts: []`, not an error. |
| getCohortMembers throws | `ApiError(403, "Not authorized to view this cohort")` — caller is not a current member/assigned tutor. |
| getCohortMembers behavior | Applies the profile-visibility split (Doc 02 §5.6): a student in a 1-to-3/1-to-5 cohort sees only the tutor's name + photo, never `education`/`totalStudentsCount`/`matchPercentage` — enforced by returning a distinct, smaller DTO shape from the service layer, not by the client hiding fields it received (API spec §3.2). The assigned tutor, for any format, sees each member's `studentId, firstName, grade` only. |

Test file: covered by `tests/services/cohort.service.test.ts` (read paths tested alongside the mutation functions above).

### src/routes/cohort.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /me | `authMiddleware` | getMyCohort |
| GET | /:cohortId/members | `authMiddleware` | getCohortMembers |

Mounted at `/cohorts`.

---

### src/services/adminMatching.service.ts (new)

#### listPendingApprovals

| Field | Detail |
|---|---|
| Signature | `listPendingApprovals(overdueOnly, path?, page, limit): Promise<PaginatedQueueDTO>` |
| Side effects | Reads `staleApproval.job.ts`-maintained flags (`isOverdue`, `adminOverdueNotifiedAt`, `studentDelayNotifiedAt`) — this function never computes staleness itself, only surfaces it (§0.4). |
| Edge cases | Overdue items (48h+) are always sorted to the top regardless of the `overdueOnly` filter value. A case under 48 hours returns `isOverdue: false` explicitly, never omitted. |

Test file: `tests/services/adminMatching.service.test.ts`

#### approveBooking / rejectBooking

| Field | Detail |
|---|---|
| Signature | `approveBooking(cohortId: string, adminId: string): Promise<CohortResultDTO>` · `rejectBooking(cohortId: string, adminId: string, internalReason?: string): Promise<CohortRejectionDTO>` |
| Throws | `ApiError(409, "This case is not awaiting approval")` — cohort not in a pending-approval state (both functions). |
| Side effects | (approve) Moves `Cohort.status: PENDING_PAYMENT` — schedule confirmation still requires payment completion via `payments-earnings`' `POST /payments/initiate` (FR-MA-005/010/015); this function never itself confirms the schedule. (reject) Delegates the actual state change to `cohort.service.ts → rejectCohort`; this service's role is authorization + orchestration (confirming Admin has rights, resolving which re-route path applies), not the raw state mutation. |

Test file: `tests/services/adminMatching.service.test.ts` — includes rejection re-routing cases (Path A exclusion vs. Path C re-queue).

#### manuallyAssignTutor / manuallyAssembleGroup

| Field | Detail |
|---|---|
| Signature | `manuallyAssignTutor(matchRequestIds: string[], tutorId: string, adminId: string): Promise<CohortAssignmentDTO>` · `manuallyAssembleGroup(matchRequestIds: string[], tutorId: string, adminId: string): Promise<CohortAssignmentDTO>` |
| Purpose | Path B single-request assignment and Path C double-fail group assembly, respectively — both share the same request shape and are dispatched from one controller handler based on `matchRequestIds.length` / context, but are separate service functions since group assembly must additionally validate mutual compatibility (grade/subject/schedule) across every request in the array, not just tutor eligibility. Also the mechanism behind tutor-exit group re-matching (UC-33) when `cohort.service.ts → tutorExitContinuity` can't find a single eligible tutor automatically. |
| Throws | `ApiError(400, "Selected tutor is not eligible for this assignment")` — `tutorId` not `VERIFIED`, or lacks the subject/grade match for one or more `matchRequestIds`. |
| Side effects | Goes straight to `Cohort.status: PENDING_PAYMENT` — no separate approval step, since this action **is** the Admin approval (API spec §3.2 — UC-27's flow has no additional approve click). |

Test file: `tests/services/adminMatching.service.test.ts`

### src/controllers/adminMatching.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listQueue | `adminMatchingService.listPendingApprovals(req.query.overdueOnly, req.query.path, req.query.page, req.query.limit)` | 200 |
| approve | `adminMatchingService.approveBooking(req.params.cohortId, req.user.id)` | 200 |
| reject | `adminMatchingService.rejectBooking(req.params.cohortId, req.user.id, req.body.internalReason)` | 200 |
| manualAssign | `adminMatchingService.manuallyAssignTutor(req.body.matchRequestIds, req.body.tutorId, req.user.id)` (or `manuallyAssembleGroup` when `matchRequestIds.length > 1`) | 201 |

### src/routes/adminMatching.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /queue | `authMiddleware, requireRole('ADMIN')` | listQueue |
| POST | /:cohortId/approve | `authMiddleware, requireRole('ADMIN')` | approve |
| POST | /:cohortId/reject | `authMiddleware, requireRole('ADMIN')` | reject |
| POST | /manual-assign | `authMiddleware, requireRole('ADMIN')` | manualAssign |

Mounted at `/admin/matching`.

---

### src/schemas/formatSwitch.schema.ts (new)

| Schema | Shape |
|---|---|
| requestFormatSwitchSchema | `z.object({ body: z.object({ studentId: z.string().uuid().optional(), toFormat: z.enum(['ONE_TO_ONE','ONE_TO_THREE','ONE_TO_FIVE']) }) })` |

### src/services/formatSwitch.service.ts (new)

#### requestSwitch

| Field | Detail |
|---|---|
| Signature | `requestSwitch(callerId, callerRole, studentId, toFormat): Promise<FormatSwitchResultDTO>` |
| Purpose | Cancels the current match/cohort membership and spawns a fresh matching cycle under the new format (UC-61). |
| Throws | `ApiError(409, "No active assignment to switch from")` — no active `CohortMembership`. `ApiError(400, "You are already in this format")` — `toFormat` equals current format. |
| Side effects | Ends the caller's current `CohortMembership` via `cohort.service.ts`; creates a new `MatchRequest` via `matching.service.ts` under `toFormat`; calls `refund.service.ts → createPendingRefund(paymentId, 'FORMAT_SWITCH')` (`payments-earnings`, soft dependency — **I1 fix**: creates a `PENDING` `Refund` row for Admin review, it does not auto-approve) if remaining paid sessions exist in the current billing cycle. |
| Edge cases | `refundId` is `null` only when zero remaining paid sessions existed to prorate — in that case `createPendingRefund` is never called. Other members of a group-format cohort, if any, are left entirely intact — this function never touches another member's `CohortMembership` row (UC-61 alternate flow). |

Test file: `tests/services/formatSwitch.service.test.ts` — includes the cohort-mates-unaffected case.

### src/controllers/formatSwitch.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| requestSwitch | `formatSwitchService.requestSwitch(req.user.id, req.user.role, req.body.studentId, req.body.toFormat)` | 201 |

### src/routes/formatSwitch.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | / | `authMiddleware, validate(requestFormatSwitchSchema)` | requestSwitch |

Mounted at `/format-switch`.

---

### src/jobs/groupFormationWindow.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan for `Cohort(status: FORMING)` rows whose `groupFormationWindowExpiresAt` has passed. |
| Effect | Closes the window; forms the class at whatever size was reached (partial-formation rule, Section 7) by calling `cohort.service.ts` internals directly rather than going through the request-response cycle. |
| Idempotency | A cohort already moved out of `FORMING` by a concurrent run is simply skipped on the next scan (filtered by `status` in the query itself). |

### src/jobs/zeroMatchEscalation.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan for `MatchRequest` rows with zero recommendations for a continuous 48 hours. |
| Effect | Auto-escalates to Path B (`status: ZERO_MATCH_PENDING`) via `matching.service.ts`. |
| Idempotency | Only acts on rows not already escalated — filtered by status. |

### src/jobs/staleApproval.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan of `Cohort`/`MatchRequest` rows in a pending-approval state. |
| Effect | At 48h: sets `adminOverdueNotifiedAt`, notifies Admin. At 5 days: sets `studentDelayNotifiedAt`, notifies the student. |
| Idempotency | Each notification field is set-once — a subsequent run checks the field is still `null` before re-notifying. |

---

**Next:** proceed to → [8-4. Backend: Class Delivery, Recording & Library]
