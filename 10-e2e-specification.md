## Project: AKEWTutor — End-to-End Test Specification (Tier 4 — Playwright)

**Links back to:** [02-requirements.md §17 — End-to-End User Journey], [09-test-file-specification/ — Doc 9, Tiers 1–3 (Unit, Integration (Persistence), Integration (HTTP contract))].
**Conventions:** see `09-test-file-specification/00-api-conventions.md` for the endpoint conventions referenced in the seeding steps below; see `09-test-file-specification/00-test-fixtures.md` for the factory functions referenced in seed scripts; see `09-test-file-specification/00-agent-rules.md` — rules 1, 3, 8, 9, and 10 bind this document directly, and rule 8 in particular governs every "third-party fake strategy" note below.

Per the standing rule: Playwright, `test`/`expect`, one spec file per journey under `tests/e2e/`, file name mirrors the journey slug (e.g. `tests/e2e/e2e-1-core-conversion-funnel.e2e.spec.ts`).

> **Update (6th pass — Three-Role Coverage Audit):** `02-requirements.md §17`'s 13-step journey is written entirely from the **Student's** point of view, and the client asked whether the **Tutor's** and **Admin's** own journeys — not just their walk-on appearances inside the student's story — are actually covered. Re-reading all 95 use cases in `03-usecases.md` against E2E-1 through E2E-8 found: **partially**. Admin and Tutor already appear as real actors in several existing journeys (E2E-1's approval, E2E-6's rejection/resubmission, E2E-7's dispute resolution, E2E-8's thread closure), but eight genuine cross-service seams that meet this document's own §10.0 bar — not just any uncovered use case — had no E2E journey at all: the two alternate matching paths that make up two-thirds of the format catalogue (Path B manual assignment, Path C group auto-match), Admin's booking-rejection reroute, a Tutor's mid-cohort exit/suspension and the group-continuity + refund cascade it triggers, a live Admin pricing/promotion change composing correctly into a real payment, the payment-lapse → auto-reschedule cycle, the Tutor earnings-ledger → payout-batch → Admin-mark-paid pipeline, and the entire auth/session lifecycle (login, refresh rotation, reuse detection, password-reset revocation) — which had **zero** E2E coverage despite every other journey in this document depending on it working. **E2E-9 through E2E-16 (§10.11–§10.18) close these eight gaps.** Not every uncovered use case became a new spec — see the expanded §10.20 for what was deliberately left out and why, in keeping with this document's own "document why, don't just skip" convention.

---

### 10.0 Purpose and Position in the Test Pyramid

This is **Tier 4** of the four-tier architecture (Unit → Integration (persistence) → Integration (HTTP contract) → E2E). It sits above every `09-N-*.md` module doc and is not nested inside `09-test-file-specification/`, not split by module, and not split by backend/frontend — per the locked structural decision in `09-redesign-implementation-plan.md` item 4.

`02-requirements.md §17` defines a single 13-step journey (register → match → pay → attend class → assess → progress-track → reschedule/make-up/switch-format) as a **requirement**, not an illustration. Before this document existed, zero E2E test files existed anywhere in the spec (Review §3.1) — every layer below whatever was under test was mocked, so nothing proved the seams between services actually work together. An agent could ship a fully green Unit + Integration (HTTP contract) suite while a student registers, gets matched, pays, and never receives a class link, because each piece was verified in isolation and each mock assumed its neighbor behaves.

This document does **not** attempt to cover all 13 steps of §17 in a single test. It defines 16 independently-runnable journeys (E2E-1 through E2E-16), each covering a bounded slice of either §17's student-facing flow or the equivalent Tutor/Admin-facing flow behind it, seeded via direct API calls wherever the journey doesn't need to exercise that specific step's UI. E2E-1 through E2E-8 together cover every step of §17 except step 8 (see §10.20 for why that step is deliberately out of scope for this tier). **E2E-8 (messaging, §10.10) was added in the Issue 8 fix pass** — closing the gap §10.20 (formerly §10.12) previously flagged, where Review §6.3's original 7 named journeys did not include one for in-platform messaging. **E2E-9 through E2E-16 (§10.11–§10.18) were added in the 6th pass (Three-Role Coverage Audit)** — see the update note above — to close the highest-risk cross-service seams that are genuinely Tutor- or Admin-initiated rather than Student-initiated: the alternate matching paths, the rejection/exit/suspension cascades, live pricing/promotion composition, the payment-pause reschedule cycle, the payout pipeline, and the auth/session lifecycle every other journey silently depends on.

Each journey below assumes the lower-tier tests it cites are green. An E2E spec is not a re-test of what a Unit or Integration (persistence) test already proves — it exists specifically to prove the *handoff* across services actually works against a real stack, which no lower tier is positioned to do.

---

### 10.1 Shared E2E Conventions (binding on every spec in this file)

1. **Real, freshly seeded test database per run.** Every spec seeds its own data (directly via Prisma using `00-test-fixtures.md` factories, or via the real API where the journey needs to exercise that call) against a database that is reset before the run — never a shared, drifting database (Review §8 checklist item).
2. **Seed via API, not UI clicks, except where the step under test is the UI itself.** E.g., E2E-1 seeds the VERIFIED tutor directly via `buildTutorProfile({ verificationStatus: 'VERIFIED' })` rather than driving the tutor-registration-and-approval UI, since tutor onboarding is not what E2E-1 exists to prove; it registers the *student* through the real registration page, since that is the journey's actual starting point.
3. **Third parties are faked only via sandbox mode or a dedicated mock HTTP server** running as its own process and replaying realistic webhook/response payloads — never `vi.mock()` or any unit-test mocking utility (`00-agent-rules.md` rule 8). Specifically:
   - **Chapa** (payments): a local mock server that accepts the real checkout-initiation shape and can be told to deliver a signed webhook callback (`SUCCESS`, `FAILED`, or a byte-identical replay) on demand.
   - **Geez SMS / Brevo** (notifications): a local mock server capturing the outbound payload so a test can assert on what was actually sent, not just that `dispatchNotification` was called.
   - **Jitsi** (class delivery): not invoked at all — per Review §6.3's own note, E2E-3 tests the platform's link-lifecycle (a session carries a `jitsiLinkUrl`, gating, and post-session access), not real video-call quality, which is out of scope for this suite (see §10.20).
4. **Test type label:** `E2E (Playwright)`, distinct from all three lower tiers.
5. **Priority (P0/P1/P2)** reflects the build-and-hardening order from Review §9 Phase 4, not a CI-blocking policy — whether a given priority tier blocks merge is a Phase 7 human decision, not something this document prescribes.
6. Every spec below states which lower-tier test rows it assumes are already green. This makes the phased build-order in Review §9 concrete: an E2E spec should not be attempted until its cited rows exist and pass.

---

### 10.2 E2E Test File Map

