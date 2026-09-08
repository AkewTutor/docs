## Project: AKEWTutor — Frontend Test Documentation: Payments & Earnings
**Links back to:** [05b §7], [07-07], [8-7. Frontend Function-Level Spec: Payments & Earnings]
**Conventions:** see `9-1-shared-config.md` for shared setup/mock patterns and OWASP category definitions.

**Depends on:** Matching & Cohorts, Class Delivery & Library (hard).

All monetary fields are `string` (Decimal-as-string) throughout, per 8-7. Every test in this doc that touches a money field asserts against the string value directly or via the decimal-safe library's own comparison methods — **never** via `parseFloat`/`Number()` coercion, since a test written that way could pass while hiding the exact precision-loss bug the string convention exists to prevent.

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| lib/money.ts (formatMoney) | (display-only utility, no FR of its own) | NFR-007 (financial data integrity in transit/display) |
| hooks/usePayments.ts (useInitiatePayment) | FR-PB-001–004, FR-PB-008–009 | NFR-007 |
| hooks/usePauseStatus, useActivePricing, useMyEarnings, useRefundQueue, usePayoutBatches, useActivePromotions | FR-PB-005–007, FR-AD-009–012, FR-AD-016, FR-TU-019 | — |
| components/ChapaCheckoutButton.tsx, pages/PaymentPage.tsx | FR-PB-001–004 | NFR-007 |
| pages/PaymentPausedPage.tsx | FR-PB-005 | NFR-009 |
| components/PricingConfigForm.tsx | FR-AD-009 | — |
| components/RefundCard.tsx, pages/admin/RefundReviewPage.tsx | FR-PB-007, FR-AD-012 | — |
| components/PayoutBatchTable.tsx | FR-TU-019, FR-AD-011 | — |
| components/PromotionForm.tsx | FR-AD-016 | — |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Notes |
|---|---|---|---|
| src/lib/money.ts | tests/lib/money.test.ts | Unit | mandatory (util, per 8-7) — full block below |
| src/hooks/usePayments.ts | tests/hooks/usePayments.test.ts | Hook | mandatory — full block below |
| src/hooks/useEarnings.ts, usePricing.ts, useRefunds.ts, usePayouts.ts, usePromotions.ts | tests/hooks/{useEarnings,usePricing,useRefunds,usePayouts,usePromotions}.test.ts | Hook | mandatory (plain reads/writes per the shared-pattern tables) |
| src/pages/PaymentPage.tsx, components/ChapaCheckoutButton.tsx | tests/pages/PaymentPage.test.tsx | Component | non-trivial: external redirect, post-return reconciliation — full block below |
| src/pages/PaymentHistoryPage.tsx | tests/pages/PaymentHistoryPage.test.tsx | Component | thin table wrapper; `formatMoney` usage checked, no separate full block |
| src/pages/PaymentPausedPage.tsx | tests/pages/PaymentPausedPage.test.tsx | Component | non-trivial: full blocking screen, per-student scoping — full block below |
| src/components/PaymentReminderBanner.tsx | tests/components/PaymentReminderBanner.test.tsx | Component | non-trivial: one-per-child instancing |
| src/pages/tutor/EarningsPage.tsx | tests/pages/EarningsPage.test.tsx | Component | non-trivial: inline itemization convention |
| src/components/PricingConfigForm.tsx | tests/components/PricingConfigForm.test.tsx | Component | non-trivial: decimal-safe arithmetic guard — full block below |
| src/pages/admin/PricingConfigPage.tsx | — | — | Not required — thin route/layout wrapper around `PricingConfigForm` (`useActivePricing()` fetch + per-format form composition, no branching logic of its own); covered by `PricingConfigForm.test.tsx` plus a smoke test confirming one form renders per format, matching the `AvailabilityPage.tsx`/`SubjectRankingPage.tsx` precedent in `9-2-accounts-guardianship.md §9.1` |
| src/components/RefundCard.tsx | tests/components/RefundCard.test.tsx | Component | non-trivial: no client-side recomputation of proration |
| src/pages/admin/RefundReviewPage.tsx | — | — | Not required — thin route/layout wrapper around `RefundCard`, no logic beyond the loading/empty/success states listed in 8-7; covered by `RefundCard.test.tsx` plus a smoke test for the empty-queue state, matching the `AvailabilityPage.tsx`/`SubjectRankingPage.tsx` precedent |
| src/components/PayoutBatchTable.tsx | tests/components/PayoutBatchTable.test.tsx | Component | non-trivial: idempotency UI guard |
| src/pages/admin/PayoutManagementPage.tsx | — | — | Not required — thin route/layout wrapper around `PayoutBatchTable` (`usePayoutBatches(page)` fetch only); covered by `PayoutBatchTable.test.tsx` plus a smoke test for the empty-batches state, matching the `AvailabilityPage.tsx`/`SubjectRankingPage.tsx` precedent |
| src/components/PromotionForm.tsx | tests/components/PromotionForm.test.tsx | Component | non-trivial: percent-range clamp |
| src/pages/admin/PromotionManagementPage.tsx | — | — | Not required — thin route/layout wrapper around `PromotionForm` + `useActivePromotions()` list, no logic of its own; covered by `PromotionForm.test.tsx` plus a smoke test for the code-list render, matching the `AvailabilityPage.tsx`/`SubjectRankingPage.tsx` precedent |

