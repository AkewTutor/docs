## Project: AKEWTutor — Frontend Function-Level Spec: Matching & Cohorts
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `03-matching-cohorts-api.md`. **Frontend spec reference:** `03-matching-cohorts-frontend.md`.

**Depends on:** Accounts & Guardianship (hard — a search/recommendation is meaningless without an existing student/tutor profile).

**Links back to:** [07-frontend-specification/03-matching-cohorts-frontend.md], [05b. Frontend Folder & File Structure §3]
**Links forward to:** [9-3. Frontend Test Spec: Matching & Cohorts]

---

### Shared Pattern: Simple Query Hook

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useTutorFullProfile | GET /matching/tutors/:tutorId | [TUTOR_PROFILE_VIEW, tutorId] | `enabled: !!tutorId` |
| useCohortMembers | GET /cohorts/:cohortId/members | [COHORT_MEMBERS, cohortId] | `enabled: !!cohortId` |

### Shared Pattern: Simple Mutation + Invalidation

| Hook | Endpoint | Invalidates |
|---|---|---|
| useApproveCohort | POST /admin/matching/:cohortId/approve | [MATCHING_QUEUE] |
| useRejectCohort | POST /admin/matching/:cohortId/reject | [MATCHING_QUEUE] |
| useManualAssign | POST /admin/matching/manual-assign | [MATCHING_QUEUE] |
| useSelectTutor | POST /matching/select-tutor | (none — see full block, drives navigation) |
| useNoExactMatch | POST /matching/no-exact-match | (none — see full block, drives navigation) |
| useRequestGroupFormat | POST /matching/group-format | (none — fire-and-navigate to status page) |
| useRequestFormatSwitch | POST /format-switch | (none — fire-and-toast, no local list to invalidate) |

---

### src/hooks/useMatching.ts — full blocks

#### useSearchTutors

| Field | Detail |
|---|---|
| Signature | `useSearchTutors(filters: { subjectId?: string; grade?: number; day?: string; budgetMax?: string; language?: string }): UseQueryResult<{ tutors: TutorSearchResult[] }>` |
| Purpose | Wraps `GET /matching/tutors/search`. |
| Logic | `enabled: Object.values(filters).some(Boolean)` — the query does not fire on an empty filter set (frontend spec §3.3), so `FindTutorPage` shows a "start by selecting a filter" empty state rather than an unfiltered full-catalog request no endpoint is designed to serve efficiently. |
| Edge cases | Every filter change re-keys the query (`filters` is part of the query key) — this is a fresh request per change, not a client-side re-filter of a cached full list, since the backend owns the matching/scoring logic. |
| Test file | `tests/hooks/useMatching.test.ts` |

#### useRecommendations

| Field | Detail |
|---|---|
| Signature | `useRecommendations(studentId?: string): UseQueryResult<{ recommendations: TutorRecommendation[]; matchRequestId: string; zeroMatchSince: string \| null }>` |
| Purpose | Wraps `GET /matching/tutors/recommendations`. |
| Logic | No `enabled` gate — fires on mount, since this is the 1-to-1 landing view once a student/parent reaches it. `recommendations: []` is treated as a normal success response, not an error (per §0.3) — `TutorRecommendationsPage` renders `NoExactMatchButton` in that case rather than an error state. |
| Test file | `tests/hooks/useMatching.test.ts` |

#### useSelectTutor

