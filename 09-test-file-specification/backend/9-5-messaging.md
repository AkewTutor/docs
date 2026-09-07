## Project: AKEWTutor — Backend Test Documentation: In-Platform Messaging
**Links back to:** [05a. Backend Folder & File Structure §5], [8-5. Function-Level Spec: In-Platform Messaging]
**Conventions:** see `00-api-conventions.md` §0.1–0.7, esp. §0.4 (thread archiving is job-driven, exposed here only as `status: "ARCHIVED"`).

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** MessageThread, Message. **Depends on:** `matching-cohorts` (hard) — a `MessageThread` is 1:1 with a `Cohort`.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| messaging.schema.ts | FR-MS-001 (text-only shape) | — |
| messaging.service.ts | FR-MS-001–003, FR-SC-004, FR-NO-011 | NFR-009 |
| adminMessaging.service.ts | FR-MS-004, FR-AD-017 | NFR-009 |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/messaging.schema.ts | tests/schemas/messaging.schema.test.ts | Unit | ☐ |
| src/services/messaging.service.ts | tests/services/messaging.service.test.ts | Unit (mocked Prisma, mocked `notification.service.dispatchNotification`) | ☐ |
| src/controllers/messaging.controller.ts | tests/controllers/messaging.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/messaging.routes.ts | tests/routes/messaging.routes.test.ts | Integration (supertest) | ☐ |
| src/services/adminMessaging.service.ts | tests/services/adminMessaging.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/adminMessaging.controller.ts | tests/controllers/adminMessaging.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminMessaging.routes.ts | tests/routes/adminMessaging.routes.test.ts | Integration (supertest) | ☐ |
| src/jobs/archiveMessageThreads.job.ts | — | Underlying logic covered via `messaging.service.test.ts`'s archived-thread-still-readable case; the interval-registration wrapper itself is excluded per the standing convention | — |

---

### 9.2 Test Case Detail — messaging.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| sendMessageSchema rejects an empty body | — | parse `{ params: { cohortId: uuid }, body: { body: "" } }` | fails — `min(1)` |
| sendMessageSchema rejects a body over 2000 chars | — | parse with `body.body` = 2001-char string | fails — `max(2000)` |
| sendMessageSchema accepts a valid 1–2000 char body | — | parse with `body.body` = `"Running 5 minutes late, sorry!"` | passes |
| sendMessageSchema rejects a non-UUID cohortId path param | — | parse `{ params: { cohortId: "not-a-uuid" }, body: { body: "hi" } } ` | fails |
| No attachment/media field exists on the schema (mass-assignment guard) | — | parse `{ params: { cohortId: uuid }, body: { body: "hi", attachmentUrl: "https://evil.example/x" } }` | the unknown `attachmentUrl` key is stripped (default Zod object behavior) — confirms a client cannot smuggle a media field past validation that the DTO/DB was never designed to carry (OWASP A08:2021 — mass assignment), matching the API spec's explicit "no attachment/media fields exist" note |

---

### 9.3 Test Case Detail — messaging.service.test.ts

