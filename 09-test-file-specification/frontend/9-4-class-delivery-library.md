## Project: AKEWTutor — Frontend Test Documentation: Class Delivery, Recording & Library
**Links back to:** [05b §4], [07-04], [8-4. Frontend Function-Level Spec: Class Delivery, Recording & Library]
**Conventions:** see `9-1-shared-config.md` for shared setup/mock patterns and OWASP category definitions.

**Depends on:** Matching & Cohorts (hard).

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| hooks/useSessions.ts | FR-CD-001–003 | — |
| hooks/useRecordingConsent.ts | FR-SC-008–009 | — |
| hooks/useRecordings.ts | FR-SP-035–037, FR-CD-004–008 | NFR-007, NFR-009, NFR-012 |
| hooks/useLibrary.ts | FR-CD-009, FR-AD-014 | — |
| hooks/useReschedule.ts, lib/classifyReschedule.ts | FR-MK-004, FR-MK-006–008 | — |
| hooks/useWeeklyAssessment.ts | FR-SP-038, FR-TU-017 | — |
| components/SessionCard.tsx, RecordingIndicatorBanner.tsx | FR-CD-001–003, FR-SC-008–009 | — |
| components/RecordingPlayer.tsx | FR-SP-035–037 | NFR-007, NFR-009 |
| components/RescheduleForm.tsx | FR-MK-004, FR-MK-006–008 | — |
| pages/admin/RecordingComplianceQueuePage.tsx | FR-AD-014 | NFR-009 |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Notes |
|---|---|---|---|
| src/hooks/useSessions.ts | tests/hooks/useSessions.test.ts | Hook | mandatory — full block below (15s polling) |
| src/hooks/useRecordingConsent.ts | tests/hooks/useRecordingConsent.test.ts | Hook | mandatory |
| src/hooks/useRecordings.ts | tests/hooks/useRecordings.test.ts | Hook | mandatory — full blocks below (`useUploadRecording`, `useSignedUrl`) |
| src/hooks/useLibrary.ts | tests/hooks/useLibrary.test.ts | Hook | mandatory |
| src/hooks/useReschedule.ts | tests/hooks/useReschedule.test.ts | Hook | mandatory — full block below |
| src/lib/classifyReschedule.ts | tests/lib/classifyReschedule.test.ts | Unit | mandatory (util, per 8-4) — full block below |
| src/hooks/useWeeklyAssessment.ts | tests/hooks/useWeeklyAssessment.test.ts | Hook | mandatory |
| src/components/SessionCard.tsx | tests/components/SessionCard.test.tsx | Component | non-trivial: join-button gate, countdown, conditional reschedule link |
| src/pages/student/UpcomingClassesPage.tsx | — | — | Not required — thin route/layout wrapper around `useUpcomingSessions()` + a stable ascending sort with no branching logic; covered by `SessionCard.test.tsx` plus a smoke test for the empty-list state, matching the `AvailabilityPage.tsx`/`SubjectRankingPage.tsx` precedent in `9-2-accounts-guardianship.md §9.1` |
| src/pages/tutor/ConductClassPage.tsx | tests/pages/ConductClassPage.test.tsx | Component | non-trivial: link-submission vs. complete-action branching, time gate |
| src/components/RecordingIndicatorBanner.tsx | — | — | purely presentational, `visible` prop only — see §9.9 |
| src/pages/RecordingConsentPage.tsx | tests/pages/RecordingConsentPage.test.tsx | Component | non-trivial: acknowledged vs. not-yet branch |
| src/pages/LibraryPage.tsx | tests/pages/LibraryPage.test.tsx | Component | non-trivial: role-gated upload form, cohort selector |
| src/components/RecordingPlayer.tsx | tests/components/RecordingPlayer.test.tsx | Component | non-trivial: signed-URL retention edge case — full block below |
| src/components/MaterialUploadForm.tsx | tests/components/MaterialUploadForm.test.tsx | Component | non-trivial: dual-field submit gate |
| src/pages/RequestReschedulePage.tsx, components/RescheduleForm.tsx | tests/components/RescheduleForm.test.tsx | Component | non-trivial: live classification vs. server reconciliation — full block below |
| src/pages/tutor/WeeklyAssessmentPage.tsx, components/AssessmentForm.tsx | tests/pages/WeeklyAssessmentPage.test.tsx | Component | non-trivial: client-side duplicate-submission check |
| src/pages/student/ProgressPage.tsx | tests/pages/ProgressPage.test.tsx | Component | thin wrapper; empty/success states only |
| src/pages/admin/RecordingComplianceQueuePage.tsx | tests/pages/RecordingComplianceQueuePage.test.tsx | Component | non-trivial: two-tier severity display |

