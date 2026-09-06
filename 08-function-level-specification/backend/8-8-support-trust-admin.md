## Project: AKEWTutor — Backend Function-Level Spec: Support, Trust & Admin Reporting
**Conventions:** see `00-api-conventions.md` §0.1–0.7. **API reference:** `08-support-trust-admin-api.md`. **Folder/file reference:** `05a-backend-structure.md` §8.

**Owns:** ComplaintReport. **Depends on:** `shared-config`, `messaging`, `class-delivery-library` (hard, via `relatedThreadId`/`relatedSessionId`). **Soft-integrates with:** `payments-earnings` (a refund can be issued as a resolution action, no FK), `accounts-guardianship` (a tutor suspension can be issued as a resolution action, no FK). This is the one feature that is soft-coupled to almost everything by design — an integration surface, not a domain of its own data (Doc 07 §1.1).

---

### src/schemas/complaint.schema.ts (new)

| Schema | Shape |
|---|---|
| createComplaintSchema | `z.object({ body: z.object({ category: z.enum(['SESSION_ISSUE','TUTOR_CONDUCT','PAYMENT_ISSUE','MESSAGE_ISSUE','OTHER']), description: z.string().min(10).max(2000), relatedCohortId: z.string().uuid().optional(), relatedSessionId: z.string().uuid().optional(), relatedPaymentId: z.string().uuid().optional() }).refine(b => b.category === 'OTHER' \|\| b.relatedCohortId \|\| b.relatedSessionId \|\| b.relatedPaymentId, "A complaint must reference a session, payment, or cohort unless filed as a general (OTHER) report") })` |
| resolveDisputeSchema | `z.object({ body: z.object({ status: z.enum(['UNDER_REVIEW','RESOLVED','DISMISSED']), resolutionAction: z.enum(['NO_ACTION','WARNING_ISSUED','REFUND_ISSUED','TUTOR_SUSPENDED']).optional(), resolutionNotes: z.string().min(1), refundAmount: z.string().optional() }).refine(b => b.status !== 'RESOLVED' || !!b.resolutionAction, "A resolution action is required to resolve a complaint").refine(b => b.resolutionAction !== 'REFUND_ISSUED' || !!b.refundAmount, "A refund amount is required for this resolution action") })` |

### src/services/complaint.service.ts (new)

#### createComplaint

| Field | Detail |
|---|---|
| Signature | `createComplaint(reporterId: string, reporterRole: Role, input): Promise<ComplaintReportDTO>` |
| Purpose | File a complaint, claim, or report against a session, tutor, payment, or message thread — the sole entry point into the Admin dispute-review queue; every `ComplaintReport` is investigated via `GET /admin/disputes/:complaintId` regardless of category. |
| Throws | `ApiError(403, "You can only file a complaint about your own sessions, payments, or cohorts")` — the referenced `relatedCohortId`/`relatedSessionId`/`relatedPaymentId` doesn't belong to the caller. (The "must reference something unless OTHER" rule is schema-enforced above, but the ownership check is a business rule requiring a DB lookup, so it lives here.) |
| Side effects | Resolves `relatedThreadId` automatically from `relatedCohortId`/`relatedSessionId` if a `MessageThread` exists for that cohort, so the Admin dispute view can link to it without the reporter needing to know a thread ID exists. Writes a `Notification` to the Admin queue via `notification.service.ts`, through the same pipeline as any other notification type. Does **not** itself close, mute, or otherwise affect the referenced thread or session — that is a separate, explicit Admin action. |

Test file: `tests/services/complaint.service.test.ts` — includes the cross-user ownership-check 403 case.

#### listForUser / getForReporter / getSupportContactInfo