FRs: FR-MS-001–003, FR-SC-004, FR-NO-011. NFRs: NFR-009. **OWASP: A01:2021 – Broken Access Control (membership scoping is this file's central risk).**

#### getThreadForCohort

| Case | Setup | Action | Expected result |
|---|---|---|---|
| 1-to-1 cohort returns a private pair thread | mock an `ACTIVE` `CohortMembership` for the caller on a `ONE_TO_ONE` cohort | call `getThreadForCohort(callerId, cohortId)` | resolves `MessageThreadDTO` with `participantCount: 2` |
| Group cohort returns a single shared thread with correct participantCount | mock a `ONE_TO_THREE` cohort with 3 `ACTIVE` memberships + 1 tutor | call `getThreadForCohort(callerId, cohortId)` | resolves `participantCount: 4` — tutor + every currently active membership |
| No private sub-threads within a group thread | mock the same group cohort; call as two different active students | call `getThreadForCohort` for each | both resolve the identical `threadId` — confirms a single shared thread, not a per-student private one |
| Cohort not yet confirmed/paid | mock cohort `status` prior to payment confirmation | call `getThreadForCohort(callerId, cohortId)` | throws `ApiError(403, "Messaging is not available for this cohort")` |
| Caller no longer an active member | mock the caller's `CohortMembership` as `ENDED`/removed | call `getThreadForCohort(callerId, cohortId)` | throws the identical `ApiError(403, "Messaging is not available for this cohort")` |
| Caller with no relation to the cohort at all (IDOR) | mock a real, existing cohort the caller was never a member or tutor of | call `getThreadForCohort(callerId, otherCohortId)` | throws the same `ApiError(403, ...)` — this is tested against a real cohort belonging to someone else, not merely a nonexistent id, so it actually exercises the membership check rather than a 404 path |

#### listMessages

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns paginated chronological history | mock several `Message` rows ordered by `createdAt` | call `listMessages(callerId, cohortId, 1, 50)` | resolves `PaginatedMessageDTO` in chronological order matching §0.6 shape |
| Same membership rule enforced | mock caller no longer an active member | call `listMessages(callerId, cohortId, 1, 50)` | throws `ApiError(403, "Messaging is not available for this cohort")` |
| Archived thread still returns full history | mock `MessageThread.status: ARCHIVED` (cohort ended 90+ days ago) with prior messages intact | call `listMessages(callerId, cohortId, 1, 50)` | resolves the full message list unchanged — archiving is a client list-visibility change only, never a deletion (UC-59); tested as its own case distinct from a non-archived thread, not inferred from one |
| No messages yet | mock an empty message set on a valid, active thread | call `listMessages(...)` | resolves `{ messages: [], page, limit, total: 0 }`, not an error |

#### sendMessage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates the message and notifies every other active participant | mock a `ONE_TO_THREE` cohort (tutor + 3 students), caller is one of the students | call `sendMessage(callerId, cohortId, "hi")` | creates a `Message` row; `dispatchNotification` (`type: NEW_MESSAGE`) is called exactly 3 times — tutor + the other 2 students — and never for the sender itself |
| 1-to-1 send notifies exactly the other party | mock a `ONE_TO_ONE` cohort, caller is the student | call `sendMessage(...)` | `dispatchNotification` called exactly once, targeting the tutor |
| Blocked when not an active member | mock caller no longer active | call `sendMessage(callerId, cohortId, "hi")` | throws `ApiError(403, "Messaging is not available for this cohort")` |
| Blocked when the thread is closed by Admin | mock `MessageThread.status: CLOSED_BY_ADMIN` | call `sendMessage(callerId, cohortId, "hi")` | throws `ApiError(403, "This conversation has been closed")` — a genuinely distinct branch from the not-a-member 403 above, triggered by thread state rather than membership state |
| Notification failure never blocks the send | mock `dispatchNotification` to throw | call `sendMessage(callerId, cohortId, "hi")` | resolves the created `MessageDTO` successfully regardless — the message write must not roll back because a downstream notification failed, matching `notification.service.ts`'s own no-block philosophy (shared-config, 9-1) |

---

### 9.4 Test Case Detail — messaging.controller.test.ts / messaging.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All three routes require auth | no Authorization header | request `GET /messaging/cohorts/:id/thread`, `GET .../messages`, `POST .../messages` | all three `401` |
| getThread/listMessages/sendMessage always use req.user.id as callerId | mock service | call each handler with a caller token | the underlying service call receives `req.user.id`, never a client-suppliable caller field from the body/query |
| cohortId is always taken from the route param | mock service | call controller | service called with `req.params.cohortId`, not any body-supplied cohort id |
| sendMessage validates body against sendMessageSchema | valid auth token | request `POST /messaging/cohorts/:id/messages` with `{ body: "" }` | rejected by `validate(sendMessageSchema)`, controller never called |

---

### 9.5 Test Case Detail — adminMessaging.service.test.ts

FRs: FR-MS-004, FR-AD-017. **OWASP: A01:2021 – Broken Access Control (Admin-only surface).**

#### viewThreadForDispute

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns full thread + paginated messages for any thread | mock an existing thread, no membership restriction applied (Admin is not scoped to being a participant) | call `viewThreadForDispute(threadId, 1, 50)` | resolves `AdminThreadDetailDTO` including message history |
| Thread doesn't exist | mock lookup → `null` | call `viewThreadForDispute(unknownId, 1, 50)` | throws common `ApiError(404, ...)` |

#### closeThread

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Closes a thread | mock an `ACTIVE` thread | call `closeThread(threadId, adminId, reason)` | resolves `{ id, status: 'CLOSED_BY_ADMIN', closedById: adminId, closedAt }` |
| Does not itself duplicate the send-blocking check | spy on any call into `messaging.service.ts` | call `closeThread(...)` | assert `closeThread` only writes the `MessageThread` status row — the resulting `403` on a future `sendMessage` is enforced entirely by `sendMessage`'s own status check (9.3), not re-implemented here |
| Closing an already-closed thread | mock thread already `CLOSED_BY_ADMIN` | call `closeThread(threadId, adminId, reason)` again | **flagged, not hard-asserted:** Doc 8-5 does not specify whether re-closing an already-closed thread is a no-op, a `409`, or simply overwrites `closedAt`/`closedById` — this doc requires the implementer's chosen behavior be applied consistently and asserts only that no unhandled exception occurs, rather than guessing a specific response shape |

---

### 9.6 Test Case Detail — adminMessaging.controller.test.ts / adminMessaging.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Both routes require Admin | no Authorization header, then a Tutor token | request `GET /admin/messaging/threads/:id` and `POST .../close` with (a) no token, (b) a Tutor token | (a) `401`; (b) `403` — a Tutor or Student/Parent JWT must never reach either handler |
| closeThread passes req.user.id as adminId | mock service | call controller with a valid Admin token | `closeThread` called with `req.user.id` as `closedById`, never a client-suppliable admin id from the body |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The group-thread notify-all-but-sender case is tested against a real 4-participant cohort (tutor + 3 students) with an explicit assertion on the *call count* (3, never 4 or 1) — not just that `dispatchNotification` was called "at least once," which would miss a bug where only the tutor or only one student is notified.
- [ ] The archived-thread-still-readable case and the not-a-member-403 case are tested as genuinely separate branches — a shared/collapsed test could pass even if archiving accidentally also blocked reads, which Doc 8-5 explicitly says must never happen.
- [ ] The closed-thread-blocks-send `403` and the not-active-member `403` are asserted with their distinct message strings, not treated as interchangeable just because both are status `403`.
- [ ] The single-shared-thread (no private sub-threads) case is tested by calling `getThreadForCohort` as two *different* students in the same group cohort and asserting the identical `threadId`, not inferred from one student's result alone.
- [ ] The IDOR case for `getThreadForCohort`/`listMessages` is tested against a real, different cohort the caller has no relation to — not just a nonexistent cohort id, which would only exercise a 404-adjacent path rather than the membership check itself.

---

### 9.8 Out of Scope for Automated Testing (and why)

- **`archiveMessageThreads.job.ts` interval scheduling** — no business logic of its own; the archiving effect it produces is already covered via `messaging.service.test.ts`'s archived-thread-still-readable case, and the cron registration itself is excluded per the standing convention.
- **Message content moderation/profanity filtering** — **explicitly flagged as an open item**: no automated content filter is documented anywhere in Docs 02/06/08 for `sendMessage`; the only content-safety control specified is manual Admin review-and-close (FR-MS-004). This test suite does not fabricate a moderation test for a control that was never specified, but records the gap for whoever owns Section 14/15 to consider before launch.
- **Server-side rate limiting on `sendMessage`** — resolved (Doc 02 NFR-013): `rateLimiter.middleware.ts` (30/min per account) is applied at the route level and unit-tested in `9-1-shared-config.md`'s `rateLimiter.middleware.test.ts` section — not re-tested per-feature, since the middleware itself is feature-agnostic and its application here is a one-line route change (`8-5-messaging.md`).
- **Real-time delivery (WebSocket/push) of new messages** — the backend's contract here is REST create/read; any live-update transport is a frontend/infrastructure concern outside this feature's `src/services/*`.
- **Jitsi/video conferencing** — unrelated to this feature; see `class-delivery-library`'s test doc (9-4) for that boundary.

---

**Next:** proceed to → [9-6. Backend Test Documentation: Gamification & Engagement]