---

### 9.2 Test Case Detail — useSessions.test.ts, useRecordings.test.ts (full blocks)

**OWASP: A01:2021 – Broken Access Control (signed-URL retention), A04:2021 – Insecure Design.**

#### useSession

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Polls at 15s | fake timers, `sessionId` set | advance 15_000ms | re-fires — `recordingMissingCheck.job.ts` flips `recordingStatus` server-side with no client trigger |
| `enabled: false` with no `sessionId` | render with `sessionId: undefined` | — | no request fires |

#### useUploadRecording

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Builds `FormData` from `sessionId` + `file` | mock upload success | `mutate({ sessionId, file })` | request body is a real `FormData` instance containing both fields, not a JSON object |
| No automatic `[RECORDINGS]` invalidation | mock success | `mutate(...)` | no query key invalidated — the success screen explicitly tells the caller to check the session detail page instead, per 8-4's note that this is a rare manual-fallback path |
| No client-side file-type/size validation duplicated in the hook | pass an arbitrary `File` object of any mime-type/size | `mutate(...)` | the hook does not reject it itself — any restriction lives in the file picker's `accept` attribute (tested at the component level, not here) or the backend; this hook is a thin wrapper only |

#### useSignedUrl

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `enabled: false` with `recordingId: null` | render with `null` | — | no request |
| `retry: false` is set — a 404 does not retry | mock a `404` | — | exactly one request attempt is made, confirmed via the mock call count staying at 1 even after the query's internal retry logic would otherwise re-attempt; this matters because a 404 here means the recording is genuinely gone past its 90-day retention (§0.3), not a transient failure — retrying would only delay the correct `EmptyState` |

---

### 9.3 Test Case Detail — useReschedule.test.ts, classifyReschedule.test.ts (full blocks)

