## Project: AKEWTutor — Frontend Specification
**Feature:** Payments & Earnings
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `07-payments-earnings-api.md`

**Depends on:** Matching & Cohorts, Class Delivery & Library (hard — a `TutorEarning` links to a `ScheduledSession`).

**Links back to:** [0. Frontend Conventions], [06-api/07-payments-earnings-api.md], [05b. Frontend Folder & File Structure §7]
**Links forward to:** [8-7. Frontend Function-Level Spec: Payments & Earnings]

---

### 7.1 Routes

```
/payments               → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → PaymentPage
/payments/history         → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → PaymentHistoryPage
/payments/paused            → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → PaymentPausedPage
/tutor/earnings               → ProtectedRoute(['TUTOR']) → DashboardLayout(TutorSidebar) → EarningsPage
/admin/pricing                  → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → PricingConfigPage
/admin/refunds                    → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → RefundReviewPage
/admin/payouts                      → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → PayoutManagementPage
/admin/promotions                     → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → PromotionManagementPage
```

### 7.2 Types (added to src/types/index.ts)

All monetary fields are `string` (Decimal-as-string), never `number` — matching `00-api-conventions.md` §0.6 exactly; the frontend must not `parseFloat()` these for display arithmetic without an explicit decimal-safe helper, to avoid the same floating-point risk the backend convention exists to prevent.

```typescript
export interface PaymentRecord {
  id: string;
  cohortId: string;
  amount: string;
  status: 'PENDING' | 'SUCCESS' | 'FAILED';
  chapaCheckoutUrl: string | null;
  createdAt: string;
}

export interface PaymentPauseStatus {
  isPaused: boolean;
  pausedSince: string | null;
  studentId: string;
}

export interface FormatPricing {
  format: 'ONE_TO_ONE' | 'ONE_TO_THREE' | 'ONE_TO_FIVE';
  pricePerStudentPerHour: string;
  totalPerHour: string;
  platformSharePerHour: string;
  tutorSharePerHour: string;
}

export interface RefundCase {
  id: string;
  paymentId: string;
  amount: string;
  reason: string;
  status: 'PENDING' | 'APPROVED' | 'REJECTED';
  approvedById: string | null;
  approvedAt: string | null;
  rejectedById: string | null;
  rejectedAt: string | null;
  rejectionReason: string | null;
  createdAt: string;
}
// I1 fix: this type was previously ahead of the backend — Doc 04's `Refund` entity now
// has a matching `status`/`rejectedBy*`/`rejectionReason` shape (06-api/07-payments-earnings-api.md),
// so `REJECTED` is a real, reachable state via `POST /admin/refunds/:refundId/reject` (useRejectRefund below).

export interface TutorEarningsSummary {
  totalEarnedThisMonth: string;
  upcomingPayoutAmount: string;
  reducedRateSessions: { sessionId: string; reason: string; rateApplied: string }[];
}

export interface PayoutBatch {
  id: string;
  tutorId: string;
  amount: string;
  status: 'PENDING' | 'PAID';
  periodStart: string;
  periodEnd: string;
}

export interface PromotionCode {
  id: string;
  code: string;
  discountPercent: number;
  isActive: boolean;
  expiresAt: string | null;
}
```

### 7.3 Hooks (src/hooks/usePayments.ts)

