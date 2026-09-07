## Project: AKEWTutor — Frontend Function-Level Spec: Shared Config (Auth, Notifications, Policies, Announcements)
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `01-shared-config-api.md`. **Frontend spec reference:** `01-shared-config-frontend.md`.

Covers every frontend file not owned by a single feature — types, constants, the full routing tree, axios, and the shared layouts — plus this feature's own owned files (auth, notifications, policies, announcements). Shared-config is the foundation feature (Feature Decomposition §1.1's "no dependency of any kind"), which is why the cross-cutting infra lives here rather than in its own separate file.

**Links back to:** [07-frontend-specification/01-shared-config-frontend.md], [05b. Frontend Folder & File Structure §1]
**Links forward to:** [9-1. Frontend Test Spec: Shared Config]

---

### src/types/index.ts (full file — consolidated across all 8 features)

One interface block per feature, copied field-for-field from each `07-frontend-specification/0X-*.md` file's §X.2 (per frontend conventions §0.5) — not re-derived here. Rather than re-listing every field again, this file is the single place the *complete* type file is assembled:

| Feature | Interfaces (see that Doc 07 file's §X.2 for exact fields) |
|---|---|
| Shared Config | `Role`, `AuthUser`, `LoginResponse`, `RegisterResponse`, `AppNotification`, `PolicyDocument`, `Announcement` |
| Accounts & Guardianship | `StudentProfile`, `ParentStudentRelationship`, `TutorProfile`, `TutorSubjectRanking`, `AvailabilitySlot`, `Subject` |
| Matching & Cohorts | `CohortFormat`, `TutorRecommendation`, `MatchRequest`, `Cohort`, `CohortMember` |
| Class Delivery & Library | `SessionStatus`, `RecordingStatus`, `ScheduledSession`, `RecordingConsentStatus`, `Recording`, `LibraryMaterial`, `RescheduleRequest`, `WeeklyAssessment` |
| Messaging | `MessageThread`, `Message` |
| Gamification & Engagement | `XPProgress`, `LeaderboardEntry`, `Leaderboard`, `Badge`, `AdminBadge`, `Challenge`, `ChallengeProgress` |
| Payments & Earnings | `PaymentRecord`, `PaymentPauseStatus`, `FormatPricing`, `RefundCase`, `TutorEarningsSummary`, `PayoutBatch`, `PromotionCode` |
| Support, Trust & Admin | `ComplaintCategory`, `ComplaintStatus`, `ResolutionAction`, `ComplaintSummary`, `ComplaintDetail`, `AdminComplaintDetail`, `SupportContact`, `PlatformHealth` |

**Ownership rule going forward:** a feature file (8-2 through 8-8 below) only *adds* to this file — it never redefines a type owned by another feature. If a later feature needs a field from an earlier one's type (e.g. `04-class-delivery-library` referencing `Cohort.format`), it imports the existing interface rather than duplicating a subset of its fields locally.

---

### src/constants/index.ts (full file — consolidated across all 8 features)

**`ROUTES`** — one entry per page in the routing tree below, grouped by feature comment block, matching frontend conventions §0.2/§0.5 exactly (a page's `ROUTES` constant and its actual path must never drift). Not re-typed field-by-field here since it's a 1:1 mechanical mirror of the tree in the next section — see that tree for the literal path strings.

**`QUERY_KEYS`** — one entry per queryable resource, collected from every hook file across Docs 8-1–8-8. Naming pattern: `SCREAMING_SNAKE_CASE`, matching the hook's own `[QUERY_KEYS.X, ...params]` usage already shown in each Doc 07 file (e.g. `NOTIFICATIONS`, `STUDENT_PROFILE`, `MY_COHORTS`, `SESSIONS`, `THREAD`, `XP_PROGRESS`, `PAYMENT_HISTORY`, `DISPUTE_QUEUE` — full list is every `QUERY_KEYS.*` token already referenced in Docs 07 §X.3 onward for that feature).

---

### src/routes/index.tsx (full file)

**Fix (2026-09-07 audit):** earlier drafts of this file specified React Router's classic declarative API (`<Routes>`/`<Route>`, an `AppRoutes` component) and mis-cited that as "per the base template." The base template (`template-react` v3, §2.5/§9.1) actually mandates React Router v7's **data-router API** — `createBrowserRouter()` building an object-based route tree, consumed by `<RouterProvider>` in `App.tsx`, with `router` (not a component) as the default export — specifically so every page can be individually code-split via `React.lazy()` + a per-route `Suspense` boundary. Rewritten below to match. Nothing about the route/guard/layout structure itself changes — only the API used to express it.

```tsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter } from 'react-router-dom';
import PublicLayout from '@/components/layouts/PublicLayout';
import AuthLayout from '@/components/layouts/AuthLayout';
import DashboardLayout from '@/components/layouts/DashboardLayout';
import ProtectedRoute from './ProtectedRoute';
import PublicRoute from './PublicRoute';
// ...lazy page imports, one per ROUTES entry, e.g.:
// const LoginPage = lazy(() => import('@/pages/auth/LoginPage'));

function PageLoader() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-background">
      <p className="text-muted-foreground text-sm">Loading...</p>
    </div>
  );
}

// Wrap each lazy element so Suspense boundaries stay per-page, per template §9.1.
const withSuspense = (element: React.ReactNode) => (
  <Suspense fallback={<PageLoader />}>{element}</Suspense>
);

const router = createBrowserRouter([
  {
    element: <PublicLayout />,
    children: [
      { path: '/', element: withSuspense(<LandingPage />) },
      { path: '/policies/:type', element: withSuspense(<PolicyPage />) },
    ],
  },

  {
    element: <PublicRoute />,
    children: [
      {
        element: <AuthLayout />,
        children: [
          { path: '/login', element: withSuspense(<LoginPage />) },
          { path: '/register/student', element: withSuspense(<RegisterPage mode="student" />) },
          { path: '/register/parent', element: withSuspense(<RegisterPage mode="parent" />) },
          { path: '/register/tutor', element: withSuspense(<RegisterPage mode="tutor" />) },
          { path: '/verify-contact', element: withSuspense(<VerifyContactPage />) },
          { path: '/forgot-password', element: withSuspense(<ForgotPasswordPage />) },
          { path: '/reset-password', element: withSuspense(<ResetPasswordPage />) },
          { path: '/invite/:token/activate', element: withSuspense(<InviteActivationPage />) },
        ],
      },
    ],
  },

  // /student/* and /parent/* are NOT wrapped in one blanket ProtectedRoute here — several
  // /student/... leaf routes are Parent-reachable too (find-tutor, upcoming-classes,
  // achievements, etc.). Each leaf route below carries its own ProtectedRoute with the
  // exact role set (or 'any') declared in its owning feature file (8-2 through 8-8);
  // DashboardLayout is still shared since all four roles use the same shell.
  {
    element: <DashboardLayout />,
    children: [
      {
        path: '/student/*',
        // StudentRoutes — each leaf route individually wrapped in ProtectedRoute(roles) per its
        // owning feature file below; role sets vary per route (e.g. ['STUDENT'] for
        // /student/profile, ['STUDENT','PARENT'] for /student/find-tutor, 'any' for /student/achievements)
        children: [], // populated by 8-2 through 8-8
      },
      {
        path: '/parent/*',
        children: [], // ParentRoutes — same per-leaf-route pattern
      },
    ],
  },
  {
    element: <ProtectedRoute roles={['TUTOR']} />,
    children: [
      {
        element: <DashboardLayout />,
        children: [{ path: '/tutor/*', children: [] /* TutorRoutes */ }],
      },
    ],
  },
  {
    element: <ProtectedRoute roles={['ADMIN']} />,
    children: [
      {
        element: <DashboardLayout />,
        children: [{ path: '/admin/*', children: [] /* AdminRoutes */ }],
      },
    ],
  },

  {
    element: <ProtectedRoute roles="any" />,
    children: [
      {
        element: <DashboardLayout />,
        children: [
          { path: '/notifications', element: withSuspense(<NotificationsPage />) },
          { path: '/messaging', element: withSuspense(<MessagingPage />) },
          { path: '/library', element: withSuspense(<LibraryPage />) },
          { path: '/recording-consent', element: withSuspense(<RecordingConsentPage />) },
          { path: '/reschedule/:sessionId', element: withSuspense(<RequestReschedulePage />) },
          { path: '/payments', element: withSuspense(<PaymentPage />) },
          { path: '/payments/history', element: withSuspense(<PaymentHistoryPage />) },
          { path: '/payments/paused', element: withSuspense(<PaymentPausedPage />) },
          { path: '/complaints', element: withSuspense(<SubmitComplaintPage />) },
          { path: '/support', element: withSuspense(<SupportContactPage />) },
        ],
      },
    ],
  },

  { path: '*', element: withSuspense(<NotFoundPage />) },
]);

export default router;
```

`router` is consumed in `src/App.tsx` via `<RouterProvider router={router} />`, per template §9.1 — `App.tsx` also wraps this in the template's `ErrorBoundary` (template §10.3) and the `QueryClientProvider`.
The exact leaf routes under each `/student/*`, `/parent/*`, `/tutor/*`, `/admin/*` block are enumerated in full in each owning feature's own file below (8-2 through 8-8) — this file only fixes the four dashboard mount points plus the shared/public/any-role routes that belong to no single feature. `ProtectedRoute`'s `roles` prop accepts either a `Role[]` or the literal string `'any'`, matching frontend conventions §0.2's four route categories. `/tutor/*` and `/admin/*` are wrapped in one blanket guard above because every leaf route in those two features is in fact role-exclusive; `/student/*` and `/parent/*` are not, since Parent-reachable leaf routes exist under the `/student/` prefix (frontend conventions §0.2's callout) — those two guard themselves per-route instead.

---

### src/routes/ProtectedRoute.tsx (new — full block)

| Field | Detail |
|---|---|
| Signature | `ProtectedRoute({ roles: Role[] \| 'any' }): JSX.Element` |
| Purpose | Gate a subtree to authenticated users, optionally restricted to specific roles. |
| Logic | 1. Read `{ token, user }` from `auth.store.ts`. 2. If `!token`, redirect to `/login?returnTo=<current path>`. 3. If `roles !== 'any'` and `user.role` is not in `roles`, redirect to that user's own role-default route (`roleDefaultRoute(user.role)`, see util below) rather than a generic "forbidden" page — a Tutor hitting `/admin/*` is sent to `/tutor`, not shown an error screen. 4. Otherwise render `<Outlet />`. |
| Edge cases | A `token` present but `user` somehow `null` (a corrupted persisted store) is treated the same as `!token` — redirect to login, never render with a `null` user. |
| Test file | `tests/routes/ProtectedRoute.test.tsx` |

### src/routes/PublicRoute.tsx (new — full block)

| Field | Detail |
|---|---|
| Signature | `PublicRoute(): JSX.Element` |
| Purpose | Gate the login/register/reset subtree away from already-authenticated visitors. |
| Logic | 1. Read `token` from `auth.store.ts`. 2. If present, redirect to `roleDefaultRoute(user.role)`. 3. Otherwise render `<Outlet />`. |
| Edge cases | None beyond §0.1's routing-structure note — this guard has no role parameter, unlike `ProtectedRoute`. |
| Test file | `tests/routes/PublicRoute.test.tsx` |

### src/lib/roleDefaultRoute.ts (new util — full block)

| Field | Detail |
|---|---|
| Signature | `roleDefaultRoute(role: Role): string` |
| Purpose | Single source of truth for "where does this role land after login," used by `LoginPage`, `ProtectedRoute`, and `PublicRoute` so the three call sites can never drift into disagreement. |
| Logic | Maps `STUDENT → /student`, `PARENT → /parent`, `TUTOR → /tutor`, `ADMIN → /admin`. No default/fallback branch — an unrecognized role is a data-integrity bug, not a routing case to silently paper over, so this throws rather than falling through to `/`. |
| Test file | `tests/lib/roleDefaultRoute.test.ts` |

---

### src/lib/axios.ts (modify)

| Field | Detail |
|---|---|
| Request interceptor | Attaches `Authorization: Bearer <token>` from `auth.store.ts` on every request when a token is present, per frontend conventions §0.4 — no separate unauthenticated client instance. |
| Response interceptor (401) | Per frontend conventions §0.6 (Critical-fix #1 revision): on a `401`, (1) if `auth.store.ts` has a `refreshToken`, call `POST /auth/refresh` once with it — on success, call `setAuth` with the new `accessToken`/`refreshToken`/existing `user` and retry the original request with the new access token; (2) if the refresh call itself fails (expired/reused/missing refresh token), or no `refreshToken` was stored, clear `auth.store.ts` (token + refreshToken + user) and redirect to `/login?returnTo=<currentPath>`. |
| Refresh concurrency | Multiple requests failing with `401` at once must trigger only one `/auth/refresh` call, not one per request — in-flight requests queue behind the single refresh promise and retry once it resolves, since the refresh token is single-use/rotating (NFR-015) and a second concurrent call would revoke the token the first call is still using. |
| M8 fix — resolved | Doc 05b's file inventory previously described this interceptor as redirecting "to the correct login route based on the failing request's role prefix," implying multiple login destinations. That line is now corrected at the source (`05b-frontend-structure.md`) to match this file and Doc 07 §0.6's single `/login?returnTo=` design — all three docs now agree, nothing left to confirm. |
| Envelope unwrap | `SuccessResponse`/`ErrorResponse` (00-api-conventions §0.1) unwrapped once in the interceptor so every hook's `.then((r) => r.data)` receives the inner `data` payload directly, not the full envelope. |

---

### src/components/layouts/PublicLayout.tsx, AuthLayout.tsx (new — light blocks)

| File | Behavior |
|---|---|
| `PublicLayout.tsx` | Renders `TopNavBar`, `<Outlet />`, `Footer` — no auth awareness at all, per frontend conventions §0.1's route-category table. |
| `AuthLayout.tsx` | Minimal chrome around the login/register/reset forms — logo + centered card, no nav. |

### src/components/layouts/DashboardLayout.tsx (new — full block)

| Field | Detail |
|---|---|
| Behavior | 1. Reads `user.role` from `auth.store.ts`. 2. Renders the matching sidebar: `StudentSidebar` / `ParentSidebar` / `TutorSidebar` / `AdminSidebar` (the last three specified in their owning feature files — `ParentSidebar` in 8-2, `AdminSidebar` composed across 8-2/8-8 as it gains feature-specific nav entries). 3. Renders a mobile header (hamburger → sidebar drawer) below a breakpoint. 4. Renders `<Outlet />` for the matched child route. |
| Edge cases | A role with no matching sidebar case (should `roleDefaultRoute`'s exhaustiveness ever be bypassed) throws in development rather than rendering a blank shell — this mirrors `roleDefaultRoute`'s own no-fallback stance. |
| Test file | `tests/components/layouts/DashboardLayout.test.tsx` |

---

### src/store/auth.store.ts (new — full block, per frontend spec 1.3)

| Field | Detail |
|---|---|
| Shape | `{ token: string \| null; refreshToken: string \| null; user: AuthUser \| null; setAuth: (token: string, refreshToken: string, user: AuthUser) => void; logout: () => void }` |
| Persistence | Zustand `persist` middleware, storage key `auth-storage` — the only persisted client state in the app (frontend conventions §0.3). |
| `setAuth` | Sets all three fields in one call — never independently, since a `token` without a matching `user` (or vice versa) is an invalid intermediate state every `ProtectedRoute`/sidebar role-check could observe mid-render. `refreshToken` is required alongside `token`/`user` for the same reason: a session with an access token but no stored refresh token cannot survive the 30-minute TTL (NFR-014), so partial auth state is never valid. |
| `logout` | Clears all three fields to `null`. Does **not** itself call `POST /auth/logout` — composed together only inside `useLogout` below, so nothing else in the codebase calls `logout()` directly and skips the API call. |
| Edge cases | A stale/expired persisted access token is not proactively cleared by the store — discovered lazily on the next authenticated request's `401`, handled by the axios interceptor's refresh-then-redirect flow below. |
| Test file | `tests/store/auth.store.test.ts` |

---

### src/hooks/useAuth.ts (per frontend spec 1.4)

#### useRegister, useVerifyContact, useResendVerification, useForgotPassword, useResetPassword — shared pattern: simple mutation, no cache invalidation

| Hook | Endpoint | Notes |
|---|---|---|
| useRegister(role) | POST /auth/register/:role | `role` is a hook parameter, not per-call — one hook instance per registration page mount |
| useVerifyContact | POST /auth/verify-contact | — |
| useResendVerification | POST /auth/resend-verification | — |
| useForgotPassword | POST /auth/forgot-password | Success and "no account found" both resolve as mutation success (API's deliberate non-disclosure, §1.8) — there is no error branch to handle for a non-existent identifier |
| useResetPassword | POST /auth/reset-password | — |

#### useLogin (full block)

| Field | Detail |
|---|---|
| Signature | `useLogin(): UseMutationResult<LoginResponse, AxiosError, { identifier: string; password: string }>` |
| Purpose | Wraps `POST /auth/login`. |
| Side effects | `onSuccess` calls `authStore.setAuth(data.accessToken, data.refreshToken, data.user)` — no query invalidation needed, since no authenticated query can have run before a token exists. |
| Edge cases | A `401` ("Invalid email/phone or password") is a genuine mutation error, rendered as a form-level banner by `LoginPage` — never attributed to a specific field, matching the API's deliberately generic message (frontend spec §1.8). |
| Test file | `tests/hooks/useAuth.test.ts` |

#### useLogout (full block)

| Field | Detail |
|---|---|
| Signature | `useLogout(): UseMutationResult<void, AxiosError, void>` |
| Purpose | Wraps `POST /auth/logout`. |
| Side effects | `onSettled` (not `onSuccess`) calls `authStore.logout()` — local session state clears even if the API call fails (e.g., an already-expired token), so the person is never stuck unable to log out client-side because of a network error. |
| Edge cases | After `onSettled`, the next render of any `ProtectedRoute` in the tree redirects to `/login` since `token` is now `null` — no imperative `navigate()` call needed inside the hook itself. |
| Test file | `tests/hooks/useAuth.test.ts` |

---

### src/hooks/useNotifications.ts (per frontend spec 1.5)

| Hook | Category | Detail |
|---|---|---|
| useMyNotifications(unreadOnly, page) | Simple query | GET /notifications, key `[QUERY_KEYS.NOTIFICATIONS, unreadOnly, page]` |
| useMarkRead | Mutation + invalidation | PATCH /notifications/:id/read → invalidates `[QUERY_KEYS.NOTIFICATIONS]` |

`NotificationBell.tsx` (see components below) is the one consumer that sets `refetchInterval: 30_000` on `useMyNotifications(true)` — the full `NotificationsPage` list does not poll, since a person actively viewing the list will refresh by paging/navigating.

---

### src/hooks/usePolicy.ts, useAdminAnnouncements.ts (per frontend spec 1.6)

| Hook | Category | Detail |
|---|---|---|
| usePolicy(type) | Simple query | GET /policies/:type, key `[QUERY_KEYS.POLICY, type]` |
| useAnnouncements(page) | Simple query | GET /admin/announcements, key `[QUERY_KEYS.ANNOUNCEMENTS, page]` |
| useCreateAnnouncement | Mutation + invalidation | POST /admin/announcements → invalidates `[QUERY_KEYS.ANNOUNCEMENTS]` |

---

### src/pages/LoginPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Single shared login form for all four roles (frontend spec §1.7). |
| Route guard + layout | `PublicRoute` + `AuthLayout` |
| Local state | React Hook Form — `identifier`, `password`. |
| Behavior | 1. Client-side Zod validation (non-empty identifier, non-empty password) — catches malformed input before any request. 2. On submit, `useLogin().mutate({ identifier, password })`. 3. On success, reads `?returnTo=` from the URL; if present, `navigate(returnTo)`; otherwise `navigate(roleDefaultRoute(data.user.role))`. The page performs this `navigate()` imperatively inside `onSuccess` (unlike `AdminLoginPage`-style guard-driven redirects in single-role apps) because `returnTo` is a page-specific concern the guard itself doesn't know about. 4. On `401`, renders a form-level banner, per §1.8. |
| Components | none beyond form primitives |

**States:** idle (form) · submitting (button disabled, spinner) · error (401 banner, form re-enabled) · success (imperative redirect, no local success state to render)

### src/pages/RegisterPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | One component parameterized by `mode: 'student' \| 'parent' \| 'tutor'` (frontend spec §1.1), not three files. |
| Route guard + layout | `PublicRoute` + `AuthLayout` |
| Local state | React Hook Form, fields vary by `mode`: student adds `grade` (6–12); parent and tutor add no extra required field beyond the shared base (name, email/phone, password, `termsAccepted`). |
| Behavior | 1. Zod schema selected by `mode` mirrors the backend's `register*Schema` per role (min-8-char password, `termsAccepted: true` required, `grade` client-pre-validated to 6–12 for student mode even though 1–5 is also rejected server-side — §1.8). 2. On submit, `useRegister(mode).mutate(body)`. 3. On success (`201`), if `verificationRequired`, `navigate('/verify-contact', { state: { userId: data.userId } })`; otherwise `navigate('/login')`. 4. A `409` ("account already exists") renders as a field-level error on the email/phone field, not assumed unreachable just because the client-side schema passed. |
| Components | none beyond form primitives, switched per `mode` |

**States:** idle · submitting · error (409 field-level, or other validation) · success (redirect to verify or login)

### src/pages/VerifyContactPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Code entry + resend action. |
| Route guard + layout | `PublicRoute` + `AuthLayout` |
| Local state | `code: string`; `userId` read from route `state` (set by `RegisterPage`) or a query param fallback if the person navigated here directly (e.g., a bookmarked link from an SMS). |
| Behavior | 1. On submit, `useVerifyContact().mutate({ userId, code })`. 2. On success, `navigate('/login')` with a success toast. 3. "Resend" button calls `useResendVerification().mutate(userId)`, disabled for 30s after each press to prevent spamming the SMS/email provider (client-side throttle only — the backend does not enforce a rate limit on this endpoint per the API spec, so this is a UX courtesy, not a substitute for one). |

**States:** idle · submitting · error (invalid/expired code) · success (redirect)

### src/pages/ForgotPasswordPage.tsx, ResetPasswordPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Standard two-step reset flow (frontend spec §1.7). |
| Route guard + layout | `PublicRoute` + `AuthLayout`, both. |
| Behavior (`ForgotPasswordPage`) | 1. On submit, `useForgotPassword().mutate(identifier)`. 2. **Always** renders the same generic success message ("If an account exists, a reset code has been sent") regardless of the mutation's actual outcome — mirrors the API's deliberate non-disclosure (§1.8); this page has no error-state branch for "identifier not found" because that case is indistinguishable from success by design. |
| Behavior (`ResetPasswordPage`) | 1. On submit, `useResetPassword().mutate({ userId, code, newPassword })`. 2. On success, `navigate('/login')`. 3. An invalid/expired code renders a field-level error. |

**States (ForgotPasswordPage):** idle · submitting · success (generic message, no error state exists)
**States (ResetPasswordPage):** idle · submitting · error (invalid/expired code) · success (redirect)

### src/pages/PolicyPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Renders one policy type by route param. |
| Route guard + layout | Public + `PublicLayout` |
| Local state | none |
| Behavior | 1. Reads `:type` from the route. 2. Validates it against the five allowed values (`PRIVACY \| TERMS \| SAFETY \| REFUND \| RULES`) **before** calling `usePolicy(type)` — an invalid `type` renders `NotFoundPage`'s content inline rather than firing a request the backend would 400 on anyway. 3. Renders `usePolicy(type)`'s markdown `content` through the app's markdown renderer. |

**States:** loading · error (network) · success

### src/pages/NotificationsPage.tsx, src/components/common/NotificationBell.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Full list (page) + header-level unread indicator (bell), same underlying data. |
| Route guard + layout (`NotificationsPage`) | `ProtectedRoute('any')` + `DashboardLayout` |
| Behavior (`NotificationsPage`) | 1. `useMyNotifications(unreadOnly, page)`, `unreadOnly` toggled by a local filter control. 2. Each row's "mark read" action calls `useMarkRead().mutate(id)`; clicking an unread row also marks it read as a side effect of opening it, not just via the explicit button. |
| Behavior (`NotificationBell`) | 1. `useMyNotifications(true)` with `refetchInterval: 30_000`. 2. Renders a badge count (capped display at `9+`) and a dropdown of the most recent unread entries. 3. Clicking a dropdown entry calls `useMarkRead().mutate(id)` then navigates to whatever the notification's `type`/`payload` implies (e.g. `NEW_MESSAGE` → `/messaging`, `COMPLAINT_RESOLVED` → `/complaints`) — this mapping is a small internal `notificationTypeToRoute(type, payload)` util, not inline per-type conditionals scattered through the component. |

**States (NotificationsPage):** loading · empty (`notifications: []`, per §0.3 — not an error) · success
**States (NotificationBell):** loading (no visible skeleton, badge simply absent until resolved) · success

### src/pages/admin/AnnouncementsPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Admin composes and reviews platform announcements. |
| Route guard + layout | `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Local state | React Hook Form — `title`, `body`, `audienceRoles: Role[]` (multi-select). |
| Behavior | 1. `useAnnouncements(page)` renders the history list. 2. On submit, `useCreateAnnouncement().mutate(body)`; `audienceRoles` must be non-empty — client-validated before submit since sending to zero roles is a no-op the backend would otherwise silently accept. |

**States:** loading (history) · error · success (list + compose form)

---

**Next:** proceed to → [8-2. Frontend: Accounts & Guardianship]