FRs: FR-MK-004, FR-MK-006–008. **OWASP: A08:2021 – Software and Data Integrity Failures (client preview must never override the server's authoritative value).**

#### classifyReschedule

| Case | Setup | Action | Expected result |
|---|---|---|---|
| ≥12h before session → `'FREE_RESCHEDULE'` | `hoursUntilSession = 12` exactly, and `13` | call `classifyReschedule(sessionScheduledStart, requestedAt)` | `'FREE_RESCHEDULE'` in both — confirms the boundary is inclusive at exactly 12h |
| <12h before session → `'SAME_DAY_MISS'` | `hoursUntilSession = 11.999` | call | `'SAME_DAY_MISS'` |
| Negative `hoursUntilSession` (session already started/past) still resolves to `'SAME_DAY_MISS'` | `requestedAt` after `sessionScheduledStart` | call | `'SAME_DAY_MISS'` — 8-4 explicitly notes this util does not separately guard against that case, relying on the form to prevent selecting a past/in-progress session as the target in the first place; this test documents that reliance rather than silently assuming it |
| **[Phase 4 — Review §6.1] hoursUntilSession is computed from Date instants, unaffected by the browser's local DST zone** | mock `Date.now()`/`requestedAt` and `sessionScheduledStart` as UTC instants exactly 12 hours apart, with the test environment's `Intl`/`Date` locale set to a DST-observing zone (e.g. `America/New_York`) straddling a transition date | call `classifyReschedule(sessionScheduledStart, requestedAt)` | resolves `'FREE_RESCHEDULE'` regardless of the browser's local zone — `hoursUntilSession` is derived from the millisecond difference between the two `Date` instants, never from a re-parsed local-calendar difference that a DST-observing browser locale could shift by an hour; this is a client-side belt-and-suspenders check, since `useRequestReschedule`'s "server value wins" case above already prevents any such drift from actually reaching the person as a wrong confirmation |

#### useRequestReschedule

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success confirmation uses the **server's** `classification`, not the client preview | mock the mutation to resolve with `classification: 'SAME_DAY_MISS'` while the form's live preview (computed via `classifyReschedule`) showed `'FREE_RESCHEDULE'` a moment before submit | `mutate(...)`, then render the confirmation | confirmation displays `'SAME_DAY_MISS'` — the server value wins outright, per 8-4's explicit clock-drift-disagreement note; this is the single most important integrity test in this feature, since silently trusting the client's own guess here would let a rescheduled session's fee/make-up eligibility display incorrectly to the person even though the backend's persisted value is different (OWASP A08:2021) |

---

### 9.4 Test Case Detail — SessionCard.test.tsx, ConductClassPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| "Join" disabled until `jitsiLinkUrl` is non-null | render with `jitsiLinkUrl: null` | — | Join button `disabled` |
| "Join" enabled once a link exists, with no separate time-window check | render with `jitsiLinkUrl` set, `scheduledStart` far in the future | — | Join button enabled — confirms 8-4's explicit statement that link-presence is the *sole* gate, no "only enable 10 minutes before" logic exists client-side to accidentally contradict |
| "Request reschedule" link hidden once `status` is `COMPLETED`/`MISSED` | render with each status | — | link absent in both cases, present otherwise |
| ConductClassPage: "Mark completed" disabled until `scheduledEnd` has passed | mock `Date.now()` to be before `scheduledEnd` | render | button `disabled`; advancing mocked time past `scheduledEnd` and re-rendering enables it — prevents a tutor marking a session complete before it's actually over |
| ConductClassPage: link-submission form shown only when `jitsiLinkUrl` is null | render with `jitsiLinkUrl: null`, then with a URL set | — | form shown only in the first case; link + complete-action shown in the second |
| `RecordingIndicatorBanner` visibility cross-references consent status | mock `useConsentStatus` → `acknowledged: false` | render `ConductClassPage` | banner not visible; flipping to `acknowledged: true` shows it |

---

### 9.5 Test Case Detail — RecordingConsentPage.test.tsx, LibraryPage.test.tsx, RecordingPlayer.test.tsx (full block), MaterialUploadForm.test.tsx

**OWASP: A01:2021 – Broken Access Control (tutor-only upload gate), A05:2021 – Security Misconfiguration (signed-URL is short-lived and never cached beyond its query).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| RecordingConsentPage: acknowledged renders confirmation with timestamp | mock `acknowledged: true, acknowledgedAt` | render | confirmation view, no acknowledge button |
| RecordingConsentPage: not acknowledged renders the action | mock `acknowledged: false` | render | consent text + "Acknowledge" button, wired to `useAcknowledgeConsent` |
| LibraryPage: upload form gated on `role === 'TUTOR'` | render with `auth.store` role `STUDENT`, then `TUTOR` | — | `MaterialUploadForm` absent for Student, present for Tutor — client-side UX gate; the real enforcement is the backend, but a leaked upload control in the Student UI would be a confusing/misleading affordance at minimum |
| LibraryPage: "Keep permanently" visible to any participant, not tutor-only | render as a Student | — | action is present — confirms this is deliberately not restricted the way upload is (8-4: the benefit accrues to the whole cohort) |
| LibraryPage: no cohorts yet renders a prompt, not the recordings/materials sections | mock `useMyCohorts` → `[]` | render | "check back once enrolled" prompt; no per-section `EmptyState` rendered underneath since there's no cohort to select in the first place |

#### RecordingPlayer (full block)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Skeleton while loading, never a blank pane | mock `useSignedUrl` pending | render | skeleton visible |
| Success renders a `<video>` sourced from the signed URL | mock success with `url` | render | `<video src=...>` matches |
| 404 renders `EmptyState` with "Recording no longer available," never a generic error banner | mock `isError: true` (per `retry: false`) | render | `EmptyState` with that specific copy — this is a documented expected outcome per §0.3, not a bug, so a generic "Something went wrong" banner here would be a UX regression the spec explicitly warns against |
| Signed URL is never persisted beyond the query's own cache lifetime | mock a successful fetch, then simulate the query going stale/refetching | inspect any storage layer (`localStorage`, module-level variable) | the URL is not written anywhere outside TanStack Query's own cache — a short-lived signed URL persisted longer than intended would undermine the backend's expiry control (OWASP A05:2021-adjacent: sensitive credential/URL over-retention on the client) |

#### MaterialUploadForm

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Submit disabled until both `title` and `file` are set | render, fill only `title` | — | submit disabled; filling `file` too enables it |

---

### 9.6 Test Case Detail — RescheduleForm.test.tsx (full block), WeeklyAssessmentPage.test.tsx, RecordingComplianceQueuePage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Live badge updates as `requestedNewTime` changes | pick a time ≥12h out, then change to <12h out | — | badge flips from `FREE_RESCHEDULE` to `SAME_DAY_MISS` live, computed via `classifyReschedule`, before any submit |
| Confirmation reconciles to the server's `classification` even if it differs from the last-shown live badge | see §9.3's `useRequestReschedule` case — re-asserted here at the full-form integration level | submit, mock a differing server response | confirmation shows the server value, not the pre-submit badge |
| WeeklyAssessmentPage: already-submitted-this-week renders read-only with "Edit" | mock `useAssessmentsForStudent` containing an entry for the current week | render that student's form | read-only view + "Edit" affordance, not an open editable form allowing a silent duplicate submit — explicitly a UX convenience per 8-4, not a guaranteed duplicate-prevention (the backend's own uniqueness rule, if any, remains authoritative; this test does not claim otherwise) |
| WeeklyAssessmentPage: one form per student, submitted independently | render a cohort with 3 members | submit one student's form | only that student's `useSubmitAssessment` mutation fires; the other two forms remain untouched/unsubmitted |
| RecordingComplianceQueuePage: `ESCALATED` visually distinguished from `MISSING` | mock one row of each status | render | distinct visual treatment per status, matching the severity-tiering convention shared with `ApprovalQueueTable` (9-3) and `PlatformStatsGrid` (9-8) |
| RecordingComplianceQueuePage: empty queue is a clear steady state | mock `{ items: [] }` | render | no error, a calm "queue clear" state |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The `useRequestReschedule` and `RescheduleForm` server-reconciliation tests both use a case where the live preview and the server's actual `classification` **genuinely disagree** — a test where they happen to match would not exercise the reconciliation logic at all, only the happy path.
- [ ] `classifyReschedule`'s 12h-boundary test checks exactly 12h (inclusive) and just under it, not just "some number well above/below 12."
- [ ] `useSignedUrl`'s `retry: false` case asserts the mock's call count stays at exactly 1, not merely that the component eventually shows the 404 state (which could pass even if a retry silently happened first).
- [ ] `SessionCard`'s "no time-window check" case is a genuine positive assertion (button enabled with a far-future `scheduledStart`), not the absence of a test — this guards against someone later "improving" the component with an undocumented time-window restriction that would contradict 8-4.
- [ ] **[Phase 4]** The DST case in `classifyReschedule.test.ts` actually configures a DST-observing locale/zone in the test environment (not the default CI runner zone, which may be UTC and would make the test pass trivially regardless of whether the implementation is instant-based or wall-clock-based) — same false-pass risk flagged in the backend doc's equivalent note.
- [ ] `RecordingPlayer`'s over-retention check inspects actual storage/module state, not just that the component re-fetches correctly on remount.

---

### 9.8 Out of Scope for Automated Testing (and why)

- **`RecordingIndicatorBanner.tsx`** — purely presentational, a single `visible: boolean` prop with no branching of its own (8-4 explicitly notes "no polling or data-fetching of its own"); its correct visibility is exercised through the host pages' tests (`ConductClassPage`, LibraryPage-adjacent flows) rather than duplicated here.
- **`ProgressPage.tsx`** — thin wrapper around `useAssessmentsForStudent`; loading/empty/success states are the same generic pattern covered elsewhere in this doc, not repeated in full.
- **Real Jitsi link generation/validity** — out of scope entirely; per `00-api-conventions.md` §0.5, this app never calls a Jitsi API and only stores/displays a tutor-pasted URL. There is no client-side test that could meaningfully verify a third-party video link works.
- **Real Cloudflare R2 signed-URL generation or expiry timing** — server-side; this doc only tests the frontend's handling of whatever URL/expiry the backend returns.
- **File-type/size enforcement UI beyond the file picker's `accept` attribute** — 8-4 is explicit that no such validation is specified at the frontend layer; a test asserting a rejection here would be testing for a control that was deliberately not built.

---

**Next:** proceed to → [9-5. Frontend Test Documentation: In-Platform Messaging]
