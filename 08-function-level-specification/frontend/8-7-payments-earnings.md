## Project: AKEWTutor — Frontend Function-Level Spec: Payments & Earnings
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `07-payments-earnings-api.md`. **Frontend spec reference:** `07-payments-earnings-frontend.md`.

**Depends on:** Matching & Cohorts, Class Delivery & Library (hard — a `TutorEarning` links to a `ScheduledSession`).

All monetary fields are `string` (Decimal-as-string) throughout this feature (§7.2) — no component or hook in this file ever calls `parseFloat()`/`Number()` on a money field for arithmetic; see `src/lib/money.ts` below for the one sanctioned exception (display-only, not computational).

---

### Shared Pattern: Simple Query Hook

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useMyPayments | GET /payments/history | [PAYMENT_HISTORY, page] | — |
| usePauseStatus | GET /payment-pause/status | [PAUSE_STATUS, studentId] | `studentId` optional (Parent) |
| useActivePricing | GET /pricing | [PRICING] | — |
| useMyEarnings | GET /tutors/me/earnings | [EARNINGS] | — |
| useRefundQueue (admin) | GET /admin/refunds | [REFUNDS, page] | — |
| usePayoutBatches (admin) | GET /admin/payouts | [PAYOUTS, page] | — |
| useActivePromotions | GET /promotions/active | [PROMOTIONS] | — |

### Shared Pattern: Simple Mutation + Invalidation

| Hook | Endpoint | Invalidates |
|---|---|---|
| useUpdatePricing (admin) | PUT /admin/pricing/:format | [PRICING] |
| useApproveRefund (admin) | POST /admin/refunds/:id/approve | [REFUNDS] |
| useMarkPaid (admin) | POST /admin/payouts/:id/mark-paid | [PAYOUTS] |
| useCreatePromotion (admin) | POST /admin/promotions | [PROMOTIONS] |

---

### src/lib/money.ts (new util — full block)

| Field | Detail |
|---|---|
| Signature | `formatMoney(value: string, currency = 'ETB'): string` |
| Purpose | The one sanctioned place a money-as-string field is converted for **display only** — every `PaymentRecord.amount`, `FormatPricing.*PerHour`, `RefundCase.amount`, etc. renders through this, never through an ad hoc `${value} ETB` template scattered per component. |
| Logic | 1. Parse `value` with a decimal-safe library call (not native `parseFloat`, to avoid the exact precision loss `00-api-conventions.md` §0.6 introduced the string convention to prevent) — e.g. `Decimal(value).toFixed(2)`. 2. Format with thousands separators and the currency suffix. |
| Edge cases | This function is display-only and its output is never fed back into a request body or a further arithmetic operation — any component needing to *compute* (e.g. verify `platformSharePerHour + tutorSharePerHour === totalPerHour` in `PricingConfigForm`, below) does so via the same decimal-safe library's arithmetic methods, not by parsing this function's formatted string back into a number. |
| Test file | `tests/lib/money.test.ts` |

---

### src/hooks/usePayments.ts — full block

#### useInitiatePayment

| Field | Detail |
|---|---|
| Signature | `useInitiatePayment(): UseMutationResult<PaymentRecord, AxiosError, { cohortId: string; promotionCode?: string }>` |
| Purpose | Wraps `POST /payments/initiate`. |
| Side effects | On success, the calling component (`ChapaCheckoutButton`) performs a full-page redirect: `window.location.href = data.chapaCheckoutUrl` — not an in-app iframe, since Chapa's checkout is external and the webhook (`POST /payments/webhook/chapa`) confirms success server-side, not anything the client observes directly (§7.3). No query invalidation happens here, since there is nothing yet to invalidate — the payment isn't confirmed until the redirect-back. |
| Edge cases | An invalid/expired `promotionCode` is a `400`-class mutation error, surfaced inline near the code input on `PaymentPage`, not as a full-page error — the rest of the checkout flow (cohort selection, price display) remains usable so the person can retry without the code. |
| Test file | `tests/hooks/usePayments.test.ts` |

---

### src/pages/PaymentPage.tsx (new), src/components/ChapaCheckoutButton.tsx (new — full block)

| Field | Detail |
|---|---|
| Route | `/payments` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| `ChapaCheckoutButton` props | `{ cohortId: string; promotionCode?: string }` |
| Behavior (button) | On click, `useInitiatePayment().mutate({ cohortId, promotionCode })`; on success, redirects per the hook's side-effect note above; on error, surfaces the mutation's error message inline (does not redirect). |
| Behavior (page) | 1. Reads the outstanding cohort(s) awaiting payment from `useMyCohorts` (8-3, filtering `membershipStatus === 'PENDING_PAYMENT'`). 2. Renders `useActivePricing()`'s relevant format's price via `formatMoney`. 3. Optional promo-code field, passed through to `ChapaCheckoutButton`. 4. On return navigation from Chapa (the page re-mounts or regains focus), calls `useMyPayments()` (below) to reflect the now-confirmed status — the page does **not** assume success purely because the browser navigated back (§7.3); it polls/refetches and renders whatever `PaymentRecord.status` actually is. |

**States:** loading (pricing/cohort data) · idle (checkout ready) · redirecting (button clicked, before navigation away) · success (post-return, confirmed via refetch) · error (initiate failed, or promo code rejected)

### src/pages/PaymentHistoryPage.tsx (new), src/components/PaymentHistoryTable.tsx (new — light block)

