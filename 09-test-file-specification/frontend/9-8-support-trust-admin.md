## Project: AKEWTutor — Frontend Test Documentation: Support, Trust & Admin Reporting
**Links back to:** [05b §8], [07-08], [8-8. Frontend Function-Level Spec: Support, Trust & Admin Reporting]
**Conventions:** see `9-1-shared-config.md` for shared setup/mock patterns and OWASP category definitions.

**Depends on:** Shared Config, Messaging, Class Delivery & Library (hard); soft-integrates with Payments & Earnings and Accounts & Guardianship (server-side-only effects, no direct client-side call — see §9.4).

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| hooks/useComplaints.ts (useFileComplaint) | FR-AD-017, complaint intake shape | NFR-009 |
| hooks/useAdminDisputes.ts (useResolveDispute) | FR-AD-012, FR-AD-017, FR-MK-003, FR-SP-048 (H4 fix) | NFR-009 |
| hooks/useAdminReporting.ts (usePlatformHealth, useTutorPerformance — H5 fix) | FR-AD-021, FR-AD-022, FR-AD-005, FR-AD-014, FR-AD-020 | — |
| components/ComplaintForm.tsx | FR-AD-017 (400 business rule mirrored client-side) | — |
| components/DisputeCard.tsx | FR-AD-012, FR-AD-017 (H4 fix) | NFR-009 |
| components/PlatformStatsGrid.tsx | FR-AD-021–022 | — |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Notes |
|---|---|---|---|
| src/hooks/useComplaints.ts | tests/hooks/useComplaints.test.ts | Hook | mandatory — full block below |
| src/hooks/useAdminDisputes.ts | tests/hooks/useAdminDisputes.test.ts | Hook | mandatory — full block below |
| src/hooks/useAdminReporting.ts | tests/hooks/useAdminReporting.test.ts | Hook | mandatory |
| src/pages/SubmitComplaintPage.tsx, components/ComplaintForm.tsx | tests/components/ComplaintForm.test.tsx | Component | non-trivial: business-rule-mirroring submit gate — full block below |
| src/pages/SupportContactPage.tsx | tests/pages/SupportContactPage.test.tsx | Component | non-trivial: deliberately not a form — negative-assertion test |
| src/pages/admin/DisputeQueuePage.tsx | tests/pages/DisputeQueuePage.test.tsx | Component | non-trivial: two-pane list+detail state coordination |
| src/components/DisputeCard.tsx | tests/components/DisputeCard.test.tsx | Component | non-trivial: H4 fix (computed refund preview) + conditional option-disabling — full block below |
| src/pages/admin/PlatformReportsPage.tsx, components/PlatformStatsGrid.tsx | tests/pages/PlatformReportsPage.test.tsx | Component | non-trivial: freshness timestamp, H5 fix (tutor-performance table) |

---

### 9.2 Test Case Detail — useComplaints.test.ts, useAdminDisputes.test.ts (full blocks)

**OWASP: A01:2021 – Broken Access Control (ownership-scoped related-entity pickers), A04:2021 – Insecure Design.**

#### useFileComplaint

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No cache invalidation on success; navigation is the caller's job | mock success | `mutate(body)` | no query key invalidated by the hook itself — `ComplaintForm`'s host page navigates to `/complaints`, which will refetch `useMyComplaints` fresh on mount |
| A `400` ("must reference a session/payment/cohort unless OTHER") is still handled as a fallback | mock `400` | `mutate(body)` | error propagates and is rendered as a form-level fallback error — 8-8 is explicit this should in practice never reach the user since `ComplaintForm` disables submit first, but the hook/error-path is still tested in case the client-side check and the backend's rule ever drift apart |

#### useResolveDispute

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success invalidates both `[DISPUTE_QUEUE]` and `[DISPUTE_DETAIL, complaintId]` | mock success | `mutate({ complaintId, status, ... })` | both keys invalidated |
| **Never separately invalidates `[REFUNDS]`, `[PAYOUTS]`, `[ADMIN_PEOPLE]`, or `[TUTOR_PROFILE]`** | mock a resolution with `resolutionAction: 'REFUND_ISSUED'`, then separately `'TUTOR_SUSPENDED'` | `mutate(...)` for each | none of those four cross-feature query keys are invalidated by this hook in either case — 8-8 is explicit this would create a hidden coupling the API itself deliberately avoids exposing as a client-facing call; a test asserting these are invalidated would actually be testing for a design the spec deliberately rejects, so this is a real negative assertion worth keeping visible in code review |
| `affectedCohortMembershipId` replaces a free-text `refundAmount` (H4 fix) | inspect the mutation's input type / call args | `mutate({ ..., affectedCohortMembershipId: 'x' })` | no `refundAmount` field is ever part of the payload this hook sends — confirms the H4 fix (amount is always server-computed) is reflected in the actual call shape, not just the type signature in the docs |

