## Project: AKEWTutor — Frontend Test Documentation: Matching & Cohorts
**Links back to:** [05b §3], [07-03], [8-3. Frontend Function-Level Spec: Matching & Cohorts]
**Conventions:** see `9-1-shared-config.md` for shared setup/mock patterns and OWASP category definitions.

**Depends on:** Accounts & Guardianship (hard).

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered |
|---|---|
| hooks/useMatching.ts (useSearchTutors, useRecommendations, useSelectTutor, useNoExactMatch, useMyMatchRequests) | FR-MA-001–002, FR-MA-007, FR-MA-012, FR-MA-016–018, FR-SP-025 |
| hooks/useCohort.ts (useMyCohorts, useCohortMembers) | FR-MA-006, FR-MA-009, FR-MA-011, FR-MA-016, FR-SP-030 (profile-visibility split) |
| hooks/useAdminMatching.ts | FR-MA-003–004, FR-MA-008, FR-MA-010, FR-MA-013–015, FR-MA-017, FR-AD-005–008 |
| hooks/useFormatSwitch.ts | FR-SP-045–049 |
| components/NoExactMatchButton.tsx | FR-MA-002/UC-26 (48h escalation) |
| components/GroupAssignmentCard.tsx | FR-SP-030 (visibility floor) |
| pages/admin/MatchingQueuePage.tsx, components/ApprovalQueueTable.tsx | FR-AD-005–008 |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Notes |
|---|---|---|---|
| src/hooks/useMatching.ts | tests/hooks/useMatching.test.ts | Hook | mandatory — full blocks below |
| src/hooks/useCohort.ts | tests/hooks/useCohort.test.ts | Hook | mandatory — full block below |
| src/hooks/useAdminMatching.ts | tests/hooks/useAdminMatching.test.ts | Hook | mandatory — full block below |
| src/hooks/useFormatSwitch.ts | tests/hooks/useFormatSwitch.test.ts | Hook | mandatory |
| src/pages/student/FindTutorPage.tsx | tests/pages/FindTutorPage.test.tsx | Component | non-trivial: URL-param-driven filters |
| src/pages/student/TutorRecommendationsPage.tsx | tests/pages/TutorRecommendationsPage.test.tsx | Component | non-trivial: empty-list branch renders `NoExactMatchButton` |
| src/components/NoExactMatchButton.tsx | tests/components/NoExactMatchButton.test.tsx | Component | non-trivial: live countdown, clamping — full block below |
| src/components/GroupAssignmentCard.tsx | tests/components/GroupAssignmentCard.test.tsx | Component | non-trivial: visibility floor is security-relevant — full block below |
| src/pages/student/GroupFormatStatusPage.tsx | tests/pages/GroupFormatStatusPage.test.tsx | Component | non-trivial: derived tab state |
| src/pages/student/FormatSwitchPage.tsx | tests/pages/FormatSwitchPage.test.tsx | Component | non-trivial: pre-fill + submit |
| src/pages/admin/MatchingQueuePage.tsx, components/ApprovalQueueTable.tsx | tests/pages/MatchingQueuePage.test.tsx | Component | non-trivial: overdue visual state, required-reason reject |
| src/pages/admin/ManualAssignmentPage.tsx, components/ManualAssignmentForm.tsx | tests/components/ManualAssignmentForm.test.tsx | Component | non-trivial: format-size-constrained submit gate |
| src/pages/student/TutorProfileViewPage.tsx | tests/pages/TutorProfileViewPage.test.tsx | Component | thin wrapper around `useTutorFullProfile`; 404 state covered, no separate full block |

---

### 9.2 Test Case Detail — useMatching.test.ts (full blocks)

**OWASP: A04:2021 – Insecure Design (empty-filter guard prevents an inefficient/abusable unfiltered query).**

#### useSearchTutors

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Does not fire with an empty filter set | render hook with `filters = {}` | — | query `enabled` is `false`; no request made — confirms the "start by selecting a filter" empty state has no unfiltered-catalog request lurking behind it (8-3) |
| Fires once any single filter is set | `filters = { subjectId: 'x' }` | — | `enabled` true, request made |
| Re-keys (fresh request) per filter change, not a client-side re-filter | change `filters.grade` after an initial successful fetch | — | a **new** request fires with the updated filter set — asserted via the mock being called again with new params, not just the returned data changing, which would also be true under an (incorrect) client-side-filter implementation |

