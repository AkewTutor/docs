## Project: AKEWTutor — Frontend Test Documentation: In-Platform Messaging
**Links back to:** [05b §5], [07-05], [8-5. Frontend Function-Level Spec: In-Platform Messaging]
**Conventions:** see `9-1-shared-config.md` for shared setup/mock patterns and OWASP category definitions.

**Depends on:** Matching & Cohorts (hard).

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| hooks/useMessaging.ts (useMessages, useSendMessage) | FR-MS-001–003, FR-SC-004, FR-NO-011 | NFR-009 |
| hooks/useAdminMessaging.ts (useReviewThread, useCloseThread) | FR-MS-004, FR-AD-017 | NFR-009 |
| pages/MessagingPage.tsx, components/MessageThreadView.tsx | FR-MS-001–003 | NFR-009 |
| components/MessageComposer.tsx | FR-SC-004 (text-only rule) | — |
| pages/admin/MessageThreadReviewPage.tsx | FR-MS-004, FR-AD-017 | NFR-009 |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Notes |
|---|---|---|---|
| src/hooks/useMessaging.ts | tests/hooks/useMessaging.test.ts | Hook | mandatory — full blocks below |
| src/hooks/useAdminMessaging.ts | tests/hooks/useAdminMessaging.test.ts | Hook | mandatory |
| src/pages/MessagingPage.tsx | tests/pages/MessagingPage.test.tsx | Component | non-trivial: zero/one/many-cohort branching |
| src/components/MessageThreadView.tsx | tests/components/MessageThreadView.test.tsx | Component | non-trivial: identity-display rule, 403 handling — full block below |
| src/components/MessageComposer.tsx | tests/components/MessageComposer.test.tsx | Component | non-trivial: closed-thread gate, no-attachment guarantee — full block below |
| src/pages/admin/MessageThreadReviewPage.tsx | tests/pages/MessageThreadReviewPage.test.tsx | Component | non-trivial: required-reason close action |

---

### 9.2 Test Case Detail — useMessaging.test.ts (full blocks)

**OWASP: A01:2021 – Broken Access Control.**

#### useMessages

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `enabled: false` with no `cohortId` | render with `cohortId: undefined` | — | no request |
| Polls at 8s unconditionally | fake timers | advance 8_000ms | re-fires — 8-5 is explicit this is the one query in the app that polls purely due to the absence of a websocket in V1, not a job-driven state; flagged here so a future V2 socket migration knows exactly which interval to remove |