| Field | Detail |
|---|---|
| Route | `/payments/history` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Behavior | `useMyPayments(page)` renders `PaymentHistoryTable` — a paginated list, each row's `amount` via `formatMoney`, `status` via `StatusBadge`. |

**States:** loading · empty (`payments: []` — no history yet) · success

### src/pages/PaymentPausedPage.tsx (new — full block, per frontend spec §7.7)

| Field | Detail |
|---|---|
| Route | `/payments/paused` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Behavior | 1. `usePauseStatus(studentId)`. 2. If `isPaused: true`, renders as a **full blocking screen** (not a dismissible banner) — this page's guard logic additionally prevents navigation into any class-delivery route for the affected student while paused, implemented as a redirect-to-here check inside `ProtectedRoute` or a route-level effect for `/student/upcoming-classes` and `/tutor/conduct-class/:sessionId` when the relevant student's pause is active. 3. Once `isPaused` flips to `false` (observed on next mount/refetch — this page does not itself poll continuously, since resuming is a discrete, infrequent event the person will naturally revisit this page to check), the block lifts and `useUpcomingSessions` (8-4) is refetched fresh to pick up whatever `paymentPause.service.rescheduleSessionsDuringPause` already corrected server-side (§0.4) — the frontend issues no reschedule call of its own. |
| Edge cases | A Parent with multiple children sees this block scoped to the specific paused child, not all children — `studentId` must be threaded through consistently from wherever the block was triggered. |
| Test file | `tests/pages/PaymentPausedPage.test.tsx` |

### src/components/PaymentReminderBanner.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ studentId: string; dueDate: string }` |
| Behavior | Renders a 3-day `CountdownTimer` (foundation component) toward `dueDate`, anchored per-student — a Parent with multiple children with outstanding payments renders one instance of this component per child, mapped from a list, never a single combined banner (§7.6). |

---

### src/pages/tutor/EarningsPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/tutor/earnings` — `ProtectedRoute(['TUTOR'])` + `DashboardLayout(TutorSidebar)` |
| Behavior | `useMyEarnings()` renders `totalEarnedThisMonth`/`upcomingPayoutAmount` via `formatMoney`, plus a `reducedRateSessions` list (session + reason + rate) shown inline, not requiring a drill-in — same "itemize inline, don't force a click-through" convention as `PayoutBatchTable` below. |

**States:** loading · success

---

### src/pages/admin/PricingConfigPage.tsx (new), src/components/PricingConfigForm.tsx (new — full block)

| Field | Detail |
|---|---|
| Route | `/admin/pricing` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| `PricingConfigForm` props | `{ format: CohortFormat; current: FormatPricing; onSave: (body) => void }` |
| Local state | `pricePerStudentPerHour`, `platformSharePerHour`, `tutorSharePerHour`, all as raw string inputs (not native number inputs, to avoid a browser locale silently reformatting a decimal string the backend expects verbatim). |
| Behavior | 1. As any of the three fields change, computes (via the decimal-safe library, not native arithmetic) whether `platformSharePerHour + tutorSharePerHour === totalPerHour`(derived as `pricePerStudentPerHour * targetGroupSize`, per the format's known group size) and disables "Save" with an inline note if not — a client-side UX guard only; the backend remains authoritative for this arithmetic and could still reject a value this check passed if its own rule is stricter (§7.6). 2. On submit, calls `onSave({ pricePerStudentPerHour, platformSharePerHour, tutorSharePerHour })`, wired at the page level to `useUpdatePricing().mutate({ format, ...body })`. |
| Test file | `tests/components/PricingConfigForm.test.tsx` |

**States (page):** loading (current pricing) · success (one form per format, via `useActivePricing()`)

### src/pages/admin/RefundReviewPage.tsx (new), src/components/RefundCard.tsx (new — light block)

| Field | Detail |
|---|---|
| Route | `/admin/refunds` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| `RefundCard` props | `{ refund: RefundCase; onApprove: () => void }` |
| Behavior | Renders `amount` (via `formatMoney`) + `reason` + a proration breakdown (whatever fields the API's refund-detail response includes, rendered as a simple list, not recomputed client-side); "Approve" wired to `useApproveRefund`. |

**States (page):** loading · empty (`refunds: []` — queue clear) · success

### src/pages/admin/PayoutManagementPage.tsx (new), src/components/PayoutBatchTable.tsx (new — light block)

| Field | Detail |
|---|---|
| Route | `/admin/payouts` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | `usePayoutBatches(page)` renders `PayoutBatchTable`, each row's "Mark paid" wired to `useMarkPaid`, disabled once `status === 'PAID'` (idempotency guard at the UI level — the backend's own idempotency, if any, is not assumed, but there is no reason to let an already-paid row be clicked again). Per-tutor `reducedRateSessions` (if present on this response too — cross-referenced from `useMyEarnings`'s shape, since `PayoutBatch` itself is per Doc 07 §7.2 a coarser record) are itemized inline where available, not requiring drill-in. |

**States:** loading · empty (no pending batches) · success

### src/pages/admin/PromotionManagementPage.tsx (new), src/components/PromotionForm.tsx (new — light block)

| Field | Detail |
|---|---|
| Route | `/admin/promotions` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | 1. `useActivePromotions()` lists current codes. 2. `PromotionForm` (`code`, `discountPercent`, optional `expiresAt`) submits via `useCreatePromotion`; `discountPercent` client-clamped to a sane 1–100 range before submit, since a value outside that range is unambiguously invalid regardless of any backend-specific business rule. |

**States:** loading · success (list + create form)

---

**Next:** proceed to → [8-8. Frontend: Support, Trust & Admin Reporting]