#### useRecommendations

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Empty `recommendations: []` is a normal success, not treated as an error | mock success with `recommendations: []` | — | `isError` is `false`, `isSuccess` is `true` — per API convention §0.3, consumed by `TutorRecommendationsPage`'s empty-branch |
| Fires on mount with no `enabled` gate | render hook | — | request fires immediately, unlike `useSearchTutors` above |

#### useSelectTutor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No automatic query invalidation on success | mock success | `mutate({ tutorId, studentId })` | no query key invalidated by the hook itself — confirms `useMyCohorts` is expected to be picked up fresh by the calling page's own navigation, not patched here (8-3) |
| 409 (capacity/expired) surfaces as a distinct, retryable-by-navigation error | mock `409` | `mutate(...)` | error propagates; hook does not auto-retry the same stale `tutorId` |

#### useNoExactMatch

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success triggers no local cache write | mock success | `mutate(studentId)` | no invalidation called directly by this hook — status pickup is the calling page's responsibility via the polled `useMyMatchRequests`/`useMyCohorts` |

#### useMyMatchRequests

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Polls at 30s | fake timers | advance 30_000ms | query re-fires — required since `zeroMatchEscalation.job.ts`/`staleApproval.job.ts` mutate this server-side with no client trigger (§0.4) |

---

### 9.3 Test Case Detail — useCohort.test.ts, useAdminMatching.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `useMyCohorts` polls at 30s | fake timers | advance 30_000ms | re-fires — `groupFormationWindow.job.ts` flips status server-side with no client action |
| `useApprovalQueue` polls at 20s | fake timers | advance 20_000ms | re-fires — `staleApproval.job.ts` flips `adminOverdueNotifiedAt` server-side |
| Neither polling hook exposes a manual "check now" action | inspect the hook's returned object | — | no `refetch`-triggering button/affordance is wired anywhere in the consuming pages (asserted at the page level in §9.5/§9.6, not the hook itself, since the hook legitimately does expose TanStack Query's own `refetch` — the point being tested is that the UI never surfaces it) |

---

### 9.4 Test Case Detail — NoExactMatchButton.test.tsx

FRs: FR-MA-002 / UC-26. **OWASP: A04:2021 – Insecure Design (client-only countdown must not be treated as authoritative).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Countdown computed from `zeroMatchSince`, ticking via `setInterval` | `zeroMatchSince = <40h ago>`, fake timers | advance 1s | displayed remaining time decreases by 1s, computed as `48h - elapsed`, not re-fetched from the server per tick |
| `zeroMatchSince: null` renders the button without a countdown | render with `zeroMatchSince: null` | — | button visible, no countdown text — the "recommendations just came back empty this instant" case |
| Clamps at `0h 0m` once the 48h threshold has elapsed | `zeroMatchSince = <50h ago>` (already past threshold) | render | displayed remaining time is exactly `0h 0m`, never negative; button copy reads "Escalating automatically" rather than implying manual action is still needed — this is the explicit edge case 8-3 flags for when the poll hasn't yet caught up to the server's own auto-escalation |
| Clicking calls `onTrigger`, not the mutation directly | render with an `onTrigger` spy | click | spy called — confirms the component doesn't call `useNoExactMatch` internally, keeping navigation-on-success at the call site per 8-3 |

---

### 9.5 Test Case Detail — GroupAssignmentCard.test.tsx

FRs: FR-SP-030 (visibility floor). **OWASP: A01:2021 – Broken Access Control / excessive data exposure.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders only `displayName` + `profilePictureUrl` | render with a `CohortMember` object | — | no other field from the member object appears in the rendered output |
| **Type-level enforcement, not just a runtime omission** — even if the prop object happens to carry extra fields, they are never rendered | pass a `CohortMember`-typed object that (via an `as any` cast, simulating a future API response drift) also carries a `qualifications`/`email`/`phone` field | render | none of those extra fields appear anywhere in the DOM — this is the single highest-priority security-relevant test in this feature file: the deliberate group-format visibility floor (Doc 02 §5.6) depends on this component never accidentally rendering more than name+photo even if a future backend change starts including more data in the response, since the component's own prop type is the last line of defense the frontend controls (OWASP A01:2021 – Broken Access Control, specifically excessive data exposure to co-members in a group cohort who should not see each other's contact/profile detail) |
| Missing `profilePictureUrl` renders a fallback, not a broken image | render with `profilePictureUrl: null` | — | placeholder/initials avatar renders, no broken `<img>` |

---

