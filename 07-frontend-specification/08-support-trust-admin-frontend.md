## Project: AKEWTutor — Frontend Specification
**Feature:** Support, Trust & Admin Reporting
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `08-support-trust-admin-api.md`

**Depends on:** Shared Config, Messaging, Class Delivery & Library (hard); soft-integrates with Payments & Earnings (refund action) and Accounts & Guardianship (suspension action) — neither soft integration surfaces as a direct client-side call from this feature's own hooks (both happen server-side as part of `PATCH /admin/disputes/:complaintId`).

**Links back to:** [0. Frontend Conventions], [06-api/08-support-trust-admin-api.md], [05b. Frontend Folder & File Structure §8]
**Links forward to:** [8-8. Frontend Function-Level Spec: Support, Trust & Admin Reporting]

---

### 8.1 Routes

```
/complaints                 → ProtectedRoute('any') → DashboardLayout → SubmitComplaintPage
/support                     → ProtectedRoute('any') → DashboardLayout → SupportContactPage
/admin/disputes                → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → DisputeQueuePage
/admin/reports                   → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → PlatformReportsPage
```

`SupportContactPage` renders static content from `GET /support/contact` — it is deliberately not a form (per Doc 01 §1.7 Assumption #4: support contact is manual, not an in-app ticketing flow). It should not be confused with `SubmitComplaintPage`, which *is* a form and *does* create a record — the two exist because "reach a human for anything" (UC-67/69) and "file something Admin needs to formally resolve" (UC-66) are genuinely different needs, not two ways of doing the same thing.

### 8.2 Types (added to src/types/index.ts)

```typescript
export type ComplaintCategory = 'SESSION_ISSUE' | 'TUTOR_CONDUCT' | 'PAYMENT_ISSUE' | 'MESSAGE_ISSUE' | 'OTHER';
export type ComplaintStatus = 'OPEN' | 'UNDER_REVIEW' | 'RESOLVED' | 'DISMISSED';
export type ResolutionAction = 'NO_ACTION' | 'WARNING_ISSUED' | 'REFUND_ISSUED' | 'TUTOR_SUSPENDED';

export interface ComplaintSummary {
  id: string;
  category: ComplaintCategory;
  status: ComplaintStatus;
  createdAt: string;
  resolvedAt: string | null;
}

export interface ComplaintDetail extends ComplaintSummary {
  description: string;
  resolutionAction: ResolutionAction | null;
}

export interface AdminComplaintDetail extends ComplaintDetail {
  reporterId: string;
  reporterRole: 'STUDENT' | 'PARENT' | 'TUTOR';
  relatedCohortId: string | null;
  relatedSessionId: string | null;
  relatedPaymentId: string | null;
  relatedThreadId: string | null;
  resolutionNotes: string | null;
  resolvedById: string | null;
}

export interface SupportContact {
  phone: string;
  telegramHandle: string;
  hours: string;
}

export interface PlatformHealth {
  openDisputes: number;
  overdueMatchApprovals: number;
  recordingComplianceEscalations: number;
  pendingPayoutBatches: number;
  generatedAt: string;
}

// H5 fix: backs the new GET /admin/reports/tutor-performance endpoint.
export interface TutorPerformanceRow {
  tutorId: string;
  fullName: string;
  verificationStatus: 'PENDING' | 'VERIFIED' | 'REJECTED';
  uniqueStudentsTaught: number;
  activeCohortCount: number;
  completedSessionCount: number;
  tutorCausedMissCount: number;
  badgeCount: number;
  complaintCount: number;
  createdAt: string;
}

export interface TutorPerformancePage {
  tutors: TutorPerformanceRow[];
  pagination: { page: number; limit: number; total: number; totalPages: number };
}
```

### 8.3 Hooks (src/hooks/useComplaints.ts)

```typescript
export function useFileComplaint() {
  return useMutation({
    mutationFn: (body: { category: ComplaintCategory; description: string; relatedCohortId?: string; relatedSessionId?: string; relatedPaymentId?: string }) =>
      api.post<ComplaintSummary>('/complaints', body).then((r) => r.data),
  });
}

export function useMyComplaints(status?: ComplaintStatus, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.MY_COMPLAINTS, status, page],
    queryFn: () => api.get<{ complaints: ComplaintSummary[]; page: number; limit: number; total: number }>(
      '/complaints/me', { params: { status, page, limit: 20 } }
    ).then((r) => r.data),
  });
}

export function useMyComplaintDetail(complaintId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.COMPLAINT_DETAIL, complaintId],
    queryFn: () => api.get<ComplaintDetail>(`/complaints/${complaintId}`).then((r) => r.data),
    enabled: !!complaintId,
  });
}

export function useSupportContact() {
  return useQuery({
    queryKey: [QUERY_KEYS.SUPPORT_CONTACT],
    queryFn: () => api.get<SupportContact>('/support/contact').then((r) => r.data),
    staleTime: Infinity, // static, Admin-configurable content — no need to refetch within a session
  });
}
```

### 8.4 Hooks (src/hooks/useAdminDisputes.ts, useAdminReporting.ts)

```typescript
export function useDisputeQueue(status?: ComplaintStatus, category?: ComplaintCategory, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.DISPUTE_QUEUE, status, category, page],
    queryFn: () => api.get('/admin/disputes', { params: { status, category, page, limit: 20 } }).then((r) => r.data),
    refetchInterval: 30_000,
  });
}

export function useReviewDispute(complaintId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.DISPUTE_DETAIL, complaintId],
    queryFn: () => api.get<AdminComplaintDetail>(`/admin/disputes/${complaintId}`).then((r) => r.data),
    enabled: !!complaintId,
  });
}

export function useResolveDispute() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ complaintId, ...body }: {
      complaintId: string;
      status: 'UNDER_REVIEW' | 'RESOLVED' | 'DISMISSED';
      resolutionAction?: ResolutionAction;
      resolutionNotes: string;
      affectedCohortMembershipId?: string; // H4 fix: replaces refundAmount — server computes the amount
    }) => api.patch(`/admin/disputes/${complaintId}`, body),
    onSuccess: (_d, vars) => {
      qc.invalidateQueries({ queryKey: [QUERY_KEYS.DISPUTE_QUEUE] });
      qc.invalidateQueries({ queryKey: [QUERY_KEYS.DISPUTE_DETAIL, vars.complaintId] });
    },
  });
}

export function usePlatformHealth() {
  return useQuery({
    queryKey: [QUERY_KEYS.PLATFORM_HEALTH],
    queryFn: () => api.get<PlatformHealth>('/admin/reports/platform-health').then((r) => r.data),
    refetchInterval: 60_000,
  });
}

export function useTutorPerformance(params: { page?: number; limit?: number; sortBy?: string; verificationStatus?: string }) {
  // H5 fix: new hook backing the previously-unbacked TutorPerformanceTable.
  return useQuery({
    queryKey: [QUERY_KEYS.TUTOR_PERFORMANCE, params],
    queryFn: () => api.get<TutorPerformancePage>('/admin/reports/tutor-performance', { params }).then((r) => r.data),
  });
}
```

### 8.5 Components

| File | Responsibility |
|---|---|
| `ComplaintForm.tsx` | Category select + description + optional related-entity picker (session/payment/cohort); mirrors the backend's business rule that a non-`OTHER` category requires at least one related entity, disabling submit until satisfied rather than only surfacing the 400 after the fact |
| `DisputeCard.tsx` (admin) | One dispute; shows linked context (`relatedThreadId` → link to `MessageThreadReviewPage`, 05-messaging-frontend.md §5.1; `relatedSessionId` → link to session detail, 04-class-delivery-library-frontend.md) and the resolution form (`status`, `resolutionAction`, `resolutionNotes`, conditionally `affectedCohortMembershipId` with a server-computed amount preview — H4 fix) |
| `AdminSidebar.tsx` | Existing per Doc 05b — admin-wide nav, used by `DashboardLayout` for the Admin role (already specified in `02-accounts-guardianship-frontend.md` §2.7; not re-specified here, only referenced since this feature adds nav entries to it: Disputes, Reports) |
| `PlatformStatsGrid.tsx` | Renders `usePlatformHealth`'s four counters as headline stat cards |
| `TutorPerformanceTable.tsx` | Tutor performance/badge history, sourced from `GET /admin/reports/tutor-performance` (H5 fix — endpoint added to `08-support-trust-admin-api.md`; this closes the gap previously flagged here) via `useTutorPerformance()`. Paginated table: name, verification status, `uniqueStudentsTaught`, active cohort count, completed sessions, tutor-caused misses, badge count, complaint count. |

### 8.6 Resolution Form Behavior

`DisputeCard`'s resolution form should conditionally require fields exactly as the API does (00-api-conventions §0.1's validation pattern, business rules beyond Zod shape called out per-endpoint): `resolutionAction` is required once `status` is set to `RESOLVED`; `affectedCohortMembershipId` is required only when `resolutionAction` is `REFUND_ISSUED` — **H4 fix:** this replaces a free-text `refundAmount` field; the form selects *which* membership's billing cycle to prorate, and the resulting amount is computed and previewed, never typed in. The form should disable the `TUTOR_SUSPENDED` and `REFUND_ISSUED` options with an inline note if the admin lacks the corresponding related-entity context (e.g. `TUTOR_SUSPENDED` without a resolvable tutor from `relatedSessionId`'s cohort) — this is a UX guard, not a substitute for the backend's own validation.

---

**This completes the 07-frontend-specification/ folder — all 9 files (0 + 01–08) are now written.**