| Field | Detail |
|---|---|
| Signature | `listForUser(reporterId: string, status?, page?, limit?): Promise<PaginatedComplaintDTO>` · `getForReporter(reporterId: string, complaintId: string): Promise<ComplaintReportDTO>` · `getSupportContactInfo(): Promise<SupportContactDTO>` |
| Throws | (getForReporter) `ApiError(403, "Not authorized to view this complaint")` — caller is not the original reporter. `ApiError(404, "Complaint not found")`. |
| Edge cases | (listForUser) No complaints filed yet → `complaints: []`, not an error. (getForReporter) Never exposes internal Admin resolution notes — the reporter-facing DTO omits `resolutionNotes`/`resolvedById` entirely, which is a different, smaller shape than the Admin-facing detail read in `adminDispute.service.ts` below, not the same object with fields redacted at the controller. (getSupportContactInfo) A static, Admin-configurable read — serves both the general support channel (UC-67) and the emergency contact channel (UC-69) from the same published details, since support is deliberately manual with no automated routing (Doc 01 §1.7 Assumption #4). |

Test file: `tests/services/complaint.service.test.ts`

### src/controllers/complaint.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| fileComplaint | `complaintService.createComplaint(req.user.id, req.user.role, req.body)` | 201 |
| listMyComplaints | `complaintService.listForUser(req.user.id, req.query.status, req.query.page, req.query.limit)` | 200 |
| getMyComplaint | `complaintService.getForReporter(req.user.id, req.params.complaintId)` | 200 |
| getSupportContact | `complaintService.getSupportContactInfo()` | 200 |

### src/routes/complaint.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | /complaints | `authMiddleware, validate(createComplaintSchema)` | fileComplaint |
| GET | /complaints/me | `authMiddleware` | listMyComplaints |
| GET | /complaints/:complaintId | `authMiddleware` | getMyComplaint |
| GET | /support/contact | — | getSupportContact |

Mounted at the root — `/complaints/*` and `/support/contact` are both handled by this router per `00-api-conventions.md` §0.7's feature-to-router mapping, since both are owned by `complaint.controller.ts`/`complaint.service.ts` in Doc 05a.

---

### src/services/adminDispute.service.ts (new)

#### listDisputeQueue

| Field | Detail |
|---|---|
| Signature | `listDisputeQueue(status?, category?, page?, limit?): Promise<PaginatedAdminComplaintDTO>` |
| Edge cases | An empty queue returns `complaints: []` with `200` — the normal steady state, not an error condition. |

Test file: `tests/services/adminDispute.service.test.ts`

#### getDisputeForReview

| Field | Detail |
|---|---|
| Signature | `getDisputeForReview(complaintId: string): Promise<AdminComplaintDetailDTO>` |
| Purpose | Full complaint detail for investigation, including any linked `relatedThreadId`/`relatedSessionId`/`relatedPaymentId`. Returns *references* to related resources rather than embedding them — Admin follows up with `GET /admin/messaging/threads/:threadId` (`messaging` feature) or `GET /sessions/:sessionId` (`class-delivery-library` feature) as needed, keeping each feature's data ownership intact rather than this service duplicating their data. |

Test file: `tests/services/adminDispute.service.test.ts`

#### resolveDispute

| Field | Detail |
|---|---|
| Signature | `resolveDispute(complaintId: string, adminId: string, input: { status, resolutionAction?, resolutionNotes, refundAmount? }): Promise<DisputeResolutionResultDTO>` |
| Purpose | Resolve or dismiss a complaint. The one function in this feature that reaches into another feature's data as a side effect: `resolutionAction: REFUND_ISSUED` calls `payments-earnings`' `refund.service.ts → approveRefund`-equivalent path with `refundAmount`; `resolutionAction: TUTOR_SUSPENDED` calls `accounts-guardianship`'s `adminPeople.service.ts → suspendAccount`. A re-matching resolution is **not** modeled as a `resolutionAction` value — it is initiated separately via `matching-cohorts`' own endpoints, with the complaint simply marked `RESOLVED` once that's done externally; this function never contains re-matching logic itself. |
| Throws | `ApiError(400, "A resolution action is required to resolve a complaint")` — redundant with the schema refine, retained as the authoritative rule. `ApiError(400, "A refund amount is required for this resolution action")` — same. `ApiError(409, "This complaint has already been closed")` — already `RESOLVED`/`DISMISSED`. |
| Side effects | Sets `status`, `resolutionAction`, `resolutionNotes` (internal only — never disclosed to the reporter verbatim), `resolvedById`, `resolvedAt`. Sends a `Notification(type: COMPLAINT_RESOLVED)` to the reporter once `status` moves to `RESOLVED` or `DISMISSED`, through the standard pipeline. |

Test file: `tests/services/adminDispute.service.test.ts` — includes both cross-feature side-effect paths (refund call, suspension call) and the already-closed 409 case.

### src/controllers/adminDispute.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listQueue | `adminDisputeService.listDisputeQueue(req.query.status, req.query.category, req.query.page, req.query.limit)` | 200 |
| getDisputeDetail | `adminDisputeService.getDisputeForReview(req.params.complaintId)` | 200 |
| resolveDispute | `adminDisputeService.resolveDispute(req.params.complaintId, req.user.id, req.body)` | 200 |

### src/routes/adminDispute.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware, requireRole('ADMIN')` | listQueue |
| GET | /:complaintId | `authMiddleware, requireRole('ADMIN')` | getDisputeDetail |
| PATCH | /:complaintId | `authMiddleware, requireRole('ADMIN'), validate(resolveDisputeSchema)` | resolveDispute |

Mounted at `/admin/disputes`.

---

### src/services/adminReporting.service.ts (new)

#### aggregatePlatformHealth

| Field | Detail |
|---|---|
| Signature | `aggregatePlatformHealth(): Promise<PlatformHealthDTO>` |
| Purpose | Read-only, computed-on-request summary combining two Doc 03 use cases: stale-approval/support-management figures (`overdueMatchApprovals`, `recordingComplianceEscalations`) and platform-wide activity/tutor-performance stats. Not a stored `Report` entity — there is no corresponding `POST`. |
| Side effects | Aggregates counts across other features' already-job-maintained flags: `openDisputes` (from `ComplaintReport` in this feature), `overdueMatchApprovals` (from `matching-cohorts`' `staleApproval.job.ts`-maintained flags), `recordingComplianceEscalations` (from `class-delivery-library`'s `recordingMissingCheck.job.ts`-maintained `recordingStatus`), `pendingPayoutBatches` (from `payments-earnings`' `Payout` rows). This function only counts — it never mutates any of those underlying flags, and never re-derives staleness/escalation logic that already lives in each owning feature's own service. |

