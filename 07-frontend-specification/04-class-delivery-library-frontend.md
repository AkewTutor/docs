## Project: AKEWTutor — Frontend Specification
**Feature:** Class Delivery, Recording & Library
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `04-class-delivery-library-api.md`

**Depends on:** Matching & Cohorts (hard — every session/recording/reschedule is scoped to a `Cohort`).

---

### 4.1 Routes

```
/student/upcoming-classes   → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → UpcomingClassesPage
/tutor/conduct-class/:sessionId → ProtectedRoute(['TUTOR']) → DashboardLayout(TutorSidebar) → ConductClassPage
/library                     → ProtectedRoute('any') → DashboardLayout → LibraryPage
/recording-consent            → ProtectedRoute('any') → DashboardLayout → RecordingConsentPage
/reschedule/:sessionId          → ProtectedRoute('any') → DashboardLayout → RequestReschedulePage
/tutor/assessments/:cohortId     → ProtectedRoute(['TUTOR']) → DashboardLayout(TutorSidebar) → WeeklyAssessmentPage
/student/progress                 → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → ProgressPage
/admin/recording-compliance         → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → RecordingComplianceQueuePage
```

`RecordingConsentPage` and `RequestReschedulePage` are listed under `ProtectedRoute('any')` because both a Student/Parent and a Tutor can reach them (consent is acknowledged by every participant per UC-45; a reschedule can be requested by either side per UC-53/54's shared auth label in the API).

### 4.2 Types (added to src/types/index.ts)

```typescript
export type SessionStatus = 'SCHEDULED' | 'COMPLETED' | 'MISSED' | 'PAYMENT_PAUSE_RESCHEDULED';
export type RecordingStatus = 'PENDING' | 'AVAILABLE' | 'MISSING' | 'ESCALATED';

export interface ScheduledSession {
  id: string;
  cohortId: string;
  scheduledStart: string;
  scheduledEnd: string;
  jitsiLinkUrl: string | null;
  jitsiLinkSentAt: string | null;
  status: SessionStatus;
  isMakeup: boolean;
  makeupForSessionId: string | null;
  recordingStatus: RecordingStatus;
}

export interface RecordingConsentStatus {
  acknowledged: boolean;
  acknowledgedAt: string | null;
}

export interface Recording {
  id: string;
  sessionId: string;
  cohortId: string;
  keepPermanently: boolean;
  expiresAt: string | null; // null once keepPermanently is true
  createdAt: string;
}

export interface LibraryMaterial {
  id: string;
  cohortId: string;
  title: string;
  fileUrl: string;
  uploadedAt: string;
}

export interface RescheduleRequest {
  id: string;
  sessionId: string;
  requestedNewTime: string;
  classification: 'FREE' | 'SAME_DAY_MISS'; // per the 12-hour boundary rule
  status: 'PENDING' | 'APPROVED' | 'REJECTED';
}

export interface WeeklyAssessment {
  id: string;
  studentId: string;
  cohortId: string;
  weekOf: string;
  scoreSummary: string;
  feedback: string;
  createdAt: string;
}
```

### 4.3 Hooks (src/hooks/useSessions.ts)

```typescript
export function useUpcomingSessions() {
  return useQuery({
    queryKey: [QUERY_KEYS.SESSIONS],
    queryFn: () => api.get<{ sessions: ScheduledSession[] }>('/sessions').then((r) => r.data),
  });
}

export function useSession(sessionId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.SESSION_DETAIL, sessionId],
    queryFn: () => api.get<ScheduledSession>(`/sessions/${sessionId}`).then((r) => r.data),
    enabled: !!sessionId,
    refetchInterval: 15_000, // recordingMissingCheck.job.ts flips recordingStatus server-side (00-api-conventions §0.4)
  });
}

export function useProvideLink() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ sessionId, jitsiLinkUrl }: { sessionId: string; jitsiLinkUrl: string }) =>
      api.post(`/sessions/${sessionId}/link`, { jitsiLinkUrl }),
    onSuccess: (_d, vars) => qc.invalidateQueries({ queryKey: [QUERY_KEYS.SESSION_DETAIL, vars.sessionId] }),
  });
}

export function useMarkCompleted() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (sessionId: string) => api.post(`/sessions/${sessionId}/complete`),
    onSuccess: (_d, sessionId) => qc.invalidateQueries({ queryKey: [QUERY_KEYS.SESSION_DETAIL, sessionId] }),
  });
}
```

### 4.4 Hooks (src/hooks/useRecordingConsent.ts, useRecordings.ts, useLibrary.ts)

```typescript
export function useConsentStatus() {
  return useQuery({
    queryKey: [QUERY_KEYS.RECORDING_CONSENT],
    queryFn: () => api.get<RecordingConsentStatus>('/recording-consent/status').then((r) => r.data),
  });
}

export function useAcknowledgeConsent() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: () => api.post('/recording-consent/acknowledge'),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.RECORDING_CONSENT] }),
  });
}

export function useUploadRecording() {
  return useMutation({
    mutationFn: (body: { sessionId: string; file: File }) => {
      const form = new FormData();
      form.append('sessionId', body.sessionId);
      form.append('file', body.file);
      return api.post('/recordings', form, { headers: { 'Content-Type': 'multipart/form-data' } });
    },
  });
}

export function useMyRecordings(cohortId?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.RECORDINGS, cohortId],
    queryFn: () => api.get<{ recordings: Recording[] }>('/recordings/me', { params: { cohortId } }).then((r) => r.data),
  });
}

export function useSignedUrl(recordingId: string | null) {
  return useQuery({
    queryKey: [QUERY_KEYS.SIGNED_URL, recordingId],
    queryFn: () => api.get<{ url: string }>(`/recordings/${recordingId}/signed-url`).then((r) => r.data),
    enabled: !!recordingId,
    retry: false, // a 404 here means the recording is genuinely gone past retention (00-api-conventions §0.3) — don't retry into a false loading state
  });
}

export function useKeepPermanently() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (recordingId: string) => api.post(`/recordings/${recordingId}/keep-permanently`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.RECORDINGS] }),
  });
}

export function useComplianceQueue(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.RECORDING_COMPLIANCE, page],
    queryFn: () => api.get('/admin/library/recording-compliance', { params: { page, limit: 20 } }).then((r) => r.data),
    refetchInterval: 30_000,
  });
}

export function useCohortMaterials(cohortId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.LIBRARY_MATERIALS, cohortId],
    queryFn: () => api.get<{ materials: LibraryMaterial[] }>(`/library/cohorts/${cohortId}/materials`).then((r) => r.data),
    enabled: !!cohortId,
  });
}

export function useUploadMaterial() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { cohortId: string; title: string; file: File }) => {
      const form = new FormData();
      form.append('cohortId', body.cohortId);
      form.append('title', body.title);
      form.append('file', body.file);
      return api.post('/library/materials', form, { headers: { 'Content-Type': 'multipart/form-data' } });
    },
    onSuccess: (_d, vars) => qc.invalidateQueries({ queryKey: [QUERY_KEYS.LIBRARY_MATERIALS, vars.cohortId] }),
  });
}
```

### 4.5 Hooks (src/hooks/useReschedule.ts, useWeeklyAssessment.ts)

```typescript
export function useRequestReschedule() {
  return useMutation({
    mutationFn: (body: { sessionId: string; requestedNewTime: string; reason: string }) =>
      api.post<RescheduleRequest>('/reschedule', body).then((r) => r.data),
  });
}

export function useSubmitAssessment() {
  return useMutation({
    mutationFn: (body: { studentId: string; cohortId: string; scoreSummary: string; feedback: string }) =>
      api.post<WeeklyAssessment>('/assessments', body).then((r) => r.data),
  });
}

export function useAssessmentsForStudent(cohortMembershipId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.ASSESSMENTS, cohortMembershipId],
    queryFn: () => api.get<{ assessments: WeeklyAssessment[] }>(`/assessments/cohort-membership/${cohortMembershipId}`).then((r) => r.data),
    enabled: !!cohortMembershipId,
  });
}
```

### 4.6 Components

| File | Responsibility |
|---|---|
| `SessionCard.tsx` | One scheduled session; renders `CountdownTimer` (foundation component) toward `scheduledStart`, "Join" button enabled only once `jitsiLinkUrl` is present |
| `RecordingIndicatorBanner.tsx` | Persistent banner shown during an active session once consent is acknowledged — purely presentational, no polling of its own |
| `RecordingPlayer.tsx` | Consumes `useSignedUrl`; shows `EmptyState` (foundation component) on a 404 ("Recording no longer available") rather than a generic error |
| `MaterialUploadForm.tsx` | Tutor-only upload form (title + file), wired to `useUploadMaterial` |
| `RescheduleForm.tsx` | New-time picker; **computes and displays the FREE vs. SAME_DAY_MISS classification live client-side** as the picker value changes, based on the 12-hour boundary rule, before the request is even submitted — mirrors backend logic for UX responsiveness, does not replace the backend's authoritative classification in the response |
| `AssessmentForm.tsx` | Tutor feedback + score-summary entry, one per student per week |

### 4.7 Form/Display Notes

- `RescheduleForm`'s live FREE/SAME_DAY_MISS preview is a UI convenience only — the actual `classification` in the `RescheduleRequest` response (§4.2) is what persists and drives make-up/refund consequences; if the two ever disagree (e.g. the client's clock drifts), the server value wins and the form should reconcile to it after submit.
- `CellDetailPanel`-style graceful degradation (per the sibling reference project's pattern) applies to `RecordingPlayer`: if `useSignedUrl` is still loading, show a skeleton; once resolved, either the player or the "no longer available" `EmptyState` — never a blank pane.

---

**Next:** proceed to → [05. Messaging Frontend]
