## Project: AKEWTutor — Backend Test Documentation: Support, Trust & Admin Reporting
**Links back to:** [05a. Backend Folder & File Structure §8], [8-8. Function-Level Spec: Support, Trust & Admin Reporting]
**Conventions:** see `00-api-conventions.md` §0.1–0.7.

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** ComplaintReport. **Depends on:** `shared-config`, `messaging`, `class-delivery-library` (hard, via `relatedThreadId`/`relatedSessionId`). **Soft-integrates with:** `payments-earnings` (refund resolution action, no FK), `accounts-guardianship` (suspension resolution action, no FK). This is the one feature soft-coupled to almost everything by design — an integration surface, not a domain of its own data (Feature Decomposition §1.1).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| complaint.schema.ts | FR-AD-017, complaint intake shape | — |
| complaint.service.ts | FR-AD-017, FR-MS-004 (linkage), FR-PB-006 (support contact), FR-NO-* (Admin queue notify) | NFR-009 |
| adminDispute.service.ts | FR-AD-012, FR-AD-017, FR-MK-003 (tutor-suspension resolution path), FR-SP-048/Section 13 (refund resolution path) | NFR-009 |
| adminReporting.service.ts | FR-AD-021, FR-AD-022, FR-AD-005, FR-AD-014, FR-AD-020 | — |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/complaint.schema.ts | tests/schemas/complaint.schema.test.ts | Unit | ☐ |
| src/services/complaint.service.ts | tests/services/complaint.service.test.ts | Unit (mocked Prisma, mocked `messaging.service.ts` thread lookup, mocked `notification.service.ts`) | ☐ |
| src/controllers/complaint.controller.ts | tests/controllers/complaint.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/complaint.routes.ts | tests/routes/complaint.routes.test.ts | Integration (supertest) | ☐ |
| src/services/adminDispute.service.ts | tests/services/adminDispute.service.test.ts | Unit (mocked Prisma, mocked `refund.service.calculateProration`/`approveRefund`, mocked `adminPeople.service.suspendAccount`, mocked `notification.service.ts`) | ☐ |
| src/controllers/adminDispute.controller.ts | tests/controllers/adminDispute.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminDispute.routes.ts | tests/routes/adminDispute.routes.test.ts | Integration (supertest) | ☐ |
| src/services/adminReporting.service.ts | tests/services/adminReporting.service.test.ts | Unit (mocked Prisma, reading across feature tables) | ☐ |
| src/controllers/adminReporting.controller.ts | tests/controllers/adminReporting.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminReporting.routes.ts | tests/routes/adminReporting.routes.test.ts | Integration (supertest) | ☐ |

---

### 9.2 Test Case Detail — complaint.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requires a related entity unless category is OTHER | — | parse `{ body: { category: "TUTOR_CONDUCT", description: "10+ chars here", relatedCohortId: undefined, relatedSessionId: undefined, relatedPaymentId: undefined } }` | fails the `.refine` — "A complaint must reference a session, payment, or cohort unless filed as a general (OTHER) report" |
| OTHER category needs no related entity | — | parse `{ body: { category: "OTHER", description: "10+ chars here" } }` | passes |
| description length bounds | — | parse with a 9-char description, then a 2001-char description | both fail — `min(10)`/`max(2000)` |
| resolveDisputeSchema requires a resolutionAction when resolving | — | parse `{ body: { status: "RESOLVED", resolutionNotes: "note" } }` (no `resolutionAction`) | fails the `.refine` — "A resolution action is required to resolve a complaint" |
| resolveDisputeSchema requires affectedCohortMembershipId for REFUND_ISSUED | — | parse `{ body: { status: "RESOLVED", resolutionAction: "REFUND_ISSUED", resolutionNotes: "note" } }` (no `affectedCohortMembershipId`) | fails the `.refine` — "affectedCohortMembershipId is required for this resolution action" |
| resolveDisputeSchema has no refundAmount field (H4 fix, mass-assignment guard) | — | parse `{ body: { status: "RESOLVED", resolutionAction: "REFUND_ISSUED", affectedCohortMembershipId: uuid, resolutionNotes: "note", refundAmount: "9999.99" } }` | the unknown `refundAmount` key is stripped — confirms a client cannot supply its own refund amount; it is always server-computed via `calculateProration` (OWASP A08:2021 — mass assignment onto a monetary field, the exact H4 fix this schema encodes) |
| DISMISSED doesn't require a resolutionAction | — | parse `{ body: { status: "DISMISSED", resolutionNotes: "note" } }` | passes |