Test file: `tests/services/adminReporting.service.test.ts`

#### getActivityHistory / getTutorPerformanceHistory

| Field | Detail |
|---|---|
| Signature | `getActivityHistory(page?, limit?, dateRange?): Promise<PaginatedActivityDTO>` · `getTutorPerformanceHistory(tutorId?: string, page?, limit?): Promise<PaginatedTutorPerformanceDTO>` |
| Purpose | Backing UC-91's platform-wide activity stats and tutor performance/badge oversight — these read across `class-delivery-library` (sessions delivered, misses), `gamification-engagement` (badges awarded), and `accounts-guardianship` (verification/suspension history) without owning any of that data directly, consistent with this feature being "an integration surface, not a domain of its own data." |

Test file: `tests/services/adminReporting.service.test.ts`

### src/controllers/adminReporting.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getStats | `adminReportingService.aggregatePlatformHealth()` | 200 |
| getActivity | `adminReportingService.getActivityHistory(req.query.page, req.query.limit, req.query.dateRange)` | 200 |
| getTutorPerformance | `adminReportingService.getTutorPerformanceHistory(req.query.tutorId, req.query.page, req.query.limit)` | 200 |

### src/routes/adminReporting.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /platform-health | `authMiddleware, requireRole('ADMIN')` | getStats |
| GET | /activity | `authMiddleware, requireRole('ADMIN')` | getActivity |
| GET | /tutor-performance | `authMiddleware, requireRole('ADMIN')` | getTutorPerformance |

Mounted at `/admin/reports`. Note: only `GET /admin/reports/platform-health` is named in `08-support-trust-admin-api.md` §8.1 — `/activity` and `/tutor-performance` are implied by Doc 05a's service function list (`getActivityHistory`, `getTutorPerformanceHistory`) and UC-91's stated scope, but aren't independently documented as separate endpoints in the current API spec. Worth confirming with whoever owns Doc 06 whether these should be split into their own documented routes or folded into `platform-health`'s response before implementation — flagged here rather than silently assumed either way.

---

**This is the last of the 8 backend feature files** — no jobs are owned by this feature (its cross-cutting nature means it only ever reads job-maintained state from other features, per the notes above).

**Next:** proceed to → [09. Test File Specification]
