## Project: AKEWTutor — Frontend Function-Level Spec: Class Delivery, Recording & Library
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `04-class-delivery-library-api.md`. **Frontend spec reference:** `04-class-delivery-library-frontend.md`.

**Depends on:** Matching & Cohorts (hard — every session/recording/reschedule is scoped to a `Cohort`).

---

### Shared Pattern: Simple Query Hook

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useUpcomingSessions | GET /sessions | [SESSIONS] | — |
| useConsentStatus | GET /recording-consent/status | [RECORDING_CONSENT] | — |
| useMyRecordings | GET /recordings/me | [RECORDINGS, cohortId] | `cohortId` optional |
| useComplianceQueue | GET /admin/library/recording-compliance | [RECORDING_COMPLIANCE, page] | `refetchInterval: 30_000` |
| useCohortMaterials | GET /library/cohorts/:cohortId/materials | [LIBRARY_MATERIALS, cohortId] | `enabled: !!cohortId` |
| useAssessmentsForStudent | GET /assessments/cohort-membership/:id | [ASSESSMENTS, cohortMembershipId] | `enabled: !!cohortMembershipId` |

### Shared Pattern: Simple Mutation + Invalidation

| Hook | Endpoint | Invalidates |
|---|---|---|
| useAcknowledgeConsent | POST /recording-consent/acknowledge | [RECORDING_CONSENT] |
| useKeepPermanently | POST /recordings/:id/keep-permanently | [RECORDINGS] |
| useUploadMaterial | POST /library/materials (multipart) | [LIBRARY_MATERIALS, cohortId] |
| useProvideLink | POST /sessions/:id/link | [SESSION_DETAIL, sessionId] |
| useMarkCompleted | POST /sessions/:id/complete | [SESSION_DETAIL, sessionId] |
| useRequestReschedule | POST /reschedule | (none — see full block, has live classification) |
| useSubmitAssessment | POST /assessments | (none — fire-and-toast, form resets) |
| useUploadRecording | POST /recordings (multipart) | (none — see full block, tutor upload flow) |

---

### src/hooks/useSessions.ts — full block

#### useSession

| Field | Detail |
|---|---|
| Signature | `useSession(sessionId: string): UseQueryResult<ScheduledSession>` |
| Purpose | Wraps `GET /sessions/:sessionId`. |
| Logic | `enabled: !!sessionId`; `refetchInterval: 15_000` — `recordingMissingCheck.job.ts` flips `recordingStatus` (`PENDING → MISSING → ESCALATED`) server-side with no client trigger (00-api-conventions §0.4), so `ConductClassPage`/`RecordingIndicatorBanner` must poll rather than assume a static status once the session starts. |
| Test file | `tests/hooks/useSessions.test.ts` |

---

### src/hooks/useRecordingConsent.ts, useRecordings.ts — full blocks

#### useUploadRecording

| Field | Detail |
|---|---|
| Signature | `useUploadRecording(): UseMutationResult<{ recordingId: string }, AxiosError, { sessionId: string; file: File }>` |
| Purpose | Wraps `POST /recordings` (multipart). |
| Logic | Builds a `FormData` from `sessionId` + `file`; no client-side file-type/size validation is specified beyond what the file picker's `accept` attribute restricts, since the backend's own limits (if any) are authoritative and not documented at the frontend layer. |
| Side effects | No automatic invalidation of `[RECORDINGS]` — a tutor's manual upload (fallback path, if the automated recording pipeline missed it) is a rare action whose success screen explicitly tells the caller to check back on the session detail page, rather than the mutation attempting to guess which cached list to refresh. |
| Test file | `tests/hooks/useRecordings.test.ts` |

#### useSignedUrl

| Field | Detail |
|---|---|
| Signature | `useSignedUrl(recordingId: string \| null): UseQueryResult<{ url: string }>` |
| Purpose | Wraps `GET /recordings/:id/signed-url`. |
| Logic | `enabled: !!recordingId`; `retry: false` — a `404` here means the recording is genuinely past its 90-day retention and not marked "keep permanently" (§0.3), a real not-found rather than a transient failure, so retrying with the same ID is pointless and would only delay the correct `EmptyState` from appearing. |
| Test file | `tests/hooks/useRecordings.test.ts` |

---

### src/hooks/useReschedule.ts — full block

#### useRequestReschedule

