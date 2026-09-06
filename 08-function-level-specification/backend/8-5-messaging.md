## Project: AKEWTutor — Backend Function-Level Spec: In-Platform Messaging
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.4 (thread archiving is job-driven, exposed here only as `status: "ARCHIVED"`). **API reference:** `05-messaging-api.md`. **Folder/file reference:** `05a-backend-structure.md` §5.

**Owns:** MessageThread, Message. **Depends on:** `matching-cohorts` (hard) — a `MessageThread` is 1:1 with a `Cohort`.

---

### src/schemas/messaging.schema.ts (new)

| Schema | Shape |
|---|---|
| sendMessageSchema | `z.object({ params: z.object({ cohortId: z.string().uuid() }), body: z.object({ body: z.string().min(1).max(2000) }) })` — text-only; no attachment/media field exists anywhere in this schema, matching the API spec's explicit "no attachment/media fields exist" note. |

### src/services/messaging.service.ts (new)

#### getThreadForCohort

| Field | Detail |
|---|---|
| Signature | `getThreadForCohort(callerId: string, cohortId: string): Promise<MessageThreadDTO>` |
| Purpose | Thread metadata — a private pair thread for 1-to-1, a single shared thread for 1-to-3/1-to-5 (no private sub-threads within a group thread). |
| Throws | `ApiError(403, "Messaging is not available for this cohort")` — cohort not yet confirmed/paid, or caller no longer an active member. |
| Output | `participantCount` = tutor + every currently active `CohortMembership` on the cohort — always `2` for 1-to-1. |

Test file: `tests/services/messaging.service.test.ts` — includes cohort-shared-thread vs. 1-to-1-private-thread cases.

#### listMessages

| Field | Detail |
|---|---|
| Signature | `listMessages(callerId: string, cohortId: string, page, limit): Promise<PaginatedMessageDTO>` |
| Throws | Same membership rule as `getThreadForCohort`. |
| Edge cases | A thread archived 90 days after the cohort ended still returns its full history here — archiving removes it from the *active* thread list on the client only; it is never a deletion (UC-59). |

Test file: `tests/services/messaging.service.test.ts`

#### sendMessage

| Field | Detail |
|---|---|
| Signature | `sendMessage(callerId: string, cohortId: string, body: string): Promise<MessageDTO>` |
| Throws | `ApiError(403, "Messaging is not available for this cohort")` — same membership rule. `ApiError(403, "This conversation has been closed")` — thread `status: CLOSED_BY_ADMIN`. |
| Side effects | Creates the `Message` row; calls `notification.service.ts → dispatchNotification` (`type: NEW_MESSAGE`) for every other active participant, through the same pipeline as any other notification type (FR-NO-011). |

Test file: `tests/services/messaging.service.test.ts` — includes the closed-thread-blocks-send case.

### src/controllers/messaging.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getThread | `messagingService.getThreadForCohort(req.user.id, req.params.cohortId)` | 200 |
| listMessages | `messagingService.listMessages(req.user.id, req.params.cohortId, req.query.page, req.query.limit)` | 200 |
| sendMessage | `messagingService.sendMessage(req.user.id, req.params.cohortId, req.body.body)` | 201 |

### src/routes/messaging.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /cohorts/:cohortId/thread | `authMiddleware` | getThread |
| GET | /cohorts/:cohortId/messages | `authMiddleware` | listMessages |
| POST | /cohorts/:cohortId/messages | `authMiddleware, validate(sendMessageSchema)` | sendMessage |

Mounted at `/messaging`.

---

### src/services/adminMessaging.service.ts (new)

#### viewThreadForDispute

| Field | Detail |
|---|---|
| Signature | `viewThreadForDispute(threadId: string, page, limit): Promise<AdminThreadDetailDTO>` |
| Purpose | Admin investigates a thread for dispute review (UC-60, FR-MS-004) — this is the connection point `support-trust-admin`'s `adminDispute.service.ts` uses when a complaint references a `MessageThread`. |
| Throws | Common `404` if the thread doesn't exist. |

Test file: `tests/services/adminMessaging.service.test.ts`

#### closeThread

| Field | Detail |
|---|---|
| Signature | `closeThread(threadId: string, adminId: string, reason: string): Promise<{ id, status: 'CLOSED_BY_ADMIN', closedById, closedAt }>` |
| Purpose | Closes/reports a thread violating platform rules. |
| Side effects | Once closed, `sendMessage` (above) returns `403` for that cohort's thread going forward — enforced by `sendMessage`'s own status check, not duplicated logic here. |

Test file: `tests/services/adminMessaging.service.test.ts`

### src/controllers/adminMessaging.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| viewThread | `adminMessagingService.viewThreadForDispute(req.params.threadId, req.query.page, req.query.limit)` | 200 |
| closeThread | `adminMessagingService.closeThread(req.params.threadId, req.user.id, req.body.reason)` | 200 |

### src/routes/adminMessaging.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /threads/:threadId | `authMiddleware, requireRole('ADMIN')` | viewThread |
| POST | /threads/:threadId/close | `authMiddleware, requireRole('ADMIN')` | closeThread |

Mounted at `/admin/messaging`.

---

### src/jobs/archiveMessageThreads.job.ts (new)

| Field | Detail |
|---|---|
| Trigger | Interval scan for `MessageThread`s whose `Cohort.endedAt` is 90+ days in the past and `status` is not already `ARCHIVED`. |
| Effect | Sets `MessageThread.status: ARCHIVED` via `messaging.service.ts`. History remains fully readable through `listMessages` — archiving is a list-visibility change on the client, never a deletion. |
| Idempotency | Filtered by `status !== ARCHIVED`, so a repeat run is a no-op for already-archived threads. |

---

**Next:** proceed to → [8-6. Backend: Gamification & Engagement]