---

### 9.3 Test Case Detail — complaint.service.test.ts

FRs: FR-AD-017, FR-MS-004 (linkage), FR-PB-006. NFRs: NFR-009. **OWASP: A01:2021 – Broken Access Control (ownership check on `createComplaint`, reporter-only visibility on reads).**

#### createComplaint

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Files a complaint referencing the caller's own cohort/session/payment | mock the referenced `relatedSessionId` belongs to the caller | call `createComplaint(reporterId, reporterRole, input)` | resolves `ComplaintReportDTO` |
| Rejects a reference to another user's resource (IDOR) | mock `relatedSessionId` belongs to a different, real cohort/session the caller has no relation to | call `createComplaint(reporterId, reporterRole, input)` | throws `ApiError(403, "You can only file a complaint about your own sessions, payments, or cohorts")` — tested against a genuinely different, existing resource, not merely a nonexistent id |
| Auto-resolves relatedThreadId from the cohort/session | mock a `MessageThread` exists for the referenced cohort | call `createComplaint(...)` | the created `ComplaintReport.relatedThreadId` is set automatically — the reporter never has to supply a thread id |
| No thread exists for the reference | mock no `MessageThread` for the referenced cohort (e.g. a 1-to-1 that never exchanged messages) | call `createComplaint(...)` | `relatedThreadId` remains `null` — not an error |
| Notifies the Admin queue | spy on `notification.service.dispatchNotification` | call `createComplaint(...)` | called once, targeting the Admin queue, through the standard notification pipeline |
| Never affects the referenced thread/session itself | spy on `messaging.service.ts`/`session.service.ts` mutation calls | call `createComplaint(...)` | assert neither was called to close, mute, or otherwise modify the referenced resource — filing a complaint is purely additive until an explicit Admin resolution action |
| OTHER category with no related entity | mock `category: 'OTHER'`, no related ids | call `createComplaint(...)` | resolves successfully — schema-level rule already covers this (9.2), service doesn't re-block it |