| Field | Detail |
|---|---|
| Signature | `useRequestReschedule(): UseMutationResult<RescheduleRequest, AxiosError, { sessionId: string; requestedNewTime: string; reason: string }>` |
| Purpose | Wraps `POST /reschedule`. |
| Side effects | On success, the calling page navigates back to `/student/upcoming-classes` (or the tutor equivalent) with a confirmation toast that includes the server's authoritative `classification` (`FREE`/`SAME_DAY_MISS`) — **not** the client-computed preview from `RescheduleForm` (below), since the two can disagree on a clock-drift edge case and the server value is what actually persists (§4.7). |
| Test file | `tests/hooks/useReschedule.test.ts` |

### src/lib/classifyReschedule.ts (new util — full block)

| Field | Detail |
|---|---|
| Signature | `classifyReschedule(sessionScheduledStart: string, requestedAt: Date): 'FREE' \| 'SAME_DAY_MISS'` |
| Purpose | Client-side preview of the backend's 12-hour boundary rule, used only for live UX feedback in `RescheduleForm` — never the value actually submitted or trusted post-response. |
| Logic | 1. Compute `hoursUntilSession = (new Date(sessionScheduledStart).getTime() - requestedAt.getTime()) / 3_600_000`. 2. Return `'FREE'` if `hoursUntilSession >= 12`, else `'SAME_DAY_MISS'`. |
| Edge cases | A negative `hoursUntilSession` (attempting to reschedule a session already in progress or past) still resolves to `'SAME_DAY_MISS'` under this formula — this util does not separately guard against that case, since the reschedule form itself should already prevent selecting a past/in-progress session as the target before this function is ever called. |
| Test file | `tests/lib/classifyReschedule.test.ts` |

---

### src/pages/student/UpcomingClassesPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/upcoming-classes` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Behavior | `useUpcomingSessions()`; renders one `SessionCard` per entry, sorted by `scheduledStart` ascending (list is not re-sorted client-side beyond this — the backend's own ordering, if any, is assumed already ascending, and this is a stable no-op sort applied defensively). |

**States:** loading · empty (`sessions: []`, no classes scheduled yet) · success

### src/components/SessionCard.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ session: ScheduledSession }` |
| Behavior | 1. Renders `CountdownTimer` (foundation component) toward `scheduledStart`. 2. "Join" button `disabled` until `jitsiLinkUrl` is non-null — this is the sole gate; there is no separate time-window check (e.g. "only enable 10 minutes before"), since the backend controls when the link is actually sent (`jitsiLinkSentAt`). 3. Renders a `RecordingIndicatorBanner`-style small badge reflecting `recordingStatus` once the session's `scheduledStart` has passed. 4. A "Request reschedule" link routes to `/reschedule/:sessionId`, hidden once `status` is `COMPLETED` or `MISSED`. |
| Test file | `tests/components/SessionCard.test.tsx` |

