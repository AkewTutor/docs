## Project: AKEWTutor — Frontend Specification
**Feature:** In-Platform Messaging
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `05-messaging-api.md`

**Depends on:** Matching & Cohorts (hard — a `MessageThread` is 1:1 with a `Cohort`).

**Links back to:** [0. Frontend Conventions], [06-api/05-messaging-api.md], [05b. Frontend Folder & File Structure §5]
**Links forward to:** [8-5. Frontend Function-Level Spec: In-Platform Messaging]

---

### 5.1 Routes

```
/messaging                       → ProtectedRoute('any') → DashboardLayout → MessagingPage
/admin/messaging/:threadId          → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → MessageThreadReviewPage
```

`MessagingPage` is reachable by Student, Parent, or Tutor (`Student|Parent|Tutor` per the API's auth label) — it does not take a `cohortId` route param directly; instead it lists the caller's cohort(s) via `useMyCohorts` (03-matching-cohorts-frontend.md §3.4) and lets them pick a thread if they have more than one active cohort, defaulting straight to the thread view if they have exactly one.

### 5.2 Types (added to src/types/index.ts)

```typescript
export interface MessageThread {
  id: string;
  cohortId: string;
  format: 'ONE_TO_ONE' | 'ONE_TO_THREE' | 'ONE_TO_FIVE';
  status: 'ACTIVE' | 'ARCHIVED' | 'CLOSED_BY_ADMIN';
  participantCount: number;
}

export interface Message {
  id: string;
  senderId: string;
  senderRole: 'STUDENT' | 'PARENT' | 'TUTOR';
  body: string;
  createdAt: string;
}
```

### 5.3 Hooks (src/hooks/useMessaging.ts)

```typescript
export function useThread(cohortId: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.THREAD, cohortId],
    queryFn: () => api.get<MessageThread>(`/messaging/cohorts/${cohortId}/thread`).then((r) => r.data),
    enabled: !!cohortId,
  });
}

export function useMessages(cohortId: string, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.MESSAGES, cohortId, page],
    queryFn: () => api.get<{ messages: Message[]; page: number; limit: number; total: number }>(
      `/messaging/cohorts/${cohortId}/messages`, { params: { page, limit: 50 } }
    ).then((r) => r.data),
    enabled: !!cohortId,
    refetchInterval: 8_000, // no websocket in V1 — light polling stands in for real-time delivery
  });
}

export function useSendMessage(cohortId: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: string) => api.post<Message>(`/messaging/cohorts/${cohortId}/messages`, { body }).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.MESSAGES, cohortId] }),
  });
}
```

### 5.4 Hooks (src/hooks/useAdminMessaging.ts)

```typescript
export function useReviewThread(threadId: string, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.ADMIN_THREAD, threadId, page],
    queryFn: () => api.get(`/admin/messaging/threads/${threadId}`, { params: { page, limit: 50 } }).then((r) => r.data),
    enabled: !!threadId,
  });
}

export function useCloseThread() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ threadId, reason }: { threadId: string; reason: string }) =>
      api.post(`/admin/messaging/threads/${threadId}/close`, { reason }),
    onSuccess: (_d, vars) => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_THREAD, vars.threadId] }),
  });
}
```

### 5.5 Components

| File | Responsibility |
|---|---|
| `MessageThreadView.tsx` | Scrollable message list; renders a pair-thread (1-to-1, `participantCount: 2`) and a group thread (`ONE_TO_THREE`/`ONE_TO_FIVE`) with the same component — the only difference is whether `senderRole`/name is shown per message (always shown in a group thread so participants can tell members apart; can be omitted in a 2-person pair thread) |
| `MessageComposer.tsx` | Text-only input, 1–2000 chars — **no attachment/file-upload UI exists here at all**, matching the text-only rule (FR-SC-004); this is a deliberate absence, not an oversight, and should not be added even as a disabled/greyed-out affordance |

### 5.6 Behavior Notes

- A `403` from `useThread`/`useMessages`/`useSendMessage` ("Messaging is not available for this cohort") should route the person back to their cohort/dashboard view with an `EmptyState`, not a generic error toast — this is the expected state for a cohort that isn't yet confirmed/paid or has since ended, not a bug.
- A `403` specifically for `status: CLOSED_BY_ADMIN` ("This conversation has been closed") should be distinguished in the composer's disabled-state copy from the "not yet available" case above, since they mean different things to the person reading it — the API doesn't return a distinguishing field beyond the message string; if the two need separate UI copy, `useThread`'s `status: 'CLOSED_BY_ADMIN'` (already in the type) should be read on the successful thread-metadata call *before* attempting to send, rather than parsed out of the 403 message text.
- `MessagingPage`'s cohort/thread archiving (90 days post-cohort-end) only affects which threads appear in an "active" list client-side filter — `useMessages` still returns full history for an archived thread if the person navigates to it directly (05-messaging-api.md §5.2), so the page should support viewing an archived thread read-only rather than hiding it entirely.

---

**Next:** proceed to → [06. Gamification & Engagement Frontend]