| # | Spec file | Priority | Journey | §17 steps covered |
|---|---|---|---|---|
| E2E-1 | `tests/e2e/e2e-1-core-conversion-funnel.e2e.spec.ts` | P0 | Register → select needs/format → get matched → admin approves → pay → schedule confirmed | 1–7 |
| E2E-2 | `tests/e2e/e2e-2-guardian-removal-hold.e2e.spec.ts` | P0 | Guardian invites, sole guardian removed mid-flow → `GUARDIAN_REQUIRED_HOLD` surfaces correctly in UI | 1–2 (parent-managed path), ongoing account-integrity concern spanning 13 |
| E2E-3 | `tests/e2e/e2e-3-join-class-access-recording.e2e.spec.ts` | P1 | Join class → access recording & materials post-session | 9–10 |
| E2E-4 | `tests/e2e/e2e-4-assessment-progress.e2e.spec.ts` | P1 | Weekly assessment submission → progress tracking reflects it | 11–12 |
| E2E-5 | `tests/e2e/e2e-5-reschedule-makeup-format-switch.e2e.spec.ts` | P1 | Reschedule / trigger make-up / switch format mid-cohort | 13 (subset) |
| E2E-6 | `tests/e2e/e2e-6-tutor-rejection-resubmission.e2e.spec.ts` | P2 | Tutor verification rejection → resubmission flow | pre-1 (tutor-side onboarding, gates step 3's match pool) |
| E2E-7 | `tests/e2e/e2e-7-refund-request-resolution.e2e.spec.ts` | P2 | Refund request → admin review → resolution, confirms amount never editable client-side | 13 (subset — dispute/refund path) |
| E2E-8 | `tests/e2e/e2e-8-messaging-thread-lifecycle.e2e.spec.ts` | P2 | Matched-pair send/receive → group-thread shared visibility → Admin thread closure | 13 (subset — "message tutor" sub-behavior) |
| E2E-9 | `tests/e2e/e2e-9-path-b-manual-assignment.e2e.spec.ts` | P1 | 1-to-1 search returns zero matches → student triggers "No Exact Match" → Admin manually assigns a tutor → pay → schedule confirmed | 1–7 (Path B variant) |
| E2E-10 | `tests/e2e/e2e-10-path-c-group-auto-match.e2e.spec.ts` | P0 | Two students request a group format → auto-matched into one candidate class → formation window closes at partial size → Admin approves → each student pays independently → shared schedule confirmed | 1–7 (Path C variant) |
| E2E-11 | `tests/e2e/e2e-11-admin-rejection-reroute.e2e.spec.ts` | P1 | Admin rejects a Path A booking (tutor excluded, student re-routed) and a Path C auto-match (clean re-queue, no exclusion) | 5 (rejection branch), pre-6 |
| E2E-12 | `tests/e2e/e2e-12-tutor-exit-suspension-continuity.e2e.spec.ts` | P1 | Admin suspends a tutor with an active group cohort → group kept together and re-matched → affected students' refund entitlement lands as a real row | ongoing account-integrity concern spanning 13 |
| E2E-13 | `tests/e2e/e2e-13-pricing-promotion-live-payment.e2e.spec.ts` | P0 | Admin changes pricing (existing payment unaffected, new booking reflects it) and creates/deactivates a promotion → a real payment composes both correctly, never a client-suppliable amount | 6 (subset — financial-integrity) |
| E2E-14 | `tests/e2e/e2e-14-payment-pause-auto-reschedule.e2e.spec.ts` | P1 | Payment lapses → schedule pauses → payment resumes → affected sessions auto-reschedule with no fault assigned to either party | 13 (subset — billing-lifecycle) |
| E2E-15 | `tests/e2e/e2e-15-tutor-payout-pipeline.e2e.spec.ts` | P1 | Tutor's itemized earnings (full-rate + reduced-make-up-rate) → monthly payout batch generation → Admin marks the batch paid | 13 (subset — Tutor-side financial visibility) |
| E2E-16 | `tests/e2e/e2e-16-auth-session-lifecycle.e2e.spec.ts` | P0 | Login (any role, incl. Admin) → access-token expiry & refresh rotation → reuse-detection revokes the whole session family → password reset revokes every session → login rate-limit enforced | pre-1 (foundational to every other journey) |

No source file has a "Not required" row in this table — all 15 are new files, and this table is this tier's Test File Map for the purposes of `00-agent-rules.md` rule 4.

---

### 10.3 E2E-1 — Core conversion funnel

**Priority:** P0. **Covers §17 steps 1–7.** The core conversion funnel; exercises 3 features' seams (`shared-config` → `accounts-guardianship` → `matching-cohorts` → `payments-earnings` → `class-delivery-library`) in one continuous run.

**Seeding strategy (API-first):**
1. `POST /auth/register/student` — self-registration path (grade 6–12), avoiding the guardian-invite dependency, which is E2E-2's job.
2. `PATCH /students/me/academic-profile` — sets grade, subject(s) of interest, and format preference.
3. Seed one `VERIFIED` tutor directly via `buildTutorProfile({ verificationStatus: 'VERIFIED' })` + `buildAvailabilitySlot()` (bypassing tutor onboarding/approval UI — not this journey's concern; see E2E-6).
4. `GET /matching/tutors/recommendations`, then `POST /matching/select-tutor` — creates the `MatchRequest`/`Cohort`.
5. `POST /admin/matching/:cohortId/approve` as a seeded Admin — moves the cohort to `PENDING_PAYMENT`.
6. `POST /payments/initiate`, then the mock Chapa server delivers a signed `SUCCESS` webhook to `POST /payments/webhook/chapa`, delivered **twice** (byte-identical replay) to confirm idempotency holds under a real HTTP round-trip, not just a mocked one.
7. `GET /cohorts/me` and `GET /sessions` — confirms the schedule was actually generated, not just that the payment record flipped to `SUCCESS`.

**The seam this spec uniquely proves:** `handleChapaWebhook`'s `SUCCESS` path calls into `class-delivery-library`'s `session.service.generateSessionsForCohort` to create the schedule. `backend/9-7-payments-earnings-persistence.md` explicitly states this cross-module call is *not* persistence-tested there — it mocks `generateSessionsForCohort` the same way a true third party is mocked, because `class-delivery-library` has no persistence tier in the Phase 5 priority order. **E2E-1 is therefore the only place in the entire suite that proves a real payment webhook actually produces real, queryable `ScheduledSession` rows.** This is precisely the failure mode Review §3.1 warns about — do not treat this step as redundant with the payments persistence tier.

**Assumed passing lower-tier tests:**
- `backend/9-1-shared-config.md` §9.8 `auth.service.test.ts` (`registerUser`), §9.10 `auth.routes.test.ts`.
- `backend/9-2-accounts-guardianship.md` §9.3–9.4 `studentProfile.service/controller/routes.test.ts`.
- `backend/9-3-matching-cohorts.md` §9.3 `matching.service.test.ts` (`selectTutor`), §9.7 `adminMatching.service.test.ts` (`approveBooking` — including the "does not itself confirm the schedule" case, which this E2E spec is what actually exercises the *other side* of).
- `backend/9-3-matching-cohorts-persistence.md` §9.15 `matching.service.persistence.test.ts` (`selectTutor` real cohort + membership creation) and §9.16 (`formOrJoinCohort` real lifecycle, last-seat race).
- `backend/9-7-payments-earnings.md` §9.4 `payment.service.test.ts` (`initiatePayment`, `handleChapaWebhook` incl. the mocked-Prisma idempotency case), §9.5 `payment.controller/routes.test.ts` (raw-body-aware webhook route).
- `backend/9-7-payments-earnings-persistence.md` §9.24 `payment.service.persistence.test.ts` (real status transitions, idempotent webhook replay against real rows).
- `frontend/9-1-shared-config.md` §9.6 `RegisterPage.test.tsx`.
- `frontend/9-3-matching-cohorts.md` §9.2 `useMatching.test.ts`.
- `frontend/9-7-payments-earnings.md` §9.4 `PaymentPage.test.tsx` (external redirect, post-return reconciliation — this E2E spec is what actually drives the redirect round-trip that test mocks).

---

### 10.4 E2E-2 — Guardian removal → `GUARDIAN_REQUIRED_HOLD`

**Priority:** P0. Security/data-integrity critical; per Review §6.3, currently only unit-tested.

**Seeding strategy:**
1. `POST /auth/register/parent`, `POST /guardianship/students` (grade 1–5, creating the invite), `POST /guardianship/invites/:token/activate` as the student — produces exactly one `ACTIVE`, `MANDATORY_GUARDIAN` relationship (sole guardian).
2. Seed an active, paid `CohortMembership` directly via `buildCohort()` / `buildCohortMembership()` / `buildPayment()` factories (the booking-to-payment flow itself is E2E-1's job; this journey starts from "already enrolled").
3. `PATCH /guardianship/relationships/:id/revoke` as the parent — the sole-guardian path, triggering `handleSoleGuardianRemoval`.

**Assertions:**
- `GET /students/me/profile` shows `accountStatus: GUARDIAN_REQUIRED_HOLD`, and the `ParentStudentRelationship` shows `status: REVOKED` — read back from the real database, not inferred from the mutation's own response.
- The student's real, already-seeded rows (cohort membership, XP ledger, recordings) are untouched — no cascading delete.
- `GuardianSettingsPage` (student- and parent-facing) renders the hold state correctly.
- **(Gap closed) `GET /sessions` is blocked while the student is on hold.** As the student, call `GET /sessions` against the already-seeded, real `ScheduledSession` row(s) from the seeded cohort membership — asserts a real `403` (not an empty list), proving `session.service.ts → assertSessionAccessAllowed` genuinely calls `assertAccountStatusAllowsAccess` end-to-end against a real, persisted `GUARDIAN_REQUIRED_HOLD` row, not a mock told what to return.
- **(Gap closed) A fresh booking attempt is blocked while the student is on hold.** As the student, call `POST /matching/select-tutor` (or, for a caller whose `formatPreference` was previously `ONE_TO_THREE`/`ONE_TO_FIVE`, `POST /matching/group-format`) against a real, verified tutor seeded for this purpose — asserts a real `403`, and that no new `MatchRequest`/`Cohort` row is created in the database as a side effect of the rejected attempt.
- **(Gap closed) The hold lifts and access is restored once a new guardian accepts.** Continuing the same run: `POST /guardianship/students` is not the right call here (the student already exists) — instead `PATCH /guardianship/relationships` re-invites a new guardian for the same student (`POST /guardianship/invites` equivalent per the API spec), the new guardian activates, and a repeat of the two calls above (`GET /sessions`, `POST /matching/select-tutor`) now succeeds against the same, real rows that were blocked moments earlier — proving the gate re-reads `accountStatus` fresh on every call rather than caching a stale "held" result.

**Two upstream spec gaps this journey originally surfaced — now resolved for the first, still open for context on the second:** `backend/9-2-accounts-guardianship-persistence.md` previously noted that no function in Docs 06/08/09-2 named a specific access-gate reading `StudentProfile.accountStatus` at booking- or session-join time. **This is now resolved** (Pre-Implementation Hardening pass, see `09-test-file-specification/phase7-review-signoff.md` for the full list of touched files): the named enforcement points are `studentProfile.service.ts → assertAccountStatusAllowsAccess` (booking half, called by `matching.service.ts`'s three booking-entry functions) and `session.service.ts → assertSessionAccessAllowed` (class-access half). The three assertions above are what this journey previously said it "should attempt... but cannot yet" — they are no longer blocked and are now part of this spec's required coverage, not optional follow-up work.

**Assumed passing lower-tier tests:**
- `backend/9-2-accounts-guardianship.md` §9.6 `guardianship.service.test.ts` (`revokeOrModifyRelationship` / `handleSoleGuardianRemoval`, including the Phase 4 audit-log addition asserting `auditLog.service.record` is called with `action: 'GUARDIAN_REMOVED'`), plus the new `assertAccountStatusAllowsAccess` Unit-tier cases (ACTIVE/hold/pending-activation/not-found, and the three booking-entry call-site cases) added in the same file.
- `backend/9-2-accounts-guardianship-persistence.md` §9.23 `guardianship.service.persistence.test.ts` ("Sole-guardian removal persists `GUARDIAN_REQUIRED_HOLD` for real" — the exact row this E2E spec's core assertion builds directly on top of, promoted from a real-DB read-back to a real full-stack round-trip) and §9.24 (`suspendAccount`'s cascading-effects-on-active-cohorts computation), plus the new "`assertAccountStatusAllowsAccess` — real gate, read against a real row" cases in the same file, which this E2E spec's session/booking assertions are the full-stack promotion of.
- `backend/9-3-matching-cohorts.md` (the new "blocked by the guardian-hold gate" cases on `selectTutor`/`triggerNoExactMatch`/`requestGroupFormat`) and `backend/9-4-class-delivery-library.md` (the new `assertSessionAccessAllowed` Unit-tier cases, including the Tutor/Admin exemption cases this E2E journey does not itself re-exercise).
- `frontend/9-2-accounts-guardianship.md` §9.2 `useGuardianship.test.ts` (full blocks) and §9.3 `GuardianSettingsPage.test.tsx` (dual-mode rendering, revoke confirmation gate).

---

### 10.5 E2E-3 — Join class → access recording & materials

**Priority:** P1. Depends on the class-delivery/session-link seam actually working end-to-end (per Review §6.3's own caveat).

**Seeding strategy:**
1. Seed an active `Cohort`/`CohortMembership` and a `ScheduledSession` (near-future, `status: SCHEDULED`) directly via factories — booking/payment is out of scope here (E2E-1).
2. As the tutor, `POST /sessions/:sessionId/link` — sets `jitsiLinkUrl`; assert the student-facing `SessionCard`'s "Join" control enables only once this call has landed.
3. `POST /recording-consent/acknowledge` for the relevant parties.
4. As the tutor, `POST /sessions/:sessionId/complete`, then `POST /recordings` to register the recording (storage client itself remains mocked — a true third party, per convention 3 above).
5. As the enrolled student, `GET /recordings/:recordingId/signed-url` — succeeds.
6. As a **separately seeded, unrelated student** with zero `CohortMembership` rows for this cohort, the same call is repeated — this is the real-stack promotion of the cross-student IDOR case.

**The seam this spec uniquely proves:** step 6 above is the one case `backend/9-4-class-delivery-library.md` explicitly flags as needing to be tested against "a recording belonging to a *different, real* cohort the caller has no relation to — not just an entirely nonexistent id." A mocked-Prisma Unit test can assert the *code path* taken; only a real request against a real, populated database proves the authorization check actually holds against real, distinguishable rows.

**Assumed passing lower-tier tests:**
- `backend/9-4-class-delivery-library.md` §9.3 `session.service.test.ts`, §9.6 `recordingConsent.service.test.ts`, §9.9 `recording.service.test.ts` (cross-student access denied — IDOR case, the one promoted above), §9.12 `library.service.test.ts`.
- `frontend/9-4-class-delivery-library.md` §9.4 `SessionCard.test.tsx` (join-button gate, link-presence-is-sole-gate case), §9.5 `RecordingConsentPage.test.tsx` / `LibraryPage.test.tsx` / `RecordingPlayer.test.tsx` (signed-URL retention edge case).
- **Out of scope for this spec:** actual Jitsi call quality/connectivity — this suite tests the platform's link lifecycle only, per §10.1 item 3.

---

### 10.6 E2E-4 — Weekly assessment submission → progress tracking

**Priority:** P1.

**Seeding strategy:**
1. Seed an active `Cohort`/`CohortMembership` directly via factories.
2. As the tutor, `POST /assessments`.
3. As the student/parent, `GET /assessments/cohort-membership/:id` — reflects the new entry.
4. Render `ProgressPage` and confirm it surfaces the new assessment.
5. `GET /gamification/xp/me` — if `submitAssessment` is wired to award `ASSESSMENT_COMPLETED` XP, the real ledger reflects it.

**The seam this spec uniquely proves:** `backend/9-6-gamification-engagement.md` explicitly states that "the exact wiring between `class-delivery-library`'s session-completion event and `xp.service.awardXP`... is confirmed at build time, not hard-asserted here beyond `awardXP` itself behaving correctly once called" — i.e., the Unit tier proves `awardXP('ASSESSMENT_COMPLETED')` awards exactly 15 XP *if called*, but no lower tier proves anything actually calls it when a real assessment is submitted. **This is a soft, event-based integration with no FK per `feature-decomposition.md §1.1` — exactly the kind of seam this tier exists for.** If, on writing this spec, the wiring turns out not to exist yet, that is a real finding for Phase 7, not a reason to weaken the assertion (per `00-agent-rules.md` rule 2) — flag it rather than silently asserting only the parts that currently pass.

**Assumed passing lower-tier tests:**
- `backend/9-4-class-delivery-library.md` §9.20 `weeklyAssessment.service.test.ts` (`submitAssessment` — duplicate-for-the-week rejection, non-assigned-tutor rejection; `getAssessmentsForStudent` — IDOR rejection for a non-party caller).
- `backend/9-6-gamification-engagement.md` §9.2 `xp.service.test.ts` (`ASSESSMENT_COMPLETED` awards exactly 15 — the constant this spec's XP assertion, if applicable, depends on).
- `frontend/9-4-class-delivery-library.md` §9.6 `WeeklyAssessmentPage.test.tsx` (client-side duplicate-submission check) and the `ProgressPage.test.tsx` thin-wrapper coverage.

---

### 10.7 E2E-5 — Reschedule / make-up / format switch mid-cohort

**Priority:** P1. Exercises the matching-cohorts and class-delivery state machines under non-happy-path conditions.

**Seeding strategy:**
1. Seed an active `CohortMembership` and a near-future `ScheduledSession`.
2. `POST /reschedule` with ≥12h notice — assert `FREE_RESCHEDULE`, no billing impact, no make-up session consumed.
3. Seed a second session and repeat with <12h notice — assert `SAME_DAY_MISS` classification and that the tutor-caused-miss path (`recordTutorCausedMiss`) creates a linked make-up session flagged `REDUCED_MAKEUP`.
4. `POST /format-switch` — assert the current membership ends, a new `MatchRequest` is created via the correct path/format entry point, and (if remaining paid sessions existed) a `Refund` is created and left `PENDING` — never auto-approved.

**The seam this spec uniquely proves:** the frontend's `classifyReschedule` preview and the backend's authoritative classification can legitimately disagree in the presence of real clock skew between client and server; `frontend/9-4-class-delivery-library.md` calls the "server value wins, not the client preview" case "the single most important integrity test in this feature." A Unit test proves the frontend *would* display whatever the server says; only a real round-trip against a real backend clock proves the server's actual computed classification is what reaches the confirmation screen.

**Assumed passing lower-tier tests:**
- `backend/9-4-class-delivery-library.md` §9.15 `reschedule.service.test.ts` (classification boundary, no-billing-impact case) and §9.17 `sessionMiss.service.test.ts` (`recordTutorCausedMiss`, `REDUCED_MAKEUP` flag; `recordStudentCausedMiss`'s full-rate exclusion).
- `frontend/9-4-class-delivery-library.md` §9.3 `useReschedule.test.ts` / `classifyReschedule.test.ts` (12h boundary inclusive, DST-safety case, "server value wins" case).
- `backend/9-3-matching-cohorts.md` §9.10 `formatSwitch.service.test.ts` (`requestSwitch` — cohort-mates unaffected, refund handoff left `PENDING` per the I1 fix, correct re-entry path per format).
- `backend/9-7-payments-earnings.md` §9.12 `refund.service.test.ts` (the proration math itself — not re-verified here; this E2E spec only proves the handoff and the `PENDING` status reach a real record, per `formatSwitch.service.test.ts`'s own note that "the actual proration-math assertions live in `9-7-payments-earnings.md`").

---

### 10.8 E2E-6 — Tutor verification rejection → resubmission

**Priority:** P2. Trust/admin feature, lower traffic but high dispute-risk.

**Seeding strategy:**
1. `POST /auth/register/tutor`.
2. As Admin, `POST /admin/tutors/:tutorId/reject` with an internal reason.
3. Assert the mock notification server received a generic rejection message with **no** literal reason string in the payload — a genuine cross-service proof of the non-disclosure rule that a mocked `notification.service` cannot fully provide.
4. As the tutor, `PATCH /tutors/me/profile` with corrected information.
5. As the tutor, `POST /tutors/me/resubmit-verification`.
6. Assert the response is `200` with `verificationStatus: PENDING`, and that a subsequent Admin `GET /admin/tutors/pending` includes this tutor again — the read-back proves the row actually left `REJECTED` in the database, not just in the response body of step 5.

**Resolved gap (Issue 2 fix):** `03-usecases.md` (UC-18 alternate flow) states a rejected tutor "may resubmit with corrected information." This previously had no named mechanism to return `verificationStatus` to `PENDING` — profile updates correctly exclude `verificationStatus` from the editable field set (the mass-assignment guard tested in `backend/9-2-accounts-guardianship.md §9.9`), and nothing else re-queued a `REJECTED` tutor for review. Resolved by adding `POST /tutors/me/resubmit-verification` (`06-api/02-accounts-guardianship-api.md`) → `tutorProfile.service.ts → resubmitVerification` (`08-function-level-specification/backend/8-2-accounts-guardianship.md`), which is the sole narrow exception allowing this service to write `verificationStatus`, and resets `verifiedAt`/`verifiedById` to `null` on the transition (`04-database-and-data-model.md`). Steps 5–6 above now assert this end-to-end.

**Assumed passing lower-tier tests:**
- `backend/9-2-accounts-guardianship.md` §9.16 `adminTutorVerification.service.test.ts` (`rejectTutor` — reason persisted internally only, non-disclosure to the tutor-facing notification, Phase 4 audit-log addition for `TUTOR_REJECTED`).
- `frontend/9-2-accounts-guardianship.md` §9.5 `TutorVerificationPage.test.tsx` (required-reason gate on reject).
- `backend/9-1-shared-config.md` §9.11–9.12 `sms.client.test.ts` / `email.client.test.ts` / `notification.service.test.ts` — the dispatch mechanism the mock server in step 3 above intercepts.

---

### 10.9 E2E-7 — Refund request → admin review → resolution

**Priority:** P2. Directly tests the "H4 fix" mass-assignment concern the unit tests already flag — confirms it holds through the whole stack, not just the schema layer.

**Seeding strategy:**
1. As a student/parent/tutor, `POST /complaints` (a billing dispute).
2. As Admin, `GET /admin/disputes`, then `PATCH /admin/disputes/:complaintId` with `resolutionAction: 'REFUND_ISSUED'` and `affectedCohortMembershipId` — and, via a raw HTTP call bypassing the UI entirely, attempt to smuggle a `refundAmount` field into the same request body.
3. Assert the resulting `Refund` row's `amount` is the server-computed proration (via `calculateProration`), never the smuggled figure, and that the refund reaches `status: APPROVED` through `createPendingRefund` → `approveRefund`, not left dangling `PENDING`.

**The seam this spec uniquely proves:** every lower-tier test of the H4 fix — the schema stripping the field, the frontend never rendering an editable amount input, the service calling `createPendingRefund`/`approveRefund` with the right arguments — is verified against a mock or a component in isolation. **Only this spec sends a real HTTP request with the smuggled field all the way through real middleware, real schema validation, and a real service call, and reads back a real, persisted `Refund` row to confirm the exact vulnerability the H4 fix closed cannot be reintroduced by a regression anywhere in that chain** — which is precisely why the review calls this the highest-priority frontend test in the entire `support-trust-admin` feature, promoted here to its full-stack form.

**Assumed passing lower-tier tests:**
- `backend/9-8-support-trust-admin.md` §9.5 `adminDispute.service.test.ts` ("`resolveDisputeSchema` has no `refundAmount` field (H4 fix, mass-assignment guard)"; "REFUND_ISSUED calls the exact same sessions-delivered proration path... — I1 fix", confirming the refund ends `APPROVED` rather than `PENDING`).
- `frontend/9-8-support-trust-admin.md` §9.5 `DisputeCard.test.tsx` ("H4 fix: ... a computed, read-only preview amount — never a free-text amount field"; "`onResolve` payload never includes a raw refund amount").
- `backend/9-7-payments-earnings.md` §9.4 (the payment-side "Mass-assignment guard on amount" row — the sibling guard on the payment-initiation path, confirming the same discipline applies platform-wide, not just in the dispute-resolution path) and §9.12 `refund.service.test.ts`.
- `backend/9-7-payments-earnings-persistence.md` §9.26 `refund.service.persistence.test.ts` (`approveRefund`/`rejectRefund` real state transitions and real claim race) — the persistence-tier proof this E2E spec's final read-back builds directly on top of.

---

### 10.10 E2E-8 — Messaging thread lifecycle

**Priority:** P2. Issue 8 fix — closes the gap this document previously flagged in what is now §10.20: Review §6.3's original 7 named journeys didn't include one for in-platform messaging, even though the feature is otherwise thoroughly Unit- and Integration-tested (`09-test-file-specification/backend|frontend/9-5-messaging.md`).

**Seeding strategy:**
1. Seed a `VERIFIED` tutor (`buildTutorProfile({ verificationStatus: 'VERIFIED' })`) and a `ONE_TO_ONE` `Cohort`/`CohortMembership` pair, both `ACTIVE` (booking/payment is out of scope here — E2E-1's job), giving a confirmed 1-to-1 pairing with messaging enabled.
2. As the student, `POST /messaging/cohorts/:cohortId/messages`. As the tutor, `GET /messaging/cohorts/:cohortId/messages` and assert the message is visible, and that the tutor received a `NEW_MESSAGE` notification (real dispatch through the notification pipeline, not a mocked call).
3. Separately, seed a `ONE_TO_THREE` group `Cohort` with 3 `ACTIVE` `CohortMembership` rows. As one student, send a message; as each of the other two students plus the tutor, `GET` the thread and assert all four participants see the identical `threadId` and the same message — proving there are no private sub-threads within a group thread (UC-58), against real rows rather than the mocked-Prisma version of this same assertion in `backend/9-5-messaging.md`.
4. As Admin, `POST /admin/messaging/threads/:threadId/close` on the 1-to-1 thread from step 2, with a reason.
5. As the student from step 2, attempt `POST /messaging/cohorts/:cohortId/messages` again.
6. Assert `403` with `"This conversation has been closed"`, and that a fresh `GET /admin/messaging/threads/:threadId` read-back confirms `status: CLOSED_BY_ADMIN` — the closure actually persisted, not just the response body of step 4.

**The seam this spec uniquely proves:** every lower-tier messaging assertion — the single-shared-thread invariant, the notification fan-out, the closed-thread block — is verified against mocked Prisma and a mocked `notification.service` in `backend/9-5-messaging.md`. This is the only spec that sends real messages through real middleware and rate-limiting into a real `MessageThread`/`Message` pair, confirms the group-thread visibility invariant across four genuinely different caller sessions rather than one caller's mocked view, and reads back the Admin-closure state from a fresh query rather than trusting the closing call's own return value.

**Assumed passing lower-tier tests:**
- `backend/9-5-messaging.md` §9.x `messaging.service.test.ts` (single-shared-thread/no-private-sub-thread case, `NEW_MESSAGE` dispatch fan-out excluding the sender, the closed-thread `403` branch, the IDOR case for a caller with no relation to the cohort).
- `frontend/9-5-messaging.md` (thread view rendering, message composer rate-limit UI state).
- `backend/9-8-support-trust-admin.md` (Admin thread-closure controller/route wiring, reused here end-to-end rather than re-specified).

---

### 10.11 E2E-9 — Path B: "No Exact Match" → Admin manual assignment

**Priority:** P1. Path B is one of three matching entry points (alongside Path A's direct selection and Path C's group auto-match) and, before this pass, had no E2E coverage at all — only E2E-1's Path A and manual-assignment's Unit/persistence tiers existed.

**Seeding strategy:**
1. Register a 1-to-1 student (`POST /auth/register/student`, grade 6–12) and complete the academic profile, in a subject with **zero** seeded `VERIFIED` tutors — a real empty candidate pool, not a mocked one.
2. As the student, `GET /matching/tutors/recommendations` — assert a real `200` with `recommendations: []` (confirms the zero-results-is-a-200-not-a-404 contract from `matching.service.ts` holds over a real query, not just a mocked one).
3. As the student, `POST /matching/no-exact-match` — assert `status: PENDING_ADMIN_ASSIGNMENT`.
4. *Now* seed a `VERIFIED` tutor eligible for the student's subject/grade (mirroring UC-27's alternate flow: "no suitable tutor currently exists — Admin holds the case open until one becomes available").
5. As Admin, `GET /admin/matching/queue?path=PATH_B` — assert the case is visible with the real, newly-eligible tutor now a candidate.
6. As Admin, `POST /admin/matching/manual-assign` with `{ matchRequestIds: [id], tutorId }` — assert `201`, `Cohort.status: PENDING_PAYMENT` **directly**, with no separate approve call in between (this is itself the approval, per UC-27).
7. As the student, `POST /payments/initiate`, then the mock Chapa server delivers a signed `SUCCESS` webhook.
8. `GET /sessions` — confirms a real, generated schedule.

**The seam this spec uniquely proves:** `manuallyAssignTutor`'s "straight to `PENDING_PAYMENT`, no separate approve step" behavior and the downstream payment→schedule handoff are each proven correct for the Path A *approve* door in E2E-1. This is the only place that same downstream handoff is proven to work correctly when entered through the Path B *manual-assignment* door instead — a different code path into the same `Cohort.status: PENDING_PAYMENT` state, which a green E2E-1 alone gives no evidence about.

**Assumed passing lower-tier tests:**
- `backend/9-3-matching-cohorts.md` §9.3 `matching.service.test.ts` ("Zero results is a 200, not a 404").
- `backend/9-3-matching-cohorts.md` §9.7 `adminMatching.service.test.ts` (`manuallyAssignTutor` — single-request assignment, ineligible-tutor rejection) and §9.8 `adminMatching.controller/routes.test.ts` (Admin-only enforcement).
- `backend/9-3-matching-cohorts-persistence.md` §9.17 `adminMatching.service.persistence.test.ts` (real claim-race guard on `manuallyAssignTutor`).
- `backend/9-7-payments-earnings.md` §9.4 / `backend/9-7-payments-earnings-persistence.md` §9.24 (the same payment/webhook rows E2E-1 already cites — reused here, not re-specified).

---

### 10.12 E2E-10 — Path C: Group auto-match, partial formation, and multi-student payment

**Priority:** P0. `ONE_TO_THREE` and `ONE_TO_FIVE` are two of the platform's three tutoring formats, and before this pass, **neither had ever been driven through a real booking-to-schedule handoff** — E2E-1 covers `ONE_TO_ONE` only. A regression that silently broke group-format payment or scheduling could ship with a fully green Tier 1–3 suite and a fully green E2E-1.

**Seeding strategy:**
1. Register two students (`POST /auth/register/student`, grade 6–12, self-registration path) with matching academic profiles: same subject, same grade band, overlapping schedule, `formatPreference: ONE_TO_THREE`.
2. Seed one `VERIFIED` tutor with a matching ranked subject and enough `AvailabilitySlot` capacity for a 3-seat group.
3. As student 1, `POST /matching/group-format` — assert `201`, `status: SEARCHING`, and — per the no-match-information rule for group formats — that the response contains **no** `tutorId`, match percentage, or any profile field.
4. As student 2, `POST /matching/group-format` for the same subject/grade/schedule — assert the real `formOrJoinCohort` logic joins the **same** candidate `Cohort` student 1's request produced (not a second, duplicate one).
5. The group-formation window is job-driven (`groupFormationWindow.job.ts`) and its interval wrapper is out of scope per the standing convention (§10.20) — call the underlying window-closing logic directly (the same "drive the real function, skip the timer" pattern E2E-1 already uses for the Chapa webhook) against this 2-of-3-seat cohort whose `groupFormationWindowExpiresAt` has been seeded in the past. Assert the real outcome: the class proceeds at size 2, per Section 7's Partial Group Formation rule.
6. `GET /pricing` — assert the per-student rate for `ONE_TO_THREE` is unchanged by the partial size (the platform/tutor absorb the shortfall, never the students).
7. As Admin, `GET /admin/matching/queue?path=PATH_C`, then `POST /admin/matching/:cohortId/approve`.
8. As student 1, `POST /payments/initiate` + a mock Chapa `SUCCESS` webhook; independently repeat for student 2 — **two real, separate payments** for the one shared cohort.
9. `GET /sessions` for both students confirms an identical, real schedule. `GET /cohorts/:cohortId/members` for each student confirms the group-format visibility rule holds against real, multi-student rows: the tutor entry carries only `name`/`photo`, never education/match-percentage/total-students-count.

**The seam this spec uniquely proves:** `formOrJoinCohort`'s join-vs-create branching, the partial-formation billing invariant, and the format-conditional profile-visibility split are each Unit-tested against mocked Prisma and a single mocked cohort. This is the only place two real students' independent payments are proven to compose into one real, shared, correctly-priced, correctly-visible schedule — the actual cross-service promise the group formats make that E2E-1 never touches.

**Assumed passing lower-tier tests:**
- `backend/9-3-matching-cohorts.md` §9.5 `cohort.service.test.ts` (`formOrJoinCohort` join/create branching, partial-formation, no-pricing-field-write boundary) and §9.3 `matching.service.test.ts` (group-format visibility split, cross-referenced with §9.5's `getCohortMembers` cases).
- `backend/9-3-matching-cohorts.md` §9.7 `adminMatching.service.test.ts` (`approveBooking` for Path C).
- `backend/9-3-matching-cohorts-persistence.md` §9.16 `cohort.service.persistence.test.ts` (real last-seat-race guard on `formOrJoinCohort`).
- `backend/9-7-payments-earnings.md` §9.9 `pricing.service.test.ts` ("Partially-formed group still bills at the full per-student rate").
- `frontend/9-3-matching-cohorts.md` (group-format request UI, no-match-information rendering rule).

---

### 10.13 E2E-11 — Admin rejects a booking → student reroute

**Priority:** P1. UC-32's two branches (Path A tutor-exclusion vs. Path C clean re-queue) are Unit-tested against mocked Prisma; before this pass, no E2E spec ever exercised an Admin *rejection*, only approvals (E2E-1, E2E-9, E2E-10).

**Seeding strategy — Path A branch:**
1. Real Path A flow through `POST /matching/select-tutor` for a real `VERIFIED` tutor — cohort reaches `PENDING_ADMIN_APPROVAL`.
2. As Admin, `POST /admin/matching/:cohortId/reject` with `internalReason: "Tutor's availability conflicted with a higher-priority booking"`.
3. Assert (real DB read-back, not the rejection call's own response): `Cohort.status: CANCELLED, endedReason: ADMIN_REJECTED`; a real `TutorExclusion(studentId, tutorId)` row exists.
4. As the student, `GET /matching/tutors/recommendations` again — the previously-rejected tutor is genuinely **absent** from the real query result, not merely flagged client-side.
5. As the student, `GET /notifications` — the message reads only the generic "assignment could not be confirmed"; `internalReason`'s exact text never appears anywhere the student can read it.
6. Confirm no `Payment` row was ever created for the cancelled cohort.

**Seeding strategy — Path C branch:**
7. Real Path C flow (per E2E-10, steps 1–3) to a candidate `ONE_TO_THREE` cohort pending approval.
8. As Admin, `POST /admin/matching/:cohortId/reject`.
9. Assert **no** `TutorExclusion` row is written (the Path A/Path C distinction is a real branch, not just a documented one) and that `GET /matching/requests/me` for the student shows a genuine `status: SEARCHING` again — a real re-queue, confirmed by a fresh read, not inferred from the rejection call succeeding.

**The seam this spec uniquely proves:** `rejectCohort`'s branching (write-an-exclusion vs. clean-re-queue) and the generic-reason notification rule are Unit-tested with the exclusion/re-queue outcome asserted against a mock told what to report. This is the only place a real subsequent `GET /matching/tutors/recommendations` query is proven to actually honor a real `TutorExclusion` row, and the only place the internal rejection reason is proven genuinely absent from what a real student session can read back — not merely absent from a payload a test constructed.

**Assumed passing lower-tier tests:**
- `backend/9-3-matching-cohorts.md` §9.5 `cohort.service.test.ts` (`approveCohort`/`rejectCohort` — both branches, internal-reason non-disclosure) and §9.3 `matching.service.test.ts` (`recommendTutorsWithMatchPercent`/`searchOneToOneTutors` — the `TutorExclusion` filter, cross-referenced).
- `backend/9-1-shared-config.md` §9.12 `notification.service.test.ts` (generic-message dispatch).
- `frontend/9-3-matching-cohorts.md` (rejection state rendering, re-entry into the recommendations view).

---

### 10.14 E2E-12 — Tutor exit / Admin suspension → group continuity and refund cascade

**Priority:** P1. UC-33/UC-34's promise — "no student is left un-tutored without a refund entitlement" — spans three modules (`accounts-guardianship`'s suspension, `matching-cohorts`'s continuity re-match, `payments-earnings`'s proration) each of which mocks the other two. Before this pass, nothing proved the full cascade against real rows.

**Seeding strategy:**
1. Seed a `VERIFIED` tutor with one `ACTIVE` `ONE_TO_FIVE` `Cohort`, 5 real `ACTIVE` `CohortMembership` rows, and a real history of already-delivered `ScheduledSession`s (so `totalSessionsBilled`/sessions-remaining are meaningful, non-zero real numbers).
2. As Admin, `POST /admin/people/:userId/suspend` with `{ reason: "Policy violation", restrictionType: "SUSPENDED" }`.
3. Assert the response's `affectedCohortIds` includes this cohort, computed against real active `CohortMembership` rows (not a stubbed list).
4. Real DB read-back: `Cohort.status: ENDED, endedReason: TUTOR_SUSPENDED`; all 5 `CohortMembership` rows `ENDED`; exactly 5 fresh `MatchRequest` rows exist, each `path: PATH_C, status: PENDING_ADMIN_ASSIGNMENT` — the `tutorExitContinuity` handoff, confirmed against real rows rather than a spied-on call.
5. As Admin, `GET /admin/matching/queue?path=PATH_C` — the 5 pending-assignment cases are visible.
6. Seed one real, eligible replacement `VERIFIED` tutor with capacity for all 5; as Admin, `POST /admin/matching/manual-assign` with all 5 `matchRequestIds` — assert exactly one new `Cohort` forms and all 5 students land in it (the group is kept together, per the M3 worked example, not split when one eligible tutor exists).
7. As Admin, `GET /admin/refunds?status=PENDING` — a real `Refund` row exists for each affected student, `amount` computed against the **old** cohort's real `totalSessionsBilled`, read back from the database rather than asserted from the suspension call's own response.
8. As each affected student, `GET /cohorts/me` confirms the new cohort/tutor, and `GET /sessions` confirms no session silently disappeared from the schedule during the transition.

**The seam this spec uniquely proves:** `suspendAccount`'s `affectedCohortIds` computation, `tutorExitContinuity`'s fresh-`MatchRequest`-spawn, `manuallyAssignTutor`'s group-keep-together behavior, and the refund-proration handoff are each Unit- or persistence-tested in isolation, with every neighboring module mocked. This is the only place a single, real Admin action is proven to cascade — in one continuous run against one real database — through cohort termination, fresh-request spawn, re-assignment, and a correctly-computed refund, which is the actual end-to-end guarantee UC-33/34 make and that no lower tier is positioned to check.

**Assumed passing lower-tier tests:**
- `backend/9-2-accounts-guardianship.md` §9.18 `adminPeople.service.test.ts` (`suspendAccount`) and `backend/9-2-accounts-guardianship-persistence.md` §9.24 `adminPeople.service.persistence.test.ts` (real `affectedCohortIds` computation, durable audit entry).
- `backend/9-3-matching-cohorts.md` §9.5 `cohort.service.test.ts` (`tutorExitContinuity`) and §9.7 `adminMatching.service.test.ts` (`manuallyAssignTutor`/`manuallyAssembleGroup`, M3 worked example).
- `backend/9-7-payments-earnings.md` §9.12 `refund.service.test.ts` (proration formula) and `backend/9-7-payments-earnings-persistence.md` §9.26 `refund.service.persistence.test.ts`.

---

### 10.15 E2E-13 — Live Admin pricing/promotion change composes correctly into a real payment

**Priority:** P0. Same class of financial-integrity risk E2E-7 (the H4 mass-assignment fix) exists to guard — but E2E-7 proves a client can't smuggle its own amount; nothing previously proved that a **live, Admin-issued** pricing change and a **live, Admin-issued** promotion, both in effect simultaneously, compose into the correct server-computed amount on a real subsequent payment, or that neither ever touches a payment already completed.

**Seeding strategy:**
1. Complete one real Path A booking-to-payment flow (per E2E-1) under the currently active `ONE_TO_ONE` pricing — call this **payment A**.
2. As Admin, `PUT /admin/pricing/ONE_TO_ONE` with a new rate — assert `201`, a new active `PricingConfig` row.
3. Re-fetch payment A (`GET /payments/history` or a direct read) — assert its `amount` is **unchanged**, never retroactively altered by the pricing change that happened after it was created.
4. As Admin, `POST /admin/promotions` creates an active promo code (e.g. 10% `PERCENT` off).
5. A second, fresh student completes a real Path A booking through to `POST /payments/initiate`, supplying the promo code. Assert the resulting real `Payment.amount` reflects the **new** pricing composed with the promo discount, computed entirely server-side — proven against a real, persisted row, not a client-echoed value.
6. As Admin, `PATCH /admin/promotions/:id` with `{ isActive: false }`. A third student attempts the same code on a fresh `POST /payments/initiate` — assert a real `400` ("Invalid or expired promotion code"), confirming deactivation takes effect immediately against a live subsequent request, not a value cached from step 4.

**The seam this spec uniquely proves:** `pricing.service.ts`'s "next booking only, never retroactive" invariant and `promotion.service.ts`'s discount math are each proven correct in isolation, against configs/codes the test itself constructs and controls. This is the only place a **live** pricing change and a **live** promotion are both in effect at once and a real payment is proven to compose them correctly — exactly the class of compounding financial logic that two independently-green Unit suites, each testing one mechanism at a time, can never catch a regression in.

**Assumed passing lower-tier tests:**
- `backend/9-7-payments-earnings.md` §9.9 `pricing.service.test.ts` ("Change applies to the next new booking only") and §9.19 `promotion.service.test.ts` (`applyToPayment` PERCENT/FIXED_ETB math, expired/inactive-code rejection).
- `backend/9-7-payments-earnings.md` §9.4 `payment.service.test.ts` (mass-assignment guard on `amount` — the sibling guard this spec confirms holds even with two live financial mutations in play, not just one).
- `backend/9-7-payments-earnings-persistence.md` §9.25 `pricing.service.persistence.test.ts` (real atomic deactivate-then-activate swap under concurrency).

---

### 10.16 E2E-14 — Payment lapse → schedule pause → auto-reschedule on resume

**Priority:** P1. UC-39–41's "a payment lapse is never mischaracterized as either party's fault" guarantee crosses `payments-earnings` (the pause) and `class-delivery-library` (the affected sessions); each module's Unit tier mocks the other's rows.

**Seeding strategy:**
1. Seed a real `ACTIVE` `CohortMembership` whose billing due date has already passed with no successful payment. Since the reminder-to-pause trigger is job-driven (`paymentReminder.job.ts`) and its interval wrapper is out of scope per the standing convention, call `pauseForNonPayment(cohortMembershipId)` directly — the same "drive the real function, skip the timer" pattern E2E-1 already uses for the Chapa webhook.
2. Seed 2 real `ScheduledSession`s falling inside the resulting pause window.
3. As the student, `GET /sessions` — the 2 sessions show the paused status; `GET /payment-pause/status` shows a real, open pause.
4. As the student, `POST /payments/initiate`, then the mock Chapa server delivers a signed `SUCCESS` webhook — a real payment completes.
5. Real DB read-back: `PaymentPause.endedAt` is now set; both sessions carry a **new** `scheduledStart` at the tutor's next available slot (not their original time); **no** `SessionMiss` row exists for either session; **no** `Refund` and **no** `TutorEarning` row exists for either session — confirmed by their real, continued absence, not by a mock never having been called.

**The seam this spec uniquely proves:** `rescheduleSessionsDuringPause` is Unit-tested against mocked sessions and a mocked Prisma client that reports whatever the test tells it to. This is the only place a real payment webhook is proven to trigger a real, persisted reschedule of real session rows, and the only place the "neither party's fault" guarantee is checked by confirming the real, continued absence of `SessionMiss`/`Refund`/`TutorEarning` rows after the fact — not merely that certain functions weren't called on a mock that was never wired to a real session in the first place.

**Assumed passing lower-tier tests:**
- `backend/9-7-payments-earnings.md` §9.6 `paymentPause.service.test.ts` (`pauseForNonPayment`/`resumeOnPayment`, `rescheduleSessionsDuringPause` — no-miss/no-refund/no-earnings assertions) and §9.7 `paymentPause.controller/routes.test.ts`.
- `backend/9-7-payments-earnings.md` §9.4 `payment.service.test.ts` (`handleChapaWebhook`) and `backend/9-7-payments-earnings-persistence.md` §9.24 (the same real webhook rows E2E-1 already cites).

---

### 10.17 E2E-15 — Tutor earnings ledger → monthly payout batch → Admin marks paid

**Priority:** P1. UC-70/UC-82's "full, itemized visibility, payout happening automatically, Admin retains oversight" promise had no E2E coverage — `earning.service.ts` and `payout.service.ts` are each Unit- and persistence-tested against seeded/mocked ledger rows in isolation from each other and from the Admin-facing read.

**Seeding strategy:**
1. Seed a `VERIFIED` tutor with several real, completed sessions across one billing period producing real `TutorEarning` rows: some `rateType: FULL`, at least one `REDUCED_MAKEUP` (reusing E2E-5's `recordTutorCausedMiss` mechanism to produce a genuine reduced-rate make-up session rather than inserting the rate type directly).
2. As the tutor, `GET /tutors/me/earnings` — assert the reduced-rate session is itemized distinctly from full-rate ones, and `upcomingPayout` previews a real, correctly-summed estimate for the period.
3. Call `generateMonthlyPayouts(periodStart, periodEnd)` directly (the job-interval wrapper is out of scope per the standing convention — the same pattern as E2E-1's webhook and E2E-14's pause trigger).
4. As Admin, `GET /admin/payouts` — assert exactly one new `Payout(status: PENDING)` for the tutor, `totalAmount` equal to the real sum of that tutor's unpaid earnings for the period.
5. Re-run `generateMonthlyPayouts` for the identical period — assert **no** duplicate `Payout` row is created (the real double-counting guard, not a mocked "second call does nothing" assertion).
6. As Admin, `POST /admin/payouts/:payoutId/mark-paid` — assert `status: PAID`, `paidAt` set; a second `mark-paid` call on the same payout returns a real `409`.
7. As the tutor, re-fetch `GET /tutors/me/earnings` — the payout now reflects as paid, no longer `upcoming`.

**The seam this spec uniquely proves:** `earning.service.ts` and `payout.service.ts` are each proven correct against ledger rows the test itself seeds or mocks. This is the only place a tutor's real, itemized earnings — crossing `class-delivery-library`'s session/miss classification into `payments-earnings`' rate logic — are proven to flow correctly into a real payout batch that an Admin can genuinely review and mark paid, end to end, rather than each half of that pipeline separately passing against data the other half never actually produced.

**Assumed passing lower-tier tests:**
- `backend/9-7-payments-earnings.md` §9.14 `earning.service.test.ts`, §9.16 `payout.service.test.ts` (batching, double-counting guard, `markPaid`/`adminAdjust`) and §9.17 `payout.controller/routes.test.ts` (Admin-only enforcement).
- `backend/9-7-payments-earnings-persistence.md` §9.27 `earning.service.persistence.test.ts` (real unique-constraint backstop) and §9.28 `payout.service.persistence.test.ts` (real ledger aggregation, real double-counting guard).
- `backend/9-4-class-delivery-library.md` (`REDUCED_MAKEUP` session-miss classification — reused from E2E-5, not re-specified here).

---

### 10.18 E2E-16 — Auth/session lifecycle: login, refresh rotation, reuse detection, password-reset revocation

**Priority:** P0. Every journey in this document, and every Admin/Tutor/Student action on the platform, depends on `POST /auth/login` and the refresh-token chain behind it — yet before this pass, **not one E2E spec exercised authentication itself**; every other journey simply assumed a valid session existed. The persistence tier (`9-1-shared-config-persistence.md`) already proves rotation and reuse-detection are correct against a real database, but explicitly by calling `auth.service.ts` functions directly — it does not, and says it cannot, prove the same guarantees hold once real middleware, the real rate-limiter, and real request/response handling sit in front of that service.

**Seeding strategy:**
1. Seed a `Student` user; `POST /auth/login` with valid credentials — assert a real `accessToken`/`refreshToken` pair and a real `RefreshToken` row.
2. `POST /auth/refresh` with the real `refreshToken` — assert real rotation: a fresh query shows the original row `revokedAt`-set with `replacedByTokenId` pointing at a new row sharing the same `familyId`; the new pair works against a protected route (`GET /students/me/profile` succeeds).
3. Reuse the **original**, now-rotated `refreshToken` in a second `POST /auth/refresh` — assert a real `401`, and a fresh query confirms **every** `RefreshToken` row sharing that `familyId` (including the still-active rotated child from step 2) is now revoked — the real HTTP round-trip triggers full-family revocation, not just the function called in isolation.
4. Seed a second, independent active session for the same student (fresh `login`). `POST /auth/forgot-password`, then `POST /auth/reset-password` with a valid code — assert `200`, and that this second session's `refreshToken` is now genuinely revoked as a result of the reset: a subsequent real `POST /auth/refresh` with it returns `401`.
5. Rate-limit check: `POST /auth/login` 6 times with a wrong password for the same identifier inside the 15-minute window (NFR-013) — assert the 6th attempt returns a real `429` from the actual rate-limiter middleware, not a mocked counter.
6. Seed an `Admin` user; `POST /auth/login` — assert the same endpoint issues a working admin session: an admin-only route (`GET /admin/matching/queue`) succeeds with it, while the Student session from step 1 gets a real `403` on the identical route.

**The seam this spec uniquely proves:** refresh rotation, reuse-detection, and rate-limiting are each already proven correct against a real database, but by calling service functions directly. This is the only place the **entire** HTTP path — real middleware, the real rate-limiter, real header/cookie handling — is driven end-to-end through genuine requests, which matters precisely because `9-1-shared-config-persistence.md` itself notes it cannot prove a real request even reaches the service correctly without that chain in front of it. Every other E2E spec in this document implicitly trusts that chain works; this is the one that actually checks.

**Assumed passing lower-tier tests:**
- `backend/9-1-shared-config.md` §9.8 `auth.service.test.ts` (`login`/`refreshAccessToken`/`logout`/`logoutAll`, rotation, reuse-detection-revokes-the-family, `resetPassword` revokes all tokens) and §9.9–9.10 `auth.controller/routes.test.ts`.
- `backend/9-1-shared-config-persistence.md` §9.23 `auth.service.persistence.test.ts` (real atomic rotation, real concurrent-refresh race, real 3-generation reuse-detection chain, durable `LOGIN_FAILED_THRESHOLD` audit entry).
- `02-requirements.md` NFR-013 (rate-limit figures) — the same source of truth `backend/9-1-shared-config.md §9.19` and `frontend/9-1-shared-config.md §9.10` already reconcile against (per the Phase 7 review's checklist item 1).

---

### 10.19 Coverage Honesty Check (per PR Steward, at review time)

- [ ] All 16 spec files listed in §10.2 exist and were run against a freshly seeded database, not a cached/shared one — listed here by filename, checked against the actual `tests/e2e/` directory, not against this document's own memory of what should exist (per `00-agent-rules.md` rule 11).
- [ ] Every "assumed passing lower-tier test" cited in §10.3–10.18 was independently verified as currently passing before this E2E spec was authored — an E2E spec written against a currently-failing lower-tier assumption is not evidence of anything.
- [ ] No spec uses `vi.mock()` or an equivalent unit-mocking utility anywhere — only the sandbox/mock-server strategy in §10.1 item 3. This applies to E2E-16 in particular: it is tempting to mock the rate-limiter or the token store to make the auth spec faster, and doing so would defeat the entire point of the spec.
- [ ] §10.4's booking/session-join enforcement point is resolved and its three new assertions (`GET /sessions` blocked, `POST /matching/select-tutor` blocked, access restored after a new guardian activates) are present and passing against real rows, not stubbed. §10.8's tutor-resubmission mechanism (Issue 2) is now resolved — verify its two new assertions (steps 5–6) pass against real rows, not stubbed.
- [ ] §10.10's group-thread visibility assertion (Issue 8) was checked across four genuinely different caller sessions in the same run, not inferred from one caller's result — matching the standard `backend/9-5-messaging.md` already set at the mocked-Prisma tier.
- [ ] E2E-1's and E2E-7's assertions read back from a fresh database query, not from the mutating call's own return value, matching the standard the Phase 5 persistence tier already set.
- [ ] **E2E-10's partial-group-formation step (§10.12, step 5) calls the real window-closing logic directly, not the `setInterval` wrapper, and the resulting 2-of-3 class is verified as actually billing the full per-student rate against a real `GET /pricing` read** — not asserted from the seeding script's own assumptions about what the rate should be.
- [ ] **E2E-13's pricing-then-promotion sequence (§10.15) is run as a single continuous spec, not as two separately-passing specs** — the whole point is that both mutations are live at once when the final payment is computed; splitting them into isolated runs would silently re-create the exact "each piece verified alone, neighbor's mock assumed correct" gap this document exists to close.
- [ ] **E2E-16's reuse-detection assertion (§10.18, step 3) confirms every row sharing the token's `familyId` is revoked via a fresh `findMany`, not just the presented row** — matching the standard `backend/9-1-shared-config-persistence.md §9.23`'s three-generation-chain case already sets, now proven through real middleware rather than a direct service call.
- [ ] **E2E-16's rate-limit case (§10.18, step 5) is confirmed to hit the actual configured `rateLimiter.middleware.ts`, not a lower test-environment override** — a rate limit relaxed for faster CI runs would make this assertion pass trivially regardless of whether the real limit is enforced in production.

---

### 10.20 Out of Scope for This Tier (and why)

- **§17 step 8 ("Receive reminder") has no dedicated E2E spec.** This step is entirely job-driven (`src/jobs/classReminder.job.ts`) and notification-dispatch-only; the underlying dispatch logic is already Unit-tested in `backend/9-1-shared-config.md §9.12` (`notification.service.test.ts`), and the job-interval wrapper itself is excluded from testing platform-wide per the standing convention noted in every backend Test File Map (§9.1) that lists a `src/jobs/*.ts` row. Adding an E2E spec here would only re-prove that a `setInterval` wrapper calls a function that is already tested — no cross-service seam is at risk. (E2E-10, E2E-14, and E2E-15 each drive a different job's *underlying logic* directly for the same reason a timer itself is never worth E2E-testing — see each spec's seeding strategy.)
- **Real Jitsi call quality, connectivity, and video/audio behavior** — E2E-3 tests the platform's session-link lifecycle only (§10.5), consistent with `backend/9-4-class-delivery-library.md`'s own posture that third-party call quality is a manual/separately-tracked concern, the same treatment `chapa.client.ts`/`sms.client.ts`/`email.client.ts` get in their respective Unit docs.
- **Sustained multi-client load and performance testing** — every concurrency case in this document (E2E-1's webhook, E2E-2's removal, E2E-3's IDOR check, E2E-13's concurrent pricing swap) is a correctness check against a real single-run database, not a load test. `backend/9-3-matching-cohorts-persistence.md §9.19`, `backend/9-6-gamification-engagement.md §9.11`, and `backend/9-1-shared-config-persistence.md §9.25` already flag genuine sustained-load testing as out of scope for the persistence tier for the same reason; it remains out of scope here.
- **Visual regression / cross-browser matrix testing** — not requested by Review §6.3 and not implied by §17; each spec runs against a single reference browser configuration, consistent with the "critical user journeys only" scope Review §6.3 explicitly sets ("do not attempt to cover all 13 steps... in one test" applies equally to not attempting a full browser matrix on top of that).
- **Admin CRUD screens with no cross-service seam** were deliberately **not** promoted to E2E in the 6th pass, even though several (UC-76 people management, UC-77 tutor badges, UC-79 subjects/grades, UC-84 Library/recording-access management, UC-85 platform policies, UC-89 notifications/announcements, UC-91 platform statistics) have no E2E spec at all. Each of these is a single-module read/write already covered at the Unit and Integration (HTTP contract) tiers (e.g. `adminPeople.service.test.ts`, `adminAnnouncement.service.test.ts`, `policy.service.test.ts`), and none of them hands off state to a second service the way, say, a suspension (E2E-12) or a pricing change (E2E-13) does. Promoting every Admin screen to E2E regardless of whether it crosses a service boundary would repeat exactly the "cover everything, prove nothing extra" mistake §10.0 already warns against — see `backend/9-2-accounts-guardianship.md §9.18`'s own note that `GET /admin/people`'s free-text search surface is instead covered by a dedicated `injection.test.ts` at the Unit tier, which is the right tier for that particular risk.
- **General support/complaint intake and account-settings management** (UC-66 complaint filing outside the billing-dispute path already covered by E2E-7, UC-67 general support contact, UC-68 account/privacy settings, UC-69 emergency contact channel, UC-73 Tutor problem-reporting) — each is a straightforward create/read against `support-trust-admin` or `shared-config` with no multi-service handoff beyond the standard notification dispatch already proven generically in E2E-1/E2E-6/E2E-8's notification assertions. Adding a dedicated E2E spec per intake form would test the same "a `POST` writes a row and a notification fires" pattern eight more times.
- **Gamification read surfaces with no financial or access-control consequence** (UC-63 XP/badges/streaks beyond the single award check already in E2E-4, UC-64 leaderboards, UC-65 challenges) — these are display/aggregation logic over `gamification-engagement`'s own tables with no downstream effect on payments, access, or scheduling; a wrong leaderboard rank is a display bug, not a seam-failure of the kind this tier exists to catch.
- **Recording-compliance escalation (UC-48) and "keep permanently" marking (UC-49)** — both are single-module state changes on an already-generated `Recording` row (`class-delivery-library`), fully covered by that module's Unit/Integration (HTTP contract) tiers; neither hands off to a second service the way E2E-3's join-class-to-recording-access journey does.