### src/pages/tutor/ConductClassPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/tutor/conduct-class/:sessionId` — `ProtectedRoute(['TUTOR'])` + `DashboardLayout(TutorSidebar)` |
| Behavior | 1. `useSession(sessionId)` (polling, per above). 2. If `jitsiLinkUrl` is `null`, renders a "Generate & submit Jitsi link" form (the tutor pastes the link they created on the public Jitsi instance, per `00-api-conventions.md` §0.5's integration note — this app never calls a Jitsi API itself); submits via `useProvideLink`. 3. Once a link exists, renders it plus a "Mark session completed" button wired to `useMarkCompleted`, enabled only after `scheduledEnd` has passed (computed client-side from `Date.now() >= new Date(session.scheduledEnd)`), so a tutor cannot mark a session complete before it's actually over. 4. Renders `RecordingIndicatorBanner` once consent has been acknowledged for the cohort (cross-references `useConsentStatus`). |

**States:** loading · error · success (link form, or link + complete action, depending on `jitsiLinkUrl`)

### src/components/RecordingIndicatorBanner.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ visible: boolean }` |
| Behavior | Purely presentational — a persistent "This session is being recorded" banner. No polling or data-fetching of its own (§4.6); the host page decides `visible` from its own `useConsentStatus`/session state. |

---

### src/pages/RecordingConsentPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/recording-consent` — `ProtectedRoute('any')` + `DashboardLayout` |
| Behavior | 1. `useConsentStatus()`. 2. If `acknowledged`, renders a simple confirmation with `acknowledgedAt`. 3. If not, renders the consent text + an "Acknowledge" button wired to `useAcknowledgeConsent`. |

**States:** loading · success (acknowledged view, or acknowledge-action view)

### src/pages/LibraryPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/library` — `ProtectedRoute('any')` + `DashboardLayout` |
| Local state | `selectedCohortId` — the page first lists the caller's cohorts (via `useMyCohorts`, 8-3) as tabs/a dropdown, then loads that cohort's recordings + materials. |
| Behavior | 1. `useMyRecordings(selectedCohortId)` + `useCohortMaterials(selectedCohortId)`, rendered as two sections. 2. Each recording row opens `RecordingPlayer` (below) in a modal/panel. 3. Tutor-only: `MaterialUploadForm` rendered above the materials list, gated on `auth.store.ts`'s `user.role === 'TUTOR'`. 4. A "Keep permanently" action per recording (visible to any participant, not tutor-only, since the consequence — avoiding the 90-day deletion — benefits the whole cohort) wired to `useKeepPermanently`. |

**States:** loading · empty (no cohorts yet → prompt to check back once enrolled; cohort selected but `recordings`/`materials` empty → per-section `EmptyState`) · success

### src/components/RecordingPlayer.tsx (new — full block, per frontend spec §4.6/§4.7)

| Field | Detail |
|---|---|
| Props | `{ recordingId: string }` |
| Behavior | 1. `useSignedUrl(recordingId)`. 2. While loading, renders a skeleton — never a blank pane (§4.7's `CellDetailPanel`-style graceful-degradation note, carried over as the house pattern for any panel with an async-resolved primary asset). 3. On success, renders a `<video>` element sourced from `data.url`. 4. On a `404` (query `isError` with `retry: false` already set on the hook), renders `EmptyState` (foundation component) with "Recording no longer available" — never a generic error banner, since this is an expected, documented outcome (§0.3), not a bug. |
| Test file | `tests/components/RecordingPlayer.test.tsx` |

### src/components/MaterialUploadForm.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ cohortId: string }` |
| Local state | `title: string`, `file: File \| null`. |
| Behavior | On submit, `useUploadMaterial().mutate({ cohortId, title, file })`; disabled until both fields are set. |

### src/pages/RequestReschedulePage.tsx, src/components/RescheduleForm.tsx (new)

| Field | Detail |
|---|---|
| Route | `/reschedule/:sessionId` — `ProtectedRoute('any')` + `DashboardLayout` |
| `RescheduleForm` local state | `requestedNewTime: Date \| null`, `reason: string`. |
| Behavior (form) | 1. As `requestedNewTime` changes, computes `classifyReschedule(session.scheduledStart, new Date())` live and renders the resulting `FREE`/`SAME_DAY_MISS` badge next to the picker (§4.6) — this is a UX preview only. 2. On submit, `useRequestReschedule().mutate({ sessionId, requestedNewTime, reason })`. 3. After the mutation resolves, the confirmation screen reads `classification` from the **response**, reconciling to it even if it differs from the live preview shown a moment earlier (§4.7) — the form does not silently keep showing its own pre-submit guess. |

**States:** idle (picker + live badge) · submitting · error · success (confirmation showing server `classification`)

---

### src/pages/tutor/WeeklyAssessmentPage.tsx, src/components/AssessmentForm.tsx (new)

| Field | Detail |
|---|---|
| Route | `/tutor/assessments/:cohortId` — `ProtectedRoute(['TUTOR'])` + `DashboardLayout(TutorSidebar)` |
| Behavior | 1. Lists the cohort's students (via `useCohortMembers`, 8-3), one `AssessmentForm` per student per current week. 2. Each form submits independently via `useSubmitAssessment`; a form that has already been submitted this week (checked against `useAssessmentsForStudent` for that student's `cohortMembershipId`, filtered to `weekOf === current week`) renders read-only with an "Edit" affordance rather than allowing a silent duplicate submission — there is no dedicated "has this week's assessment been submitted" endpoint, so this check is done client-side against the existing list query, and is therefore a UX convenience rather than a guaranteed duplicate-prevention (the backend's own uniqueness rule, if any, remains authoritative). |

**States:** loading (cohort members) · success (per-student forms)

### src/pages/student/ProgressPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/progress` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Behavior | `useAssessmentsForStudent(cohortMembershipId)` (per active cohort, tab-switchable if the student has more than one); renders a chronological list of `scoreSummary`/`feedback` entries. |

**States:** loading · empty (no assessments yet) · success

---

### src/pages/admin/RecordingComplianceQueuePage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/recording-compliance` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | `useComplianceQueue(page)`; rows render `recordingStatus` via `StatusBadge`, with `ESCALATED` rows visually distinguished (per `recordingMissingCheck.job.ts`'s 24h escalation, §0.4) from `MISSING` (2h) rows — same severity-tiering convention as `ApprovalQueueTable` (8-3) and `PlatformStatsGrid` (8-8), kept visually consistent across the three admin queues in the app. |

**States:** loading · empty (queue clear) · success

---

**Next:** proceed to → [8-5. Frontend: In-Platform Messaging]
