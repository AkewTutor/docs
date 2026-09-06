## Project: AKEWTutor — Frontend Specification
**Feature:** Matching & Cohorts
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `03-matching-cohorts-api.md`

**Depends on:** Accounts & Guardianship (hard — student/tutor profiles must exist before a search or recommendation means anything).

---

### 3.1 Routes

```
/student/find-tutor          → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → FindTutorPage
/student/recommendations      → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → TutorRecommendationsPage
/student/tutors/:tutorId       → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → TutorProfileViewPage
/student/group-status           → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → GroupFormatStatusPage
/student/format-switch           → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → FormatSwitchPage
/admin/matching-queue             → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → MatchingQueuePage
/admin/manual-assignment           → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → ManualAssignmentPage
```

Student and Parent share every route in this feature (both roles listed on `ProtectedRoute`) since a Parent acting for a Grade 1–5 child performs the exact same matching actions the child would if old enough — this mirrors the API's `Student|Parent` auth label (03-matching-cohorts-api.md §3.1) applying to every endpoint here.

### 3.2 Types (added to src/types/index.ts)

```typescript
export type CohortFormat = 'ONE_TO_ONE' | 'ONE_TO_THREE' | 'ONE_TO_FIVE';

export interface TutorRecommendation {
  tutorId: string;
  name: string;
  profilePictureUrl: string | null;
  matchPercentage: number;
}

export interface MatchRequest {
  id: string;
  format: CohortFormat;
  status: 'SEARCHING' | 'PENDING_APPROVAL' | 'CONFIRMED' | 'REJECTED';
  zeroMatchSince: string | null;
}

export interface Cohort {
  cohortId: string;
  format: CohortFormat;
  status: 'FORMING' | 'PENDING_APPROVAL' | 'ACTIVE' | 'ENDED';
  targetGroupSize: number;
  groupFormationWindowExpiresAt: string | null;
  membershipStatus: 'PENDING_PAYMENT' | 'ACTIVE' | 'ENDED';
}

export interface CohortMember {
  // Full detail only for the 1-to-1 tutor's own profile view (§3.3's separate endpoint);
  // group-format members are always name+photo-only, per Doc 02 §5.6's visibility split.
  id: string;
  role: 'TUTOR' | 'STUDENT';
  displayName: string;
  profilePictureUrl: string | null;
}
```

### 3.3 Hooks (src/hooks/useMatching.ts)

```typescript
export function useSearchTutors(filters: { subjectId?: string; grade?: number; day?: string; budgetMax?: string; language?: string }) {
  return useQuery({
    queryKey: [QUERY_KEYS.TUTOR_SEARCH, filters],
    queryFn: () => api.get('/matching/tutors/search', { params: filters }).then((r) => r.data),
    enabled: Object.values(filters).some(Boolean), // don't fire on an empty filter set
  });
}

export function useRecommendations(studentId?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.RECOMMENDATIONS, studentId],
    queryFn: () => api.get<{ recommendations: TutorRecommendation[]; matchRequestId: string; zeroMatchSince: string | null }>(
      '/matching/tutors/recommendations', { params: { studentId } }
    ).then((r) => r.data),
  });
}

export function useTutorFullProfile(tutorId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.TUTOR_PROFILE_VIEW, tutorId],
    queryFn: () => api.get(`/matching/tutors/${tutorId}`).then((r) => r.data),
    enabled: !!tutorId,
  });
}

export function useSelectTutor() {
  return useMutation({
    mutationFn: (body: { tutorId: string; studentId?: string }) => api.post('/matching/select-tutor', body),
  });
}

export function useNoExactMatch() {
  // UC-25 — manual Path B trigger when 1-to-1 recommendations come back empty
  return useMutation({
    mutationFn: (studentId?: string) => api.post('/matching/no-exact-match', { studentId }),
  });
}

export function useRequestGroupFormat() {
  return useMutation({
    mutationFn: (body: { format: 'ONE_TO_THREE' | 'ONE_TO_FIVE'; studentId?: string }) =>
      api.post('/matching/group-format', body),
  });
}

export function useMyMatchRequests() {
  return useQuery({
    queryKey: [QUERY_KEYS.MATCH_REQUESTS],
    queryFn: () => api.get<{ requests: MatchRequest[] }>('/matching/requests/me').then((r) => r.data),
    refetchInterval: 30_000, // status can change server-side via zeroMatchEscalation.job.ts / staleApproval.job.ts (00-api-conventions §0.4) — poll, don't assume static
  });
}
```