```typescript
export function useInitiatePayment() {
  return useMutation({
    mutationFn: (body: { cohortId: string; promotionCode?: string }) =>
      api.post<PaymentRecord>('/payments/initiate', body).then((r) => r.data),
  });
}

export function useMyPayments(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.PAYMENT_HISTORY, page],
    queryFn: () => api.get<{ payments: PaymentRecord[]; page: number; limit: number; total: number }>(
      '/payments/history', { params: { page, limit: 20 } }
    ).then((r) => r.data),
  });
}

export function usePauseStatus(studentId?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.PAUSE_STATUS, studentId],
    queryFn: () => api.get<PaymentPauseStatus>('/payment-pause/status', { params: { studentId } }).then((r) => r.data),
  });
}
```
`useInitiatePayment` returns `chapaCheckoutUrl` (implied by the endpoint's documented purpose in `00-api-conventions.md` §0.5 — "returns a Chapa checkout URL") — `PaymentPage` performs a full browser redirect (`window.location.href = data.chapaCheckoutUrl`) rather than an in-app iframe, since Chapa's own checkout flow is external and the webhook (`POST /payments/webhook/chapa`) is what actually confirms success server-side, not anything the client observes directly. `PaymentPage` should poll `useMyPayments` after redirect-back to reflect the confirmed status, rather than assuming success from the redirect alone.

### 7.4 Hooks (src/hooks/usePricing.ts, useEarnings.ts)

```typescript
export function useActivePricing() {
  return useQuery({
    queryKey: [QUERY_KEYS.PRICING],
    queryFn: () => api.get<{ pricing: FormatPricing[] }>('/pricing').then((r) => r.data),
  });
}

export function useUpdatePricing() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ format, ...body }: { format: string; pricePerStudentPerHour: string; platformSharePerHour: string; tutorSharePerHour: string }) =>
      api.put(`/admin/pricing/${format}`, body),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.PRICING] }),
  });
}

export function useMyEarnings() {
  return useQuery({
    queryKey: [QUERY_KEYS.EARNINGS],
    queryFn: () => api.get<TutorEarningsSummary>('/tutors/me/earnings').then((r) => r.data),
  });
}
```

### 7.5 Hooks (src/hooks/useRefunds.ts, usePayouts.ts, usePromotions.ts)

```typescript
export function useRefundQueue(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.REFUNDS, page],
    queryFn: () => api.get('/admin/refunds', { params: { page, limit: 20 } }).then((r) => r.data),
  });
}

export function useApproveRefund() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (refundId: string) => api.post(`/admin/refunds/${refundId}/approve`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.REFUNDS] }),
  });
}

// I1 fix: previously missing — REJECTED was a type-level state with no way to reach it.
export function useRejectRefund() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ refundId, rejectionReason }: { refundId: string; rejectionReason: string }) =>
      api.post(`/admin/refunds/${refundId}/reject`, { rejectionReason }),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.REFUNDS] }),
  });
}

export function usePayoutBatches(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.PAYOUTS, page],
    queryFn: () => api.get<{ payouts: PayoutBatch[] }>('/admin/payouts', { params: { page, limit: 20 } }).then((r) => r.data),
  });
}

export function useMarkPaid() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (payoutId: string) => api.post(`/admin/payouts/${payoutId}/mark-paid`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.PAYOUTS] }),
  });
}

export function useActivePromotions() {
  return useQuery({
    queryKey: [QUERY_KEYS.PROMOTIONS],
    queryFn: () => api.get<{ promotions: PromotionCode[] }>('/promotions/active').then((r) => r.data),
  });
}

export function useCreatePromotion() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { code: string; discountPercent: number; expiresAt?: string }) =>
      api.post<PromotionCode>('/admin/promotions', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.PROMOTIONS] }),
  });
}
```

### 7.6 Components

| File | Responsibility |
|---|---|
| `ChapaCheckoutButton.tsx` | Wraps `useInitiatePayment`; on success, performs the full-page redirect described in §7.3 |
| `PaymentHistoryTable.tsx` | Paginated transaction list |
| `PaymentReminderBanner.tsx` | 3-day countdown (`CountdownTimer`, foundation component), per-student anchored — a Parent with multiple children sees one banner per child with an outstanding payment, not one combined banner |
| `PricingConfigForm.tsx` (admin) | Per-format price/split editor; client-side validates `platformSharePerHour + tutorSharePerHour === totalPerHour` before submit as a UX guard, though the backend remains authoritative for this arithmetic |
| `RefundCard.tsx` (admin) | One refund case with its proration breakdown; Approve action wired to `useApproveRefund`, Reject action (with a required reason prompt) wired to `useRejectRefund` — **I1 fix** |
| `PayoutBatchTable.tsx` (admin) | Payout batches; itemizes any `reducedRateSessions` inline per tutor row rather than requiring a drill-in |
| `PromotionForm.tsx` (admin) | Create/edit a promo code |

### 7.7 Notes on the `PaymentPausedPage` Blocking Screen

`PaymentPausedPage` is a full blocking screen (not a banner) per Doc 05b's naming — it renders whenever `usePauseStatus().isPaused` is `true`, and should prevent navigation into class-delivery pages for the affected student rather than merely warning. Once payment resumes, `paymentPause.service.rescheduleSessionsDuringPause` (00-api-conventions §0.4) automatically reschedules any session that fell inside the pause window server-side — the frontend does not need its own reschedule call here; it should simply re-fetch `useUpcomingSessions` (04-class-delivery-library-frontend.md §4.3) after `isPaused` flips back to `false` to pick up the corrected schedule.

---

**Next:** proceed to → [08. Support, Trust & Admin Reporting Frontend]