---

### 9.3 Test Case Detail — ComplaintForm.test.tsx (full block)

FRs: FR-AD-017. **OWASP: A01:2021 – Broken Access Control (related-entity picker scoped to the caller's own resources).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Submit disabled unless `category === 'OTHER'` OR a related-entity field is set | select a non-OTHER category, leave all related fields empty | — | submit disabled — mirrors the backend's 400 rule pre-emptively, per 8-8 |
| Selecting `OTHER` alone enables submit with no related entity required | select `OTHER`, leave related fields empty | — | submit enabled |
| Setting any one related field (without `OTHER`) enables submit | select a non-OTHER category, set `relatedSessionId` | — | submit enabled |
| Related-entity pickers are populated only from the caller's own resources | mock `useUpcomingSessions`/`useMyPayments`/`useMyCohorts` for the current caller | open the session/payment/cohort picker | only entries from those hooks' results appear — there is no free-text id entry field anywhere that could let a caller reference another person's session/payment/cohort id; this mirrors the backend's ownership check (§8.2's 403 case) as a UX guard, explicitly not a substitute for it (8-8), but still worth verifying the picker itself can't be used to attempt referencing an arbitrary id |
| Description length validated (10–2000 chars) | attempt 5 chars, then 2001 chars | — | both rejected client-side |
| Success navigates to `/complaints` with a confirmation toast | mock success | submit | `navigate('/complaints')`, toast shown |

---

### 9.4 Test Case Detail — SupportContactPage.test.tsx

**Included as a deliberate negative-assertion test, not a gap.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders static content, no submit action anywhere on the page | render | inspect the full DOM | zero `<form>`/submit-button elements exist — 8-8 is explicit this page must never be confused with or merged into `SubmitComplaintPage`; a future contributor accidentally adding a "quick complaint" mini-form here would be caught by this negative assertion |
| `phone`/`telegramHandle`/`hours` render from `useSupportContact()` | mock the hook's response | render | all three fields visible, matching the response exactly |

---

### 9.5 Test Case Detail — DisputeQueuePage.test.tsx, DisputeCard.test.tsx (full block)

FRs: FR-AD-012, FR-AD-017 (H4 fix). **OWASP: A01:2021 – Broken Access Control (Admin-only), A04:2021 – Insecure Design (option-disabling UX guard, not a substitute for backend validation).**

#### DisputeQueuePage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Selecting a row drives the detail panel without navigating to a new route | click a row in the queue | — | `selectedComplaintId` set, `useReviewDispute(selectedComplaintId)` fires, queue remains visible alongside the detail panel — confirms the two-pane design (8-8), not a route-per-complaint |
| Filters re-query | change `statusFilter`/`categoryFilter` | — | `useDisputeQueue` re-invoked with the new filters |
| Empty queue is the normal steady state | mock `{ complaints: [] }` | render | no error, calm empty state |

#### DisputeCard (full block)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders context links only when the corresponding id is present | render with `relatedThreadId` set but `relatedSessionId` null | — | thread link shown, session link absent |
| `resolutionAction` select disabled until `status === 'RESOLVED'` | render with `status: 'UNDER_REVIEW'` | — | `resolutionAction` control disabled; changing `status` to `'RESOLVED'` enables it |
| **H4 fix: `affectedCohortMembershipId` renders as a select of candidate memberships with a computed, read-only preview amount — never a free-text amount field** | select `resolutionAction: 'REFUND_ISSUED'`, then select a candidate membership | — | a select control (not a text input) is used for the membership; once selected, a **read-only** preview of the refund amount renders, computed via the same proration the backend will apply — there is no editable numeric input anywhere in this flow that could let an Admin type an arbitrary refund figure, which is precisely the vulnerability the H4 fix closed (replacing a free-text `refundAmount` field); this is the single highest-priority test in this feature file |
| `TUTOR_SUSPENDED`/`REFUND_ISSUED` options disabled without resolvable context | render with `relatedSessionId: null` and a category that isn't `TUTOR_CONDUCT`-with-an-identifiable-tutor | — | those two options are greyed out with an inline note, not silently selectable through to a backend 400 — explicitly a UX guard, not a substitute for backend validation (8-8) |
| `onResolve` payload never includes a raw refund amount | complete a `REFUND_ISSUED` resolution and submit | — | the call to `onResolve` contains `affectedCohortMembershipId`, not any amount field — confirms the H4 fix end-to-end through to the actual submitted payload, not just the UI's visual shape |

---

### 9.6 Test Case Detail — PlatformReportsPage.test.tsx (PlatformStatsGrid, H5 fix)

FRs: FR-AD-021, FR-AD-022. **OWASP: A01:2021 – Broken Access Control (Admin-only).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders all four headline stats plus `generatedAt` | mock `usePlatformHealth` | render | `openDisputes`, `overdueMatchApprovals`, `recordingComplianceEscalations`, `pendingPayoutBatches`, and a "last updated" timestamp all present |
| Polls at 60s | fake timers | advance 60_000ms | `usePlatformHealth` re-fires; `generatedAt` reflects the refreshed value on the next successful poll |
| **H5 fix: `TutorPerformanceTable` renders from `useTutorPerformance`, previously omitted entirely** | mock `useTutorPerformance({ page, limit, sortBy })` | render | the table renders below the stat grid — this test exists specifically to confirm the H5 gap (the page previously shipped without this table pending the endpoint's addition) is actually closed in the current build, not still silently missing |
| Sort/pagination state changes re-invoke the hook with new params | change sort column, change page | — | `useTutorPerformance` re-invoked with updated `sortBy`/`page` |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `DisputeCard`'s H4-fix test asserts there is **no** numeric text input anywhere in the `REFUND_ISSUED` flow — not just that a select exists alongside one, since a form with both a select and a leftover free-text amount field would still technically "have a select" while remaining vulnerable to an arbitrary typed figure.
- [ ] `useResolveDispute`'s negative-invalidation test explicitly lists all four query keys it must **not** touch (`REFUNDS`, `PAYOUTS`, `ADMIN_PEOPLE`, `TUTOR_PROFILE`) rather than a generic "only two keys invalidated" count assertion, which could pass even if the wrong key were substituted in.
- [ ] `SupportContactPage`'s no-form assertion queries for the complete absence of `<form>`/submit-role elements, not merely that a specific "Submit Complaint" button is absent — a differently-labeled sneaky form would still be caught.
- [ ] `PlatformReportsPage`'s H5-fix test would actually fail on the pre-fix version of the page (i.e., it's a real regression test against a documented prior gap, not a test that happens to pass regardless of whether the table exists).
- [ ] `ComplaintForm`'s related-entity-picker test confirms the picker's option list, not just that *a* picker renders — a picker populated from the wrong hook (e.g. an admin-wide list instead of the caller's own) would still "render a picker" while failing the actual ownership-scoping intent.

---

### 9.8 Out of Scope for Automated Testing (and why)

- **Server-side effects of `resolutionAction: REFUND_ISSUED`/`TUTOR_SUSPENDED`** (the actual refund/suspension logic) — these happen entirely inside `PATCH /admin/disputes/:complaintId` server-side (API spec §8.2); this doc only tests that the frontend sends the correct request shape and never invents its own cross-feature side effects, per §9.2's negative-invalidation case.
- **The proration formula's own numeric correctness** (used for `DisputeCard`'s read-only refund preview) — the frontend renders whatever the backend computes; the formula itself is covered by the backend's `9-7-payments-earnings.md` (Section 13 proration formula).
- **Platform-health metric calculation correctness** (`openDisputes`, `overdueMatchApprovals`, etc.) — server-side aggregation, covered by the backend's `9-8-support-trust-admin.md`; this doc verifies rendering and polling only.

---

**This completes the `09-test-file-specification/frontend/` folder — all 8 feature files (9-1 through 9-8) are now written, mirroring the backend's 9-1 through 9-8.**