### 3.4 Hooks (src/hooks/useCohort.ts)

```typescript
export function useMyCohorts(studentId?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.MY_COHORTS, studentId],
    queryFn: () => api.get<{ cohorts: Cohort[] }>('/cohorts/me', { params: { studentId } }).then((r) => r.data),
    refetchInterval: 30_000, // groupFormationWindow.job.ts can flip FORMING → ACTIVE server-side (00-api-conventions §0.4)
  });
}

export function useCohortMembers(cohortId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.COHORT_MEMBERS, cohortId],
    queryFn: () => api.get<{ members: CohortMember[] }>(`/cohorts/${cohortId}/members`).then((r) => r.data),
    enabled: !!cohortId,
  });
}
```

### 3.5 Hooks (src/hooks/useAdminMatching.ts, useFormatSwitch.ts)

```typescript
export function useApprovalQueue(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.MATCHING_QUEUE, page],
    queryFn: () => api.get('/admin/matching/queue', { params: { page, limit: 20 } }).then((r) => r.data),
    refetchInterval: 20_000, // staleApproval.job.ts overdue flags update server-side
  });
}

export function useApproveCohort() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (cohortId: string) => api.post(`/admin/matching/${cohortId}/approve`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.MATCHING_QUEUE] }),
  });
}

export function useRejectCohort() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ cohortId, reason }: { cohortId: string; reason: string }) =>
      api.post(`/admin/matching/${cohortId}/reject`, { reason }),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.MATCHING_QUEUE] }),
  });
}

export function useManualAssign() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { tutorId: string; studentIds: string[]; format: CohortFormat }) =>
      api.post('/admin/matching/manual-assign', body),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.MATCHING_QUEUE] }),
  });
}

export function useRequestFormatSwitch() {
  return useMutation({
    mutationFn: (body: { cohortId: string; targetFormat: CohortFormat }) => api.post('/format-switch', body),
  });
}
```

### 3.6 Local (Non-Global) State

Per conventions §0.3 — `useState`, not Zustand:
- `TutorSearchFilters`' filter values on `FindTutorPage` — not persisted globally, reflected in URL query params so a shared/bookmarked search link works
- `activeTab` on `GroupFormatStatusPage` (waiting vs. assigned view) — derived from `useMyCohorts`' `status`, not separately tracked

### 3.7 Components

| File | Responsibility |
|---|---|
| `TutorSearchFilters.tsx` | Subject/grade/schedule/budget/language filters — **1-to-1 only**, never rendered for a group-format flow |
| `TutorRecommendationCard.tsx` | One recommendation + `matchPercentage`; "Select" action wired to `useSelectTutor` |
| `GroupAssignmentCard.tsx` | Name+photo-only assigned-tutor card for `ONE_TO_THREE`/`ONE_TO_FIVE` — must never render fields only present in the full 1-to-1 profile response |
| `NoExactMatchButton.tsx` | Shown only when `recommendations: []`; displays a live countdown from `zeroMatchSince` toward the 48h auto-escalation (UC-26), computed client-side from the timestamp, not polled separately |
| `ApprovalQueueTable.tsx` (admin) | Pending approvals; rows flagged overdue per `adminOverdueNotifiedAt` from the queue response, highlighted distinctly from normal-aging rows |
| `ManualAssignmentForm.tsx` (admin) | Tutor picker + multi-student picker for Path B/C manual assembly |

### 3.8 Notes on Job-Driven State (per 00-api-conventions §0.4)

Several states here change without any client action: `Cohort.status` (`groupFormationWindow.job.ts`), `MatchRequest.zeroMatchSince`/status escalation (`zeroMatchEscalation.job.ts`), and the queue's overdue flags (`staleApproval.job.ts`). None of these has a trigger endpoint — the corresponding hooks above (`useMyCohorts`, `useMyMatchRequests`, `useApprovalQueue`) poll via `refetchInterval` rather than the UI ever attempting to "check for updates" through a dedicated action.

---

**Next:** proceed to → [04. Class Delivery, Recording & Library Frontend]