---

### 9.2 Test Case Detail — money.test.ts (full block)

**OWASP: A04:2021 – Insecure Design (financial precision failures are a data-integrity risk, treated with the same rigor as a security defect on a platform handling real payments).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Formats a decimal string with thousands separators and currency suffix | `formatMoney("1234.5")` | — | e.g. `"1,234.50 ETB"` (exact format per the design, but the separator/decimal-places/suffix presence is asserted regardless of exact copy) |
| Never uses native `parseFloat`/`Number()` internally on the value | inspect the implementation (static check as part of this test file's setup, or a spy on `Number`/`parseFloat` if feasible) | call `formatMoney` | the decimal-safe library's own parsing path is used — this is the exact precision-loss vector `00-api-conventions.md` §0.6 introduced the string convention to prevent, and a native-number code path anywhere in this function silently reintroduces float rounding error on real currency values |
| A value with more than 2 decimal places is not silently truncated in a way that loses cents | `formatMoney("99.999")` | — | rounds per the decimal-safe library's documented rounding mode, not a native-float truncation artifact (e.g. never renders `"99.99" ` from a value that should round to `"100.00"`) |
| **[Phase 4 — Review §6.1] Round-half-up applied at exactly the `.XX5` boundary** | `formatMoney("33.125")` | — | renders `"33.13"`, not `"33.12"` — confirms the same round-half-up discipline `9-7-payments-earnings.md §9.12`'s `calculateProration` enforces server-side is not undermined by a different (e.g. round-half-to-even) rounding mode on the display side; a value server-computed as `"33.13"` and then re-rounded client-side must never visibly disagree with what the server actually charged/refunded |
| Output is display-only — never fed back into a request | grep-style/static assertion in the test file's own documentation comment (not a runtime check) | — | no component in this feature file's other tests ever passes a `formatMoney(...)` return value into a mutation payload — cross-referenced against §9.3's `PricingConfigForm` case, which explicitly uses the raw decimal-safe arithmetic instead |

---

### 9.3 Test Case Detail — usePayments.test.ts (full block: useInitiatePayment)

FRs: FR-PB-001–004. **OWASP: A08:2021 – Software and Data Integrity Failures (payment confirmation must come from the server, never assumed from client navigation).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success triggers a full-page redirect, not an in-app iframe | mock success → `{ chapaCheckoutUrl }` | `mutate({ cohortId, promotionCode })`, observe the calling component's side effect | `window.location.href` is set to `chapaCheckoutUrl` — asserted via a mocked `window.location`, not merely that the mutation resolved |
| No query invalidation on success | mock success | `mutate(...)` | no query key invalidated by the hook — 8-7 is explicit there is nothing yet to invalidate, since the payment isn't confirmed until the webhook-driven redirect-back |
| Invalid/expired promo code is a `400`-class error surfaced inline, not a full-page error | mock `400` for a bad `promotionCode` | `mutate(...)` | error is scoped to the promo-code UI (asserted at the `PaymentPage` level in §9.4), and critically, the rest of checkout (cohort selection, price display) remains interactive after this error — a full-page error boundary catching this would incorrectly block retry-without-code |

---

### 9.4 Test Case Detail — PaymentPage.test.tsx, PaymentPausedPage.test.tsx (full blocks)

**OWASP: A08:2021 (payment status must never be assumed client-side), A01:2021 – Broken Access Control (pause block scoped per-student).**

#### PaymentPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Does **not** assume payment success purely from returning to the page | simulate the page regaining focus/re-mounting after a redirect to Chapa and back (no query params indicating outcome) | — | the page calls/refetches `useMyPayments()` and renders whatever `PaymentRecord.status` the refetch actually returns — it does not render a hardcoded "Payment successful" state purely because the browser navigated back, per 8-7's explicit warning against exactly this assumption |
| Promo-code error keeps the rest of the flow usable | mock `useInitiatePayment` to reject on a promo code | submit with a code | cohort selection and price display remain interactive; only the promo-code field/message shows an error |
| Price shown via `formatMoney`, never a raw string template | render with pricing data | — | the rendered price string matches `formatMoney`'s output exactly, confirmed by spying on `formatMoney` and asserting it was called with the pricing value, not that a component-local `${value} ETB` template was used instead |

#### PaymentPausedPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `isPaused: true` renders as a full blocking screen, not a dismissible banner | mock `usePauseStatus` → `{ isPaused: true }` | render | no dismiss/close affordance exists on this screen at all |
| Navigation into class-delivery routes is blocked for the affected student while paused | mock a paused `studentId`, attempt to render/navigate to `/student/upcoming-classes` for that same student | — | redirected to `/payments/paused` (via the route-level guard/effect noted in 8-7) rather than rendering the class list — this is the security/business-integrity-relevant case: a payment-paused student must not be able to reach class-delivery pages through a direct URL, not just have the nav link hidden |
| Scoped to the specific paused child, not all children (Parent with multiple children) | mock a Parent with 2 children, only one paused | render for each child's context | only the paused child's context shows the block; the other child's pages remain fully accessible |
| Once `isPaused` flips to `false` on next mount/refetch, the block lifts and `useUpcomingSessions` refetches fresh | mock a transition from `true` to `false` across a remount | — | block clears; sessions refetch — the frontend issues no reschedule call of its own, since `paymentPause.service.rescheduleSessionsDuringPause` already corrected server-side (§0.4) |

---

### 9.5 Test Case Detail — PricingConfigForm.test.tsx (full block)

FRs: FR-AD-009. **OWASP: A04:2021 – Insecure Design (client-side arithmetic guard is a UX convenience, backend remains authoritative — must not be conflated with real validation).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Fields are raw string inputs, not native number inputs | render the form | — | inputs are `type="text"` (or an equivalent non-numeric input), confirmed to avoid a browser-locale silently reformatting a decimal string (e.g. `,` vs `.` as the decimal separator) the backend expects verbatim — this is a real, easy-to-regress detail 8-7 calls out explicitly |
| "Save" disabled when `platformSharePerHour + tutorSharePerHour !== totalPerHour` | enter values that don't sum correctly for the format's known group size | — | Save disabled, inline note shown; the sum check uses the decimal-safe library's arithmetic, not native `+` on parsed floats — asserted the same way as `money.test.ts`'s no-native-arithmetic case |
| Enabled once the values do sum correctly | correct the values | — | Save enabled |
| This is a UX guard only — the backend's own stricter rule, if any, remains authoritative | pass values that pass this component's check | submit, mock the backend rejecting anyway with a `400` | the form surfaces that server-side rejection as a real error rather than assuming its own pre-check guarantees backend acceptance — a test suite that only checked the client-side gate and never exercised a server-side rejection-despite-passing-client-check would be over-trusting a UX convenience as if it were a security/validation boundary |

---

### 9.6 Test Case Detail — RefundCard.test.tsx, PayoutBatchTable.test.tsx, PromotionForm.test.tsx

**OWASP: A01:2021 – Broken Access Control (Admin-only), A08:2021 (no client-side recomputation of authoritative financial figures).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| RefundCard renders `amount`/proration breakdown exactly as returned, never recomputed client-side | render with a refund detail response | — | rendered figures match the API response verbatim (via `formatMoney` for the amount); no local proration formula is invoked anywhere in this component — confirms the frontend never re-derives a number the backend's `Section 13` proration formula already computed authoritatively |
| Empty refund queue renders as a clear steady state | mock `{ refunds: [] }` | render | "queue clear," not an error |
| RefundCard reject requires a reason before submit — **I1 fix** | render with a `PENDING` refund; open the reject prompt | attempt to submit with an empty reason | `useRejectRefund().mutate` is not called; a validation message is shown |
| RefundCard reject wires reason through to the mutation — **I1 fix** | render with a `PENDING` refund | fill in a reason and confirm reject | `useRejectRefund().mutate` called with `{ refundId, rejectionReason }` matching the entered text |
| RefundCard hides Approve/Reject once actioned — **I1 fix** | render with a refund at `status: 'APPROVED'`, then one at `status: 'REJECTED'` | — | neither the Approve nor Reject action is rendered in either case; the stored audit fields (`approvedBy`/`approvedAt` or `rejectedBy`/`rejectedAt`/`rejectionReason`) are shown instead |
| PayoutBatchTable "Mark paid" disabled once `status === 'PAID'` | render a row with `status: 'PAID'` | — | action disabled — an idempotency UI guard, explicitly not assumed to be backed by real backend idempotency (8-7), so this test is scoped to "the button can't be clicked again," not "clicking it twice is provably safe" |
| PromotionForm clamps `discountPercent` to 1–100 before submit | attempt `0`, then `101` | — | both rejected client-side before `useCreatePromotion().mutate` fires; `1` and `100` themselves are accepted (boundary-inclusive) |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] Every test in this file that asserts a rendered money value does so against `formatMoney`'s actual output or a spy on the function being called with the right raw string — no test hardcodes a pre-computed display string independent of the function under test, which would make the test pass even if `formatMoney` itself broke.
- [ ] `PricingConfigForm`'s arithmetic-guard test uses values whose native-float sum would produce a classic floating-point artifact (e.g. `0.1 + 0.2 !== 0.3`-style inputs) specifically to prove the decimal-safe library path is actually exercised, not values that would coincidentally sum correctly under naive float math too.
- [ ] `PaymentPausedPage`'s route-blocking case is tested as an actual navigation/redirect assertion, not merely "the nav link is hidden" — a hidden link is a UX nicety, but a reachable direct URL would be the real gap.
- [ ] `useInitiatePayment`'s redirect test asserts `window.location.href` was set to the exact returned `chapaCheckoutUrl`, not a hardcoded/assumed URL shape.
- [ ] **[Phase 4]** `money.test.ts`'s `.XX5` boundary case (`"33.125"` → `"33.13"`) is checked against the same rounding mode `9-7-payments-earnings.md` (backend) §9.12's `calculateProration` case uses — if the two ever specify different rounding modes, that is itself a bug this pair of tests exists to catch, not a discrepancy to quietly resolve by loosening either assertion.
- [ ] No test in this file coerces a money-as-string field to a JS `number` for a correctness assertion (e.g. `Number(rendered) === 1234.5`) — doing so would itself reintroduce the exact precision risk this whole feature's string convention exists to avoid, even inside a test.

---

### 9.8 Out of Scope for Automated Testing (and why)

- **Real Chapa checkout page behavior, webhook signature verification** — entirely external/server-side; `POST /payments/webhook/chapa`'s HMAC verification is covered by the backend's `9-7-payments-earnings.md`. This frontend doc only tests up to the redirect hand-off and the post-return refetch reconciliation.
- **`PaymentHistoryPage.tsx`, `EarningsPage.tsx`** — thin table/summary wrappers whose only non-generic behavior (money formatting, inline itemization) is asserted via a spy on `formatMoney`/the itemized-list rendering, not given a full dedicated block.
- **Decimal-safe library's own arithmetic correctness** — this doc verifies the app's code *calls* the library correctly and never falls back to native float math; the library's own precision guarantees are that library's test suite's responsibility.
- **Server-side idempotency guarantees on `POST /admin/payouts/:id/mark-paid`** — the frontend's disabled-button guard (§9.6) is explicitly a UX courtesy, not a substitute for backend idempotency, which is covered (if at all) in the backend's own `9-7-payments-earnings.md`.

---

**Next:** proceed to → [9-8. Frontend Test Documentation: Support, Trust & Admin Reporting]
