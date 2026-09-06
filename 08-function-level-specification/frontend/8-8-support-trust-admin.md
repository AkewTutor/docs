## Project: AKEWTutor — Frontend Function-Level Spec: Support, Trust & Admin Reporting
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `08-support-trust-admin-api.md`. **Frontend spec reference:** `08-support-trust-admin-frontend.md`.

**Depends on:** Shared Config, Messaging, Class Delivery & Library (hard); soft-integrates with Payments & Earnings (refund action) and Accounts & Guardianship (suspension action) — neither surfaces as a direct client-side call from this feature's own hooks, since both happen server-side inside `PATCH /admin/disputes/:complaintId` (frontend spec header).

---

### Shared Pattern: Simple Query Hook

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useMyComplaints | GET /complaints/me | [MY_COMPLAINTS, status, page] | — |
| useMyComplaintDetail | GET /complaints/:id | [COMPLAINT_DETAIL, complaintId] | `enabled: !!complaintId` |
| useSupportContact | GET /support/contact | [SUPPORT_CONTACT] | `staleTime: Infinity` — static, Admin-configurable content, no need to refetch within a session |
| useDisputeQueue (admin) | GET /admin/disputes | [DISPUTE_QUEUE, status, category, page] | `refetchInterval: 30_000` |
| useReviewDispute (admin) | GET /admin/disputes/:id | [DISPUTE_DETAIL, complaintId] | `enabled: !!complaintId` |
| usePlatformHealth (admin) | GET /admin/reports/platform-health | [PLATFORM_HEALTH] | `refetchInterval: 60_000` |

---

### src/hooks/useComplaints.ts — full block

#### useFileComplaint

| Field | Detail |
|---|---|
| Signature | `useFileComplaint(): UseMutationResult<ComplaintSummary, AxiosError, { category: ComplaintCategory; description: string; relatedCohortId?: string; relatedSessionId?: string; relatedPaymentId?: string }>` |
| Purpose | Wraps `POST /complaints`. |
| Side effects | No cache invalidation of a shared list here — `ComplaintForm`'s host page navigates to `/complaints` (the caller's own list) on success rather than patching a cached entry, since `useMyComplaints` will simply refetch on that page's next mount. |
| Edge cases | A `400` ("must reference a session, payment, or cohort unless OTHER") should in practice never reach the user, since `ComplaintForm` (below) disables submit until that rule is satisfied client-side — but the mutation's error handler still renders it as a fallback form-level error in case the client-side check and the backend's rule ever drift. |
| Test file | `tests/hooks/useComplaints.test.ts` |

---

### src/hooks/useAdminDisputes.ts — full block

#### useResolveDispute