| Field | Detail |
|---|---|
| Signature | `useSelectTutor(): UseMutationResult<{ cohortId: string; status: string }, AxiosError, { tutorId: string; studentId?: string }>` |
| Purpose | Wraps `POST /matching/select-tutor`. |
| Side effects | No query invalidation on its own — the caller (`TutorRecommendationCard`'s parent page) is responsible for navigating to `/student/group-status` (or the 1-to-1 equivalent) on success, since the resulting `Cohort` is picked up fresh by `useMyCohorts` (below) rather than patched into an existing cache entry. |
| Edge cases | A `409` (tutor already at capacity, or the recommendation has since expired) renders as a toast prompting the person back to the recommendation list — the stale card is not silently retried. |
| Test file | `tests/hooks/useMatching.test.ts` |

#### useNoExactMatch

| Field | Detail |
|---|---|
| Signature | `useNoExactMatch(): UseMutationResult<{ matchRequestId: string; status: 'SEARCHING' }, AxiosError, string \| undefined>` |
| Purpose | Wraps `POST /matching/no-exact-match` — the manual Path B trigger (UC-25). |
| Side effects | On success, the calling page (`NoExactMatchButton`'s host, `TutorRecommendationsPage`) navigates to `/student/group-status`, where `useMyMatchRequests`/`useMyCohorts` (below) pick up the now-`SEARCHING` state — no local cache write here, since the source of truth for that status is the polled query. |
| Test file | `tests/hooks/useMatching.test.ts` |

#### useMyMatchRequests

| Field | Detail |
|---|---|
| Signature | `useMyMatchRequests(): UseQueryResult<{ requests: MatchRequest[] }>` |
| Purpose | Wraps `GET /matching/requests/me`. |
| Logic | `refetchInterval: 30_000` — status can change server-side via `zeroMatchEscalation.job.ts`/`staleApproval.job.ts` with no client-triggered event (00-api-conventions §0.4), so this must poll rather than rely on a one-time fetch. |
| Test file | `tests/hooks/useMatching.test.ts` |

---

### src/hooks/useCohort.ts — full block

#### useMyCohorts

| Field | Detail |
|---|---|
| Signature | `useMyCohorts(studentId?: string): UseQueryResult<{ cohorts: Cohort[] }>` |
| Purpose | Wraps `GET /cohorts/me`. |
| Logic | `refetchInterval: 30_000` — `groupFormationWindow.job.ts` can flip a `Cohort.status` from `FORMING` to `ACTIVE` with no client action (§0.4); `GroupFormatStatusPage`'s tab derivation (§3.6) reads this poll's result directly rather than tracking its own "has it changed" state. |
| Test file | `tests/hooks/useCohort.test.ts` |

---

### src/hooks/useAdminMatching.ts, useFormatSwitch.ts — full block

#### useApprovalQueue

| Field | Detail |
|---|---|
| Signature | `useApprovalQueue(page = 1): UseQueryResult<{ items: MatchingQueueItem[]; page: number; limit: number; total: number }>` |
| Logic | `refetchInterval: 20_000` — `staleApproval.job.ts` flips `adminOverdueNotifiedAt` server-side (§0.4); `ApprovalQueueTable` (below) relies on this poll to move a row into its overdue visual state without a manual refresh. |
| Test file | `tests/hooks/useAdminMatching.test.ts` |

---

### src/pages/student/FindTutorPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/find-tutor` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Local state | `filters` object (§3.6) reflected in the URL's `?` query params via `useSearchParams`, not `useState` alone — so a shared/bookmarked search link reproduces the same result set. |
| Behavior | 1. `TutorSearchFilters` writes into the URL params on change. 2. `useSearchTutors(filters)` re-fires per param change. 3. Results render as a grid/list of tutor summary cards, each linking to `/student/tutors/:tutorId`. |

**States:** idle (no filters yet — `EmptyState` prompting selection) · loading · empty (filters set, zero results) · success

### src/pages/student/TutorRecommendationsPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/recommendations` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Behavior | 1. `useRecommendations(studentId)`. 2. Renders one `TutorRecommendationCard` per entry. 3. If `recommendations.length === 0`, renders `NoExactMatchButton` instead of the grid, passing `zeroMatchSince` through for its live countdown. 4. `TutorRecommendationCard`'s "Select" action calls `useSelectTutor().mutate({ tutorId, studentId })`; on success, `navigate('/student/group-status')`. |

**States:** loading · empty (renders `NoExactMatchButton`, not a blocked page) · success (recommendation grid)

### src/components/NoExactMatchButton.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ zeroMatchSince: string \| null; onTrigger: () => void }` |
| Local state | `secondsElapsed`, ticked by a 1s `setInterval`, computed from `Date.now() - new Date(zeroMatchSince).getTime()` at mount and each tick — not re-derived from a server round-trip. |
| Behavior | 1. Displays a live countdown toward the 48h auto-escalation threshold (UC-26): `remaining = 48h - elapsed`, formatted as `Hh Mm`. 2. If `zeroMatchSince` is `null` (recommendations just came back empty this instant, no escalation clock started yet), renders the button without a countdown. 3. Clicking the button calls `onTrigger` — wired at the call site to `useNoExactMatch().mutate(studentId)`, then navigates to `/student/group-status` on success. |
| Edge cases | If `remaining <= 0` (the 48h has already elapsed but the poll hasn't caught up to the server's auto-escalation yet), the countdown clamps to `0h 0m` rather than showing a negative duration, and the button copy shifts to "Escalating automatically" to avoid implying manual action is still required. |
| Test file | `tests/components/NoExactMatchButton.test.tsx` |

### src/pages/student/TutorProfileViewPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/tutors/:tutorId` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Behavior | `useTutorFullProfile(tutorId)`; "Select this tutor" action same as the recommendation card's, wired to `useSelectTutor`. |

**States:** loading · error (404 — tutor no longer available) · success

### src/pages/student/GroupFormatStatusPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/group-status` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Local state | `activeTab: 'waiting' \| 'assigned'` — **derived**, not separately tracked (§3.6): computed as `cohorts.some(c => c.status === 'FORMING' || c.status === 'PENDING_APPROVAL') ? 'waiting' : 'assigned'` from `useMyCohorts()`'s result each render, so the tab flips on its own once the poll observes a status change, with no explicit "refresh" affordance needed. |
| Behavior | 1. `useMyCohorts(studentId)`, `useMyMatchRequests()`. 2. "Waiting" tab shows `MatchRequest`/`Cohort` status with a `groupFormationWindowExpiresAt` countdown (`CountdownTimer`, foundation component) when present. 3. "Assigned" tab renders `GroupAssignmentCard` per active cohort member. 4. A "Request format switch" action links to `/student/format-switch`, pre-filled with the current cohort's `format`. |

**States:** loading · empty (no requests/cohorts yet — first-time visitor) · success (tab content per derived `activeTab`)

### src/components/GroupAssignmentCard.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ member: CohortMember }` |
| Behavior | Renders `displayName` + `profilePictureUrl` only. |
| Edge cases | **Must never render any field beyond name+photo** even if a future API response happens to include more (e.g. a tutor's qualifications) — this is the deliberate group-format visibility floor from Doc 02 §5.6, and the component's own prop type (`CohortMember`, not `TutorProfile`) is what enforces this at the type level, not a runtime filter that could silently be bypassed. |
| Test file | `tests/components/GroupAssignmentCard.test.tsx` |

### src/pages/student/FormatSwitchPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/format-switch` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Local state | `targetFormat`, pre-filled from the navigating cohort's current format's "next" option (e.g. currently `ONE_TO_ONE` → default target `ONE_TO_THREE`) but changeable. |
| Behavior | On submit, `useRequestFormatSwitch().mutate({ cohortId, targetFormat })`; success renders a confirmation and links back to `/student/group-status`. |

**States:** idle · submitting · error · success

---

### src/pages/admin/MatchingQueuePage.tsx, src/components/ApprovalQueueTable.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/matching-queue` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| `ApprovalQueueTable` props | `{ items: MatchingQueueItem[]; onApprove: (cohortId) => void; onReject: (cohortId, reason) => void }` |
| Behavior (table) | Rows render a distinct visual treatment (background tint + icon, via `StatusBadge`) when `adminOverdueNotifiedAt` is non-null, computed purely from the field already present in the poll response — no separate overdue-check logic client-side, since `staleApproval.job.ts` is the sole source of that flag (§0.4/§3.8). |
| Behavior (page) | `useApprovalQueue(page)`; wires row actions to `useApproveCohort`/`useRejectCohort` (reject requires a reason, same required-reason pattern as 8-2's tutor rejection). |

**States:** loading · empty (queue clear — steady state) · success

### src/pages/admin/ManualAssignmentPage.tsx, src/components/ManualAssignmentForm.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/manual-assignment` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| `ManualAssignmentForm` local state | `tutorId: string \| null`, `studentIds: string[]`, `format: CohortFormat`. |
| Behavior | 1. Tutor picker (searches via `useSearchTutors`-style filtering, reused rather than a separate hook — this page is an admin-facing composition of the same tutor-lookup primitive rather than a new endpoint). 2. Multi-select student picker constrained to `studentIds.length` matching `format`'s expected size (`ONE_TO_ONE` → exactly 1, `ONE_TO_THREE` → up to 3, `ONE_TO_FIVE` → up to 5) — enforced as a disabled "Assign" button rather than only a post-submit error, mirroring the same UX-guard convention used for `SubjectRankingForm` (8-2) and `PricingConfigForm` (8-7). 3. On submit, `useManualAssign().mutate({ tutorId, studentIds, format })`. |

**States:** idle · submitting · error · success (confirmation, form resets)

---

### 3.8 Notes on Job-Driven State (carried forward from frontend spec §3.8)

`useMyCohorts`, `useMyMatchRequests`, and `useApprovalQueue` are the three polling hooks in this feature. None of the three has a paired "check now" action anywhere in the UI — there is deliberately no refresh button next to `GroupFormatStatusPage`'s status card or `MatchingQueuePage`'s table, since a manual refresh would imply there's something to trigger, and per `00-api-conventions.md` §0.4 there isn't.

---

**Next:** proceed to → [8-4. Frontend: Class Delivery, Recording & Library]