#### listForUser / getForReporter / getSupportContactInfo

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Lists the caller's own complaints, paginated | mock several `ComplaintReport` rows for the reporter | call `listForUser(reporterId, undefined, 1, 20)` | resolves `PaginatedComplaintDTO` scoped to `reporterId` |
| No complaints yet | mock an empty result | call `listForUser(...)` | resolves `{ complaints: [], page, limit, total: 0 }`, not an error |
| Filters by status | mock a mix of `UNDER_REVIEW`/`RESOLVED` complaints | call `listForUser(reporterId, 'RESOLVED', 1, 20)` | resolves only the `RESOLVED` ones |
| getForReporter returns the reporter's own complaint | mock caller is the original reporter | call `getForReporter(reporterId, complaintId)` | resolves `ComplaintReportDTO` |
| getForReporter rejects a non-reporter caller (IDOR) | mock caller is not the original reporter | call `getForReporter(otherUserId, complaintId)` | throws `ApiError(403, "Not authorized to view this complaint")` |
| getForReporter 404s on an unknown complaint | mock lookup → `null` | call `getForReporter(reporterId, unknownId)` | throws `ApiError(404, "Complaint not found")` |
| getForReporter never exposes internal resolution notes | mock a `RESOLVED` complaint with `resolutionNotes`/`resolvedById` populated | call `getForReporter(reporterId, complaintId)` | the resolved DTO omits `resolutionNotes` and `resolvedById` entirely — this is a genuinely smaller/different shape than the Admin-facing detail read, not the same object with fields redacted after the fact at the controller |
| getSupportContactInfo is a static published read | mock Admin-configured contact details | call `getSupportContactInfo()` | resolves the same `SupportContactDTO` regardless of caller — serves both the general support channel (UC-67) and the emergency contact channel (UC-69) from the same published data, since support is deliberately manual with no automated routing (Doc 01 §1.7 Assumption #4) |

---

### 9.4 Test Case Detail — complaint.controller.test.ts / complaint.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| fileComplaint/listMyComplaints/getMyComplaint require auth; getSupportContact doesn't | no Authorization header | request all four endpoints | the first three `401`; `GET /support/contact` `200` |
| fileComplaint validates body | valid auth token | request with a body missing a related entity and `category !== 'OTHER'` | rejected by `validate(createComplaintSchema)`, controller never called |
| fileComplaint forwards req.user.id/role, never a client-suppliable reporter | mock service | call controller | `createComplaint` called with `req.user.id`/`req.user.role` |
| listMyComplaints/getMyComplaint always scoped to req.user.id | mock service | call each handler | the underlying service call receives `req.user.id`, never a client-suppliable reporter id from query/params |

---

### 9.5 Test Case Detail — adminDispute.service.test.ts

FRs: FR-AD-012, FR-AD-017, FR-MK-003, FR-SP-048. **OWASP: A01:2021 – Broken Access Control (Admin-only surface), A04:2021 – Insecure Design (the H4-fixed server-computed-refund path is the central integrity control in this file).**

#### listDisputeQueue

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Lists queued complaints, filterable by status/category | mock a mix of complaints | call `listDisputeQueue('UNDER_REVIEW', 'TUTOR_CONDUCT', 1, 20)` | resolves only matching rows |
| Empty queue is a normal 200 state | mock no complaints match | call `listDisputeQueue(...)` | resolves `{ complaints: [], ... }`, not an error — the normal steady state |

#### getDisputeForReview

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns full detail including linked references | mock a complaint with `relatedThreadId`/`relatedSessionId`/`relatedPaymentId` set | call `getDisputeForReview(complaintId)` | resolves `AdminComplaintDetailDTO` containing those ids as *references* |
| Does not embed another feature's data | inspect the resolved DTO | call `getDisputeForReview(complaintId)` | the thread/session content itself is not duplicated inline — Admin follows up via `GET /admin/messaging/threads/:threadId` or `GET /sessions/:sessionId` separately, keeping each feature's data ownership intact |
| 404 on an unknown complaint | mock lookup → `null` | call `getDisputeForReview(unknownId)` | throws common `ApiError(404, ...)` |

#### resolveDispute

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Dismisses a complaint | mock `status: UNDER_REVIEW` | call `resolveDispute(complaintId, adminId, { status: 'DISMISSED', resolutionNotes: 'no violation found' })` | sets `status: DISMISSED`, `resolvedById`, `resolvedAt`; sends `Notification(type: COMPLAINT_RESOLVED)` to the reporter |
| Resolves with NO_ACTION | — | call `resolveDispute(complaintId, adminId, { status: 'RESOLVED', resolutionAction: 'NO_ACTION', resolutionNotes: '...' })` | sets `status: RESOLVED`; no downstream feature call made |
| REFUND_ISSUED calls the exact same sessions-delivered proration path as any other refund reason (H4 fix) | mock `affectedCohortMembershipId` has an active, paid billing cycle | call `resolveDispute(complaintId, adminId, { status: 'RESOLVED', resolutionAction: 'REFUND_ISSUED', affectedCohortMembershipId, resolutionNotes: '...' })` | spy on `refund.service.calculateProration` — called with `(payment.id, 'ADMIN_DISPUTE_RESOLUTION')`, then `approveRefund` — confirms this is never a bare client-supplied amount, the exact H4-fixed integrity guarantee |
| REFUND_ISSUED rejects a membership with no paid cycle to prorate | mock `affectedCohortMembershipId` has no `SUCCESS` `Payment` in its current cycle | call `resolveDispute(...)` | throws `ApiError(400, "affectedCohortMembershipId does not have an active, paid billing cycle to prorate")` |
| TUTOR_SUSPENDED calls accounts-guardianship's suspension | mock a valid resolution input | call `resolveDispute(complaintId, adminId, { status: 'RESOLVED', resolutionAction: 'TUTOR_SUSPENDED', resolutionNotes: '...' })` | spy on `adminPeople.service.suspendAccount` — called with the tutor's id |
| A re-matching resolution is never modeled here | — | (documentation-level check) confirm `resolutionAction`'s enum has no `RE_MATCH`-style value | no such value exists — re-matching is initiated separately via `matching-cohorts`' own endpoints; the complaint is simply marked `RESOLVED` once that's done externally, per Doc 8-8's explicit note |
| resolutionNotes never disclosed to the reporter verbatim | mock a resolution with sensitive internal notes | call `resolveDispute(...)`, then call `getForReporter` (9.3) for the same complaint | the reporter-facing read never surfaces `resolutionNotes` — cross-checked against 9.3's own assertion, not merely assumed here |
| Already-closed complaint rejected | mock `status` already `RESOLVED`/`DISMISSED` | call `resolveDispute(complaintId, adminId, ...)` again | throws `ApiError(409, "This complaint has already been closed")` |
| Missing resolutionAction on a RESOLVED submission rejected at the service layer too | bypass the schema, simulate an internal call | call `resolveDispute(complaintId, adminId, { status: 'RESOLVED', resolutionNotes: '...' })` | throws `ApiError(400, "A resolution action is required to resolve a complaint")` — redundant with the schema, retained as authoritative |

---

### 9.6 Test Case Detail — adminDispute.controller.test.ts / adminDispute.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All three routes require Admin | no Authorization header, then a Parent token | request `GET /admin/disputes`, `GET /admin/disputes/:id`, `PATCH /admin/disputes/:id` | (a) `401`; (b) `403` for each |
| resolveDispute forwards req.user.id as adminId | mock service | call controller | `resolveDispute` called with `req.user.id`, never a client-suppliable admin id |
| resolveDispute validates body | valid Admin token | request `PATCH /admin/disputes/:id` with `{ status: "RESOLVED", resolutionNotes: "..." }` (no `resolutionAction`) | rejected by `validate(resolveDisputeSchema)` |

---

### 9.7 Test Case Detail — adminReporting.service.test.ts

FRs: FR-AD-021, FR-AD-022, FR-AD-005, FR-AD-014, FR-AD-020. **OWASP: A01:2021 – Broken Access Control (Admin-only surface).**

#### aggregatePlatformHealth

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Aggregates counts across all owning features | mock `ComplaintReport`, `matching-cohorts`' stale-approval flags, `class-delivery-library`'s recording-compliance flags, `payments-earnings`' `Payout` rows | call `aggregatePlatformHealth()` | resolves `PlatformHealthDTO` with `openDisputes`, `overdueMatchApprovals`, `recordingComplianceEscalations`, `pendingPayoutBatches` all populated from those sources |
| Never mutates any of the underlying flags | spy on all Prisma write calls | call `aggregatePlatformHealth()` | assert zero `update`/`create`/`delete` calls — this function only counts, it's read-only end to end |
| Never re-derives another feature's staleness/escalation logic | inspect the query | call `aggregatePlatformHealth()` | the counts are read directly from each owning feature's already-job-maintained flag columns (e.g. `Cohort.adminOverdueNotifiedAt`, `ScheduledSession.recordingStatus`) — no independent re-computation of the 48h/5-day or 2h/24h windows happens in this file, which already live in `staleApproval.job.ts`/`recordingMissingCheck.job.ts` |

#### getActivityHistory / getTutorPerformanceHistory

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns paginated activity across the documented source tables | mock rows across `Cohort`, `Payment`, `ComplaintReport`, `TutorProfile`, `Refund`, `Payout` | call `getActivityHistory(1, 20, dateRange)` | resolves `PaginatedActivityDTO` reflecting all six sources (H5 follow-up fix — now independently documented) |
| Filters by dateRange | mock activity spanning multiple periods | call `getActivityHistory(1, 20, { from, to })` | resolves only rows inside the range |
| getTutorPerformanceHistory reads across features without owning the data | mock sessions-delivered/misses (`class-delivery-library`), badges (`gamification-engagement`), verification/suspension history and `uniqueStudentsTaught` (`accounts-guardianship`) | call `getTutorPerformanceHistory(tutorId, 1, 20)` | resolves `PaginatedTutorPerformanceDTO` combining all three without this service owning any of that underlying data directly |
| getTutorPerformanceHistory supports filtering/sorting | mock several tutors | call `getTutorPerformanceHistory(undefined, 1, 20, sortBy, verificationStatus)` | resolves filtered/sorted per the given params |
| H5 fix — now backs a documented endpoint | — | (documentation-level check) confirm `GET /admin/reports/tutor-performance` is specified in `08-support-trust-admin-api.md` | documented — no longer an implied-only route |

---

### 9.8 Test Case Detail — adminReporting.controller.test.ts / adminReporting.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All three routes require Admin | no Authorization header, then a Tutor token | request `GET /admin/reports/platform-health`, `/activity`, `/tutor-performance` | `401` then `403` for each |
| getActivity validates dateRange/eventType | valid Admin token | request `GET /admin/reports/activity?dateRange=NOT_AN_ENUM` | rejected — `400` per the Zod enum shape (Doc 06 §8.1) |
| Each handler delegates its query params unchanged | mock service | call each controller with representative query params | the service receives exactly the parsed `req.query` values, no silent transformation |

---

### 9.9 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `createComplaint`'s cross-user ownership 403 is tested against a real, different cohort/session/payment the caller has no relation to — not just a nonexistent id, which would only exercise a 404-adjacent path rather than the ownership check itself.
- [ ] `getForReporter`'s omission of `resolutionNotes`/`resolvedById` is tested as an actual assertion on the resolved object's keys — not inferred from "the test didn't check for it," which would pass even if a future change accidentally leaked the field.
- [ ] `resolveDispute`'s `REFUND_ISSUED` path is tested by spying on `refund.service.calculateProration`/`approveRefund` and asserting the exact arguments passed — not merely that *some* refund-shaped object appears in the response, which would miss a regression back to a bare client-supplied amount (the exact bug H4 fixed).
- [ ] The already-closed `409` case is tested against both prior states (`RESOLVED` and `DISMISSED`) independently, not only one of the two.
- [ ] `aggregatePlatformHealth`'s read-only guarantee is tested by spying on all Prisma write methods and asserting zero calls — not inferred from the return value looking correct.
- [ ] The re-matching-is-never-a-resolutionAction check is tested against the actual enum definition in the schema, not merely asserted in prose in this doc.
- [ ] `getTutorPerformanceHistory`'s cross-feature reads (sessions/misses, badges, verification/suspension, `uniqueStudentsTaught`) are each independently present in the mocked fixture and independently asserted in the resolved DTO — a test that only checks one of the four could pass while silently missing a broken join to another.

---

### 9.10 Out of Scope for Automated Testing (and why)

- **The actual re-matching workflow triggered externally after a dispute** — `resolveDispute` deliberately contains no re-matching logic itself (Doc 8-8's explicit note); the `matching-cohorts` endpoints that perform the re-match are covered in that feature's own test doc (9-3), not duplicated here.
- **The real downstream Chapa refund call / real account-suspension side effects** — both `refund.service.approveRefund` and `adminPeople.service.suspendAccount` are called from here as already-tested units (9-7, 9-2 respectively); this doc verifies only that `adminDispute.service.ts` calls them correctly, not their own internal correctness a second time.
- **Manual support-channel responsiveness** (phone/Telegram staffing, response-time SLAs) — `getSupportContactInfo` is tested as a static, Admin-configured data read; the human process behind FR-PB-006/UC-67/UC-69 is an operational concern, not a unit-test concern.
- **Real-time dashboard refresh of `aggregatePlatformHealth`** — the backend contract is a computed-on-request `GET`; any polling/refresh cadence on the Admin dashboard is a frontend concern (Doc 07/08 frontend specs).
- **Exhaustive cross-feature join correctness beyond the documented six source tables for `getActivityHistory`** — this doc confirms the six named tables (`Cohort`, `Payment`, `ComplaintReport`, `TutorProfile`, `Refund`, `Payout`) are each represented; a broader data-warehouse-style audit of every possible activity type is out of scope for a unit-test suite.

---

**This is the last of the 8 backend feature test-file specification docs.** No jobs are owned by this feature (its cross-cutting nature means it only ever reads job-maintained state from other features, per 9.7's notes above).

**Next:** proceed to → [09. Test File Specification: Frontend] (if planned) or back to [08. Function-Level Specification] for implementation.