#### useSendMessage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success invalidates `[MESSAGES, cohortId]`, no optimistic append | mock success | `mutate(body)` | that query key invalidated; the message list is not directly mutated in the cache — confirms 8-5's explicit choice to avoid reconciling a client-generated temp ID against the server's real one |
| 403 is not handled by this hook | mock a `403` (either the "not available" or `CLOSED_BY_ADMIN` variant) | `mutate(body)` | the error propagates unhandled by the hook itself — `MessageComposer` is responsible for the two distinct 403 UX cases, tested separately below; this hook-level test exists to confirm the separation of concerns actually holds (a hook that silently swallowed or reinterpreted the error would break the composer's ability to distinguish the two cases) |

---

### 9.3 Test Case Detail — MessagingPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Zero cohorts renders `EmptyState`, no thread UI | mock `useMyCohorts` → `[]` | render | `EmptyState` ("no active class yet"); no picker, no `MessageThreadView` |
| Exactly one active cohort skips the picker | mock one cohort | render | `MessageThreadView` renders directly, `selectedCohortId` auto-set |
| More than one cohort shows a picker first | mock two cohorts | render | picker list shown; selecting one sets `selectedCohortId` and renders the thread view |

---

### 9.4 Test Case Detail — MessageThreadView.test.tsx (full block)

FRs: FR-MS-001–003. **OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sender identity shown only when `format !== 'ONE_TO_ONE'` | mock `useThread` → `participantCount: 2` (pair thread), then `participantCount: 3+` (group) | render each | sender name/role hidden in the pair case, shown in the group case — the single component branches on this one condition rather than two separate components existing (8-5) |
| 403 from either `useThread` or `useMessages` replaces the whole view with `EmptyState` | mock either query to reject with `403` | render | `EmptyState` + link back to dashboard — this must **not** render as a generic error toast, since a not-yet-confirmed or ended cohort's thread is an expected state per 8-5, not a bug |
| Archived thread (90+ days ended) still renders read-only when navigated to directly | mock `useThread`/`useMessages` for an old, ended cohort, navigated to via a direct link (bypassing `MessagingPage`'s own "active" filter) | render | full history renders — confirms the "active" filter is `MessagingPage`-only and this component itself always shows what it's given a valid `cohortId` for |
| Passes `useThread`'s `status` through to `MessageComposer` | mock `useThread` → `status: 'CLOSED_BY_ADMIN'` | render | `MessageComposer` receives that status as a prop (asserted via the composer rendering its own closed-state copy, tested directly in §9.5) |

---

### 9.5 Test Case Detail — MessageComposer.test.tsx (full block)

FRs: FR-SC-004. **OWASP: A03:2021 – Injection (no attachment surface at all is itself a defense against file-upload-borne attacks in V1).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Body length validated client-side (1–2000 chars) | attempt to submit empty, then a 2001-char string | — | submit blocked in both cases before any `mutate` call |
| `CLOSED_BY_ADMIN` renders disabled with distinguishing copy | render with `threadStatus: 'CLOSED_BY_ADMIN'` | — | composer disabled, copy specifically reads "This conversation has been closed" — read directly from `useThread`'s metadata, not parsed out of a 403 error string (8-5's explicit guidance) |
| Successful send clears the input | mock success | submit, observe post-submit state | input cleared |
| **No attachment/file-upload affordance exists anywhere in the DOM, not even a disabled one** | render in any `threadStatus` | inspect the full rendered output | zero file-input elements, zero "attach"/"upload" buttons of any kind, enabled or disabled — 8-5 is explicit that even a visibly-disabled control would misrepresent the feature as "coming soon" rather than the deliberate, permanent text-only rule (FR-SC-004); this is a real assertion (absence of specific DOM elements), not a skipped test, and exists precisely so a future contributor adding a disabled "Attach file (coming soon)" button gets caught by CI rather than merged silently |

---

### 9.6 Test Case Detail — MessageThreadReviewPage.test.tsx

**OWASP: A01:2021 – Broken Access Control (Admin-only).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Read-only full history, no composer rendered at all | render as Admin | — | no `MessageComposer` anywhere on this page — Admin reviews, never participates in the thread |
| Close action requires a non-empty reason | click "Close conversation" | leave reason blank | `useCloseThread().mutate` not called until a reason is entered — same required-reason convention used across every admin action that terminates something for another party (8-2, 8-3, 8-5) |
| 404 (thread not found) renders distinctly from a loading/empty state | mock `useReviewThread` → `404` | render | error state, not a blank/loading-forever screen |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `MessageComposer`'s no-attachment case queries for the *absence* of file-input/upload-button elements using an explicit `queryBy*` (returns `null` rather than throwing) and asserts `null`, not a `getBy*` wrapped in `.not.toBeInTheDocument()` on a selector that might silently match nothing for an unrelated reason.
- [ ] `useMessages`'s 8-second polling test is not accidentally merged with or confused for a job-driven polling test (like `useMyCohorts` in 9-3) in code review — the comment trail in the test file should make clear this one polls due to no-websocket, not a server job.
- [ ] `MessageThreadView`'s 403 case is tested for **both** `useThread` and `useMessages` failing with 403 independently, not just one, since either query could plausibly be the one to fail first in practice.
- [ ] `MessageThreadReviewPage`'s "no composer" test is a real assertion against the rendered DOM, not an assumption based on the page not importing `MessageComposer` (an accidental import with conditional rendering could still leak it).

---

### 9.8 Out of Scope for Automated Testing (and why)

- **Real-time delivery latency/ordering under concurrent senders** — this is a polling-based V1 design (8-5); no websocket exists to test message-ordering guarantees against. The 8-second interval itself is the only "real-time" behavior in scope.
- **Message content moderation/profanity filtering** — not specified anywhere in Docs 02/06/08 for this feature; not fabricated here.
- **CSRF** — not applicable; stateless Bearer-JWT, no cookie session, per the same reasoning in `9-1-shared-config.md`.

---

**Next:** proceed to → [9-6. Frontend Test Documentation: Gamification & Engagement]