| Field | Detail |
|---|---|
| Signature | `useResolveDispute(): UseMutationResult<AdminComplaintDetail, AxiosError, { complaintId: string; status: 'UNDER_REVIEW' \| 'RESOLVED' \| 'DISMISSED'; resolutionAction?: ResolutionAction; resolutionNotes: string; affectedCohortMembershipId?: string }>` — **H4 fix:** `refundAmount` replaced with `affectedCohortMembershipId`; the refund amount is always server-computed. |
| Purpose | Wraps `PATCH /admin/disputes/:complaintId`. |
| Side effects | `onSuccess` invalidates both `[DISPUTE_QUEUE]` and `[DISPUTE_DETAIL, complaintId]` — the queue list and the currently-open detail view both need to reflect the new status, since an Admin resolving a dispute is typically looking at both the detail panel and the queue behind it in the same screen. |
| Edge cases | This mutation's `resolutionAction: REFUND_ISSUED`/`TUTOR_SUSPENDED` values trigger server-side effects in Payments & Earnings / Accounts & Guardianship (API spec §8.2) that this hook has no visibility into beyond the response — it does **not** separately invalidate `[REFUNDS]`, `[PAYOUTS]`, `[ADMIN_PEOPLE]`, or `[TUTOR_PROFILE]`, since those features' own pages will simply refetch fresh whenever an Admin next navigates to them; wiring a cross-feature invalidation here would create a hidden coupling the API itself deliberately avoids exposing as a client-facing call (§8.2's "neither call is exposed as a separate client-facing endpoint"). |
| Test file | `tests/hooks/useAdminDisputes.test.ts` |

---

### src/pages/SubmitComplaintPage.tsx (new), src/components/ComplaintForm.tsx (new — full block)

| Field | Detail |
|---|---|
| Route | `/complaints` — `ProtectedRoute('any')` + `DashboardLayout` |
| `ComplaintForm` local state | `category: ComplaintCategory`, `description: string`, `relatedCohortId?/relatedSessionId?/relatedPaymentId?`. |
| Behavior | 1. Category select + description textarea (10–2000 chars, client-validated) + an optional related-entity picker (session/payment/cohort, populated from the caller's own `useUpcomingSessions`/`useMyPayments`/`useMyCohorts` lists so only entities they actually own are selectable — mirrors the backend's ownership check, §8.2's 403 case, as a UX guard rather than a substitute for it). 2. **Submit is disabled** unless either `category === 'OTHER'` or at least one related-entity field is set — mirrors the backend's business rule (§8.2's 400 case) exactly, so the error is prevented rather than only surfaced after a round trip (§8.5). 3. On submit, `useFileComplaint().mutate(body)`; on success, `navigate('/complaints')` (this same route, now rendering the list view — see below) with a confirmation toast. |
| Test file | `tests/components/ComplaintForm.test.tsx` |

`SubmitComplaintPage` itself, beyond hosting `ComplaintForm`, also renders `useMyComplaints(status, page)` below the form as the caller's own filed-complaint history (§8.1's routing note distinguishes this from `SupportContactPage` — one is a form that creates a record, the other is static contact info; this page is squarely the former, with its own history list underneath, not a separate route).

**States:** idle (form) · submitting · error · success (confirmation + history list)

### src/pages/SupportContactPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/support` — `ProtectedRoute('any')` + `DashboardLayout` |
| Behavior | `useSupportContact()` renders `phone`/`telegramHandle`/`hours` as static content. **Deliberately not a form** (§8.1) — there is no submit action anywhere on this page, since support contact is manual by design (Doc 01 §1.7 Assumption #4); this page must not be confused with or merged into `SubmitComplaintPage` even though both are reachable from similar nav positions. |

**States:** loading · success (no error/empty states meaningfully distinct from loading, since this is static content with no per-caller variation)

---

### src/pages/admin/DisputeQueuePage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/disputes` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Local state | `statusFilter`, `categoryFilter`, `page`, plus `selectedComplaintId: string \| null` for the detail panel. |
| Behavior | 1. `useDisputeQueue(statusFilter, categoryFilter, page)` renders one `DisputeCard` per row. 2. Selecting a row sets `selectedComplaintId`, which drives `useReviewDispute(selectedComplaintId)` for the detail panel — a two-pane list+detail layout, not a separate route per complaint (keeps the queue visible while reviewing one). |

**States:** loading · empty (`complaints: []` — queue clear, the normal steady state per §8.1, not an error) · success

### src/components/DisputeCard.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ complaint: AdminComplaintDetail; onResolve: (body) => void }` |
| Behavior | 1. Shows linked context: `relatedThreadId` renders a link to `MessageThreadReviewPage` (8-5 §5.1) if present; `relatedSessionId` renders a link to the session detail (8-4); `relatedPaymentId` similarly could link into payment history, though no dedicated admin single-payment detail route exists yet in this doc set beyond the paginated `PaymentHistoryTable` (8-7) — this card links to that list filtered/scrolled to the relevant entry rather than a nonexistent detail page. 2. Renders the resolution form: `status` (select), `resolutionAction` (select, only enabled once `status === 'RESOLVED'`), `resolutionNotes` (required textarea), `affectedCohortMembershipId` (only rendered when `resolutionAction === 'REFUND_ISSUED'` — **H4 fix:** a select of the candidate `CohortMembership`(s) derived from `relatedSessionId`/`relatedThreadId`'s Cohort, not a free-text amount; the actual refund amount is computed and shown as a read-only preview once a membership is selected, via the same proration the backend will apply, so Admin sees the number before confirming rather than typing one in). 3. **`TUTOR_SUSPENDED` and `REFUND_ISSUED` options are disabled with an inline note** if the admin lacks the corresponding related-entity context to resolve them meaningfully — e.g. `TUTOR_SUSPENDED` requires a resolvable tutor via `relatedSessionId`'s cohort; if `relatedSessionId` is null and `category !== 'TUTOR_CONDUCT'`-with-an-identifiable-tutor, the option is greyed out rather than silently allowed through to a backend 400 (§8.6 — this is a UX guard, not a substitute for the backend's own validation). 4. On submit, calls `onResolve({ status, resolutionAction, resolutionNotes, affectedCohortMembershipId })`, wired at the page level to `useResolveDispute`. |
| Test file | `tests/components/DisputeCard.test.tsx` |

### src/pages/admin/PlatformReportsPage.tsx (new), src/components/PlatformStatsGrid.tsx (new — light block)

| Field | Detail |
|---|---|
| Route | `/admin/reports` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| `PlatformStatsGrid` props | `{ health: PlatformHealth }` |
| Behavior | 1. Renders `openDisputes`, `overdueMatchApprovals`, `recordingComplianceEscalations`, `pendingPayoutBatches` as four headline stat cards, plus `generatedAt` as a small "last updated" timestamp — since `usePlatformHealth` polls every 60s (per its query definition above), this timestamp reassures the viewer the numbers are current rather than stale from a much earlier load. 2. **H5 fix:** below the stat grid, `TutorPerformanceTable` now renders from `useTutorPerformance({ page, limit, sortBy })` (page-level pagination/sort state, `useState`/URL params per Doc 07 §0.3) against the new `GET /admin/reports/tutor-performance` endpoint — this page previously omitted the table entirely pending that endpoint's addition; it no longer needs to. |

**States:** loading · success

---

### 8.6 Note on the UC-66/UC-87 Cross-Reference (M1 — resolved at the source)

Doc 03's UC-66 main flow originally referenced "UC-88 in Section Q" where **UC-87** (*Admin manages complaints and disputes*) was actually the matching use case — this has been corrected directly in `03-usecases.md`, not just noted downstream. This function-level spec's `DisputeQueuePage`/`DisputeCard` were already built against UC-87 as the correct target, so no behavior described above needed to change — this section exists only as a pointer for anyone tracing which frontend files assumed which UC number before the fix.

---

**This completes the `08-function-level-specification/frontend/` folder — all 8 feature files (8-1 through 8-8) are now written.**
