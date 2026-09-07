## Project: AKEWTutor — Frontend Function-Level Spec: In-Platform Messaging
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `05-messaging-api.md`. **Frontend spec reference:** `05-messaging-frontend.md`.

**Depends on:** Matching & Cohorts (hard — a `MessageThread` is 1:1 with a `Cohort`).

**Links back to:** [07-frontend-specification/05-messaging-frontend.md], [05b. Frontend Folder & File Structure §5]
**Links forward to:** [9-5. Frontend Test Spec: In-Platform Messaging]

---

### Shared Pattern: Simple Query Hook

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useThread | GET /messaging/cohorts/:cohortId/thread | [THREAD, cohortId] | `enabled: !!cohortId` |
| useReviewThread (admin) | GET /admin/messaging/threads/:threadId | [ADMIN_THREAD, threadId, page] | `enabled: !!threadId` |

### Shared Pattern: Simple Mutation + Invalidation

| Hook | Endpoint | Invalidates |
|---|---|---|
| useCloseThread | POST /admin/messaging/threads/:id/close | [ADMIN_THREAD, threadId] |

---

### src/hooks/useMessaging.ts — full blocks

#### useMessages

| Field | Detail |
|---|---|
| Signature | `useMessages(cohortId: string, page = 1): UseQueryResult<{ messages: Message[]; page: number; limit: number; total: number }>` |
| Purpose | Wraps `GET /messaging/cohorts/:cohortId/messages`. |
| Logic | `enabled: !!cohortId`; `refetchInterval: 8_000` — there is no websocket in V1, so light polling stands in for real-time delivery (§5.3). This is the one query in the feature that polls unconditionally (not gated on a job-driven server state per §0.4, but on the absence of push infrastructure) — worth keeping in mind if a future V2 replaces this with a socket, at which point the polling interval should be removed rather than left running alongside a push channel. |
| Test file | `tests/hooks/useMessaging.test.ts` |

#### useSendMessage

| Field | Detail |
|---|---|
| Signature | `useSendMessage(cohortId: string): UseMutationResult<Message, AxiosError, string>` |
| Purpose | Wraps `POST /messaging/cohorts/:cohortId/messages`. |
| Side effects | `onSuccess` invalidates `[MESSAGES, cohortId]` — the new message is picked up on the next poll/refetch rather than optimistically appended to the cache, since the 8s polling interval already makes an optimistic update's window of benefit small, and avoiding it sidesteps having to reconcile a client-generated temporary ID against the server's real one. |
| Edge cases | A `403` (`"Messaging is not available for this cohort"` or the `CLOSED_BY_ADMIN` variant) is handled by the composer, not this hook — see `MessageComposer` below (§5.6's two distinct 403 cases). |
| Test file | `tests/hooks/useMessaging.test.ts` |

---

### src/pages/MessagingPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/messaging` — `ProtectedRoute('any')` + `DashboardLayout` |
| Local state | `selectedCohortId: string \| null`. |
| Behavior | 1. Reads the caller's cohorts via `useMyCohorts()` (8-3). 2. If exactly one active cohort exists, sets `selectedCohortId` to it automatically and renders `MessageThreadView` directly — no thread-picker step. 3. If more than one, renders a thread-picker list first; selecting one sets `selectedCohortId`. 4. If zero, renders `EmptyState` ("no active class yet") rather than any thread UI. |

**States:** loading (cohort list) · empty (no cohorts) · success (picker, or thread view if exactly one)

### src/components/MessageThreadView.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ cohortId: string }` |
| Behavior | 1. `useThread(cohortId)` for metadata (`format`, `status`, `participantCount`), `useMessages(cohortId, page)` for the message list. 2. Renders sender name/role per message **only when `format !== 'ONE_TO_ONE'`** (i.e., `participantCount > 2`) — the same component renders both a pair thread and a group thread; the identity-display rule is the only branch (§5.5), not two separate components. 3. Renders `MessageComposer` at the bottom, passing through `useThread`'s `status` so the composer can render the correct disabled-state copy (see below) before ever attempting a send. 4. If `useThread` or `useMessages` resolves with a `403`, replaces the whole view with `EmptyState` and a link back to the cohort/dashboard (§5.6) — this is the expected state for a not-yet-confirmed or ended cohort's thread, not a bug, so it must not render as a generic error toast. |
| Edge cases | An archived thread (cohort ended 90+ days ago) still renders here read-only if navigated to directly — `MessagingPage`'s own "active" filter (client-side only) is what would have hidden it from a list view; this component itself always shows full history when given a valid `cohortId` (§5.6). |
| Test file | `tests/components/MessageThreadView.test.tsx` |

### src/components/MessageComposer.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ cohortId: string; threadStatus: MessageThread['status'] }` |
| Local state | `body: string` (1–2000 chars, client-validated before enabling submit). |
| Behavior | 1. If `threadStatus === 'CLOSED_BY_ADMIN'`, renders disabled with copy distinguishing "This conversation has been closed" from any other unavailable case (§5.6) — this branch is read directly off `useThread`'s successful metadata response, not parsed out of a 403 error message string, per the frontend spec's explicit guidance. 2. Otherwise, on submit, `useSendMessage(cohortId).mutate(body)`; clears the input on success. 3. **No attachment/file-upload affordance exists in this component at all** — not even a disabled/greyed-out button — since text-only is a deliberate rule (FR-SC-004, §5.5), and rendering even a disabled control would misrepresent the feature as "coming soon" rather than "not offered." |
| Test file | `tests/components/MessageComposer.test.tsx` |

---

### src/pages/admin/MessageThreadReviewPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/messaging/:threadId` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | 1. `useReviewThread(threadId, page)` — read-only full history, reached from `DisputeCard`'s (8-8) `relatedThreadId` link when investigating a complaint. 2. A "Close conversation" action (required reason) wired to `useCloseThread`, consistent with the required-reason pattern used for tutor rejection (8-2) and cohort rejection (8-3) — reasons are never optional on any admin action across the app that terminates something for another party. |

**States:** loading · error (404 — thread not found) · success (read-only history + close action)

---

**Next:** proceed to → [8-6. Frontend: Gamification & Engagement]