### 9.6 Test Case Detail — GroupFormatStatusPage.test.tsx, FormatSwitchPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `activeTab` is derived, not separately tracked | mock `useMyCohorts` → a cohort with `status: 'FORMING'` | render | "waiting" tab active — no separate `useState('waiting')` driving this; changing the mocked cohort's status to `'ACTIVE'` and re-rendering flips the tab with no explicit user action, confirming the derivation is live per-render (8-3) |
| No manual refresh affordance | render the page | — | no "refresh"/"check now" button present anywhere on this page (ties to §9.3's note) |
| `CountdownTimer` renders only when `groupFormationWindowExpiresAt` is present | mock a cohort without that field | render | no countdown shown |
| FormatSwitchPage pre-fills `targetFormat` from the current cohort's "next" option but remains changeable | navigate in with a `ONE_TO_ONE` cohort | render, then change the selector | initial value is `ONE_TO_THREE`; user can override before submit |
| Submit calls `useRequestFormatSwitch` with the current selection, not the original pre-fill, if changed | change the selector then submit | — | mutation payload reflects the user's final choice |

---

### 9.7 Test Case Detail — MatchingQueuePage.test.tsx (ApprovalQueueTable), ManualAssignmentForm.test.tsx

**OWASP: A01:2021 – Broken Access Control (Admin-only), A04:2021 – Insecure Design (format-size UX guard).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Overdue rows get distinct visual treatment purely from `adminOverdueNotifiedAt` | mock one row with the field set, one without | render | only the flagged row shows the overdue treatment — no separate client-side staleness calculation, confirming `staleApproval.job.ts` is the sole source of truth per 8-3 |
| Reject requires a non-empty reason | click "Reject" on a row | leave reason blank | `onReject` not called until a reason is entered — same required-reason convention as 8-2 |
| Empty queue renders `EmptyState` | mock `{ items: [] }` | render | `EmptyState`, not an error — steady-state "queue clear" per 8-3 |
| ManualAssignmentForm: "Assign" disabled until `studentIds.length` matches the selected format | select `format: 'ONE_TO_THREE'` with only 1 student chosen | — | Assign button `disabled` |
| ManualAssignmentForm: exact-1 required for `ONE_TO_ONE` | select `format: 'ONE_TO_ONE'` with 2 students chosen | — | Assign button `disabled` — "exactly 1," not "up to 1," per 8-3's explicit distinction between `ONE_TO_ONE` (exact) and the other two formats (up to N) |
| ManualAssignmentForm: submits the exact tutor/student/format triple | fill correctly, submit | — | `useManualAssign().mutate({ tutorId, studentIds, format })` called with the exact selections |

---

### 9.8 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `GroupAssignmentCard`'s extra-field test actually injects a field the type wouldn't normally allow (via a deliberate cast) — a test that only ever passes well-typed `CohortMember` objects would never exercise the defense this component exists to provide.
- [ ] `NoExactMatchButton`'s clamp test uses an elapsed time strictly greater than 48h (not exactly 48h) to confirm the clamp handles the "poll is behind the server" case, not just the boundary itself.
- [ ] `useSearchTutors`'s re-key test asserts the mock fetch function was called a second time with new arguments, not merely that the returned `data` object changed reference.
- [ ] `ManualAssignmentForm`'s `ONE_TO_ONE` case is tested as "exactly 1," with both an under-count (0) and an over-count (2) case, not just one direction.
- [ ] No test in this file asserts a manual "refresh" control exists for any of the three polling hooks — an accidentally-added refresh button anywhere in this feature is itself the regression `useCohort.test.ts`/`useAdminMatching.test.ts`'s §9.3 note exists to catch, so a reviewer should treat a newly-appearing refresh-button test here with suspicion, not approval.

---

### 9.9 Out of Scope for Automated Testing (and why)

- **`TutorProfileViewPage.tsx`** — a thin wrapper around `useTutorFullProfile`; its loading/error/success states are the same generic pattern already exercised across every other simple detail page in this doc set, not repeated in full here.
- **Real scoring/matching algorithm correctness** — entirely server-side (`matching.service.ts`); covered by the backend's own `9-3-matching-cohorts.md`. This doc only verifies the frontend renders and reacts to whatever the API returns.
- **Drag/multi-select picker library internals in `ManualAssignmentForm`'s tutor/student pickers** — the test suite asserts the resulting selection state and submit-gate logic, not the picker widget's own internal event handling.

---

**Next:** proceed to → [9-4. Frontend Test Documentation: Class Delivery, Recording & Library]
