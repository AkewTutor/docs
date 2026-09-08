## Project: AKEWTutor — Frontend Test Documentation: Shared Config (Auth, Notifications, Policies, Announcements) + Cross-Cutting Foundations
**Links back to:** [05b. Frontend Folder & File Structure §0–1], [07. Frontend Spec §0, §1], [8-1. Frontend Function-Level Spec: Shared Config]
**Conventions:** see `0-frontend-conventions.md` for routing/state/API-layer conventions; `00-api-conventions.md` §0.1–0.7 for the envelope/error shapes mocked throughout.

**Test runner:** Vitest + React Testing Library (`render`, `screen`, `userEvent`) for components/pages; `renderHook`/`waitFor` from `@testing-library/react` for hooks, wrapped in a fresh `QueryClientProvider` (`new QueryClient({ defaultOptions: { queries: { retry: false } } })`) per test. `vi.mock('@/lib/axios')` mocks the shared `api` client; `vi.mock('@/store/auth.store')` or a real Zustand store reset via `store.setState(initialState)` between tests, per case. `beforeEach(() => vi.clearAllMocks())`.

**Per the standing rule in `05b-frontend-structure.md`** ("every hook file gets a mirrored test file under `tests/`"): every hook in this feature is tested. Beyond hooks, this doc additionally covers every component/page identified as having non-trivial logic per `8-1`'s function-level spec (forms, guards, redirects, generic-response non-disclosure, type validation) — see §9.9 for what is deliberately excluded and why. Every suite here is required to cover: (1) the FR/NFR it implements, traced below; (2) the relevant OWASP Top 10 (2021) risk categories, per `9.X` security subsections; (3) any edge case already flagged in Doc 8-1.

**Scope note:** this is the one frontend test doc that also covers the cross-cutting foundation files (`ProtectedRoute`, `PublicRoute`, `auth.store.ts`, `src/lib/axios.ts`) — they have no owning feature, and shared-config is the foundation feature they sit alongside, mirroring `9-1-shared-config.md`'s backend scope note for middleware/utils.

See `00-test-fixtures.md` and `00-agent-rules.md` for conventions binding this document.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| routes/ProtectedRoute.tsx, PublicRoute.tsx | (enforces role/session gating for every protected page in the app) | NFR-009 |
| store/auth.store.ts | FR-SP-003 (session persistence) | NFR-007, NFR-008 (no plaintext password ever held client-side) |
| lib/axios.ts (interceptors) | FR-SP-003 | NFR-007, NFR-009 |
| hooks/useAuth.ts | FR-SP-001–005, FR-TU-001–002, FR-AC-002, FR-AC-005 | NFR-007, NFR-008 |
| hooks/useNotifications.ts | FR-NO-001–011 | — |
| hooks/usePolicy.ts | FR-SC-001 | — |
| hooks/useAdminAnnouncements.ts | FR-AD-019 | — |
| pages/LoginPage.tsx | FR-SP-003 | NFR-007 |
| pages/RegisterPage.tsx | FR-SP-001–002, FR-TU-001, FR-AC-002 | NFR-008 |
| pages/VerifyContactPage.tsx | FR-SP-004 | — |
| pages/ForgotPasswordPage.tsx, ResetPasswordPage.tsx | FR-SP-005 | NFR-007, NFR-008 |
| pages/PolicyPage.tsx | FR-SC-001 | — |
| pages/NotificationsPage.tsx, components/NotificationBell.tsx | FR-NO-001–011 | — |
| pages/admin/AnnouncementsPage.tsx | FR-AD-019 | NFR-009 (Admin-only) |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/routes/ProtectedRoute.tsx | tests/routes/ProtectedRoute.test.tsx | Component (mocked auth.store, `MemoryRouter`) | ☐ |
| src/routes/PublicRoute.tsx | tests/routes/PublicRoute.test.tsx | Component (mocked auth.store, `MemoryRouter`) | ☐ |
| src/store/auth.store.ts | tests/store/auth.store.test.ts | Unit (real Zustand store instance, real `localStorage` shim) | ☐ |
| src/lib/axios.ts | tests/lib/axios.test.ts | Unit (mocked `axios` instance, mocked `auth.store`, mocked `window.location`) | ☐ |
| src/hooks/useAuth.ts | tests/hooks/useAuth.test.ts | Hook (mocked axios, mocked auth.store) | ☐ |
| src/hooks/useNotifications.ts | tests/hooks/useNotifications.test.ts | Hook (mocked axios) | ☐ |
| src/hooks/usePolicy.ts | tests/hooks/usePolicy.test.ts | Hook (mocked axios) | ☐ |
| src/hooks/useAdminAnnouncements.ts | tests/hooks/useAdminAnnouncements.test.ts | Hook (mocked axios) | ☐ |
| src/pages/LoginPage.tsx | tests/pages/LoginPage.test.tsx | Component (mocked useAuth, `MemoryRouter` with `?returnTo=`) | ☐ |
| src/pages/RegisterPage.tsx | tests/pages/RegisterPage.test.tsx | Component (mocked useAuth), parameterized over `mode` | ☐ |
| src/pages/VerifyContactPage.tsx | tests/pages/VerifyContactPage.test.tsx | Component (mocked useAuth, fake timers for the resend throttle) | ☐ |
| src/pages/ForgotPasswordPage.tsx | tests/pages/ForgotPasswordPage.test.tsx | Component (mocked useAuth) | ☐ |
| src/pages/ResetPasswordPage.tsx | tests/pages/ResetPasswordPage.test.tsx | Component (mocked useAuth) | ☐ |
| src/pages/PolicyPage.tsx | tests/pages/PolicyPage.test.tsx | Component (mocked usePolicy, route param variants) | ☐ |
| src/pages/NotificationsPage.tsx | tests/pages/NotificationsPage.test.tsx | Component (mocked useNotifications) | ☐ |
| src/components/common/NotificationBell.tsx | tests/components/NotificationBell.test.tsx | Component (mocked useNotifications, fake timers for polling) | ☐ |
| src/pages/admin/AnnouncementsPage.tsx | tests/pages/AnnouncementsPage.test.tsx | Component (mocked useAdminAnnouncements) | ☐ |
| src/components/layouts/PublicLayout.tsx, AuthLayout.tsx, DashboardLayout.tsx | — | Not required — no branching logic beyond composing children + the correct sidebar (see §9.9) | — |
| src/components/common/TopNavBar.tsx, Footer.tsx | — | Not required — presentational, active-link styling only (see §9.9) | — |

---

### 9.2 Test Case Detail — ProtectedRoute.test.tsx / PublicRoute.test.tsx

FRs: infrastructure for every authenticated route. **OWASP: A01:2021 – Broken Access Control.**

#### ProtectedRoute

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No session — redirects to login | `auth.store` state: `{ token: null, user: null }` | render `<ProtectedRoute roles={['STUDENT']} />` wrapping a protected page, at path `/student/profile` | redirected to `/login?returnTo=%2Fstudent%2Fprofile` — the `returnTo` is URL-encoded, not raw-concatenated |
| Valid session, role in allow-list | `auth.store` state: `{ token: 'x', user: { role: 'STUDENT' } }` | render `<ProtectedRoute roles={['STUDENT']} />` | protected content renders, no redirect |
| Valid session, role NOT in allow-list | `auth.store` state: `{ token: 'x', user: { role: 'TUTOR' } }` | render `<ProtectedRoute roles={['STUDENT']} />` | redirected (to a "not authorized"/role-default page per Doc 8-1), protected content never rendered — this is the frontend UX layer only; the real enforcement is the backend's `requireRole`, but a client-side gap here would let a Tutor's browser render Student-only DOM before any request is even made |
| `roles: 'any'` — any authenticated role passes | `auth.store` state: `{ token: 'x', user: { role: 'PARENT' } }` | render `<ProtectedRoute roles="any" />` | protected content renders |
| Malformed/partial store state (token present, user null) | `auth.store` state: `{ token: 'x', user: null }` | render `<ProtectedRoute roles={['STUDENT']} />` | treated as unauthenticated — redirected to login, not a crash on `user.role` — defends against a corrupted persisted store surviving a partial `logout()` |

#### PublicRoute

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No session — renders the public/auth page | `auth.store` state: `{ token: null, user: null }` | render `<PublicRoute />` wrapping `LoginPage` | `LoginPage` renders |
| Already authenticated — redirects away | `auth.store` state: `{ token: 'x', user: { role: 'TUTOR' } }` | render `<PublicRoute />` at `/login` | redirected to that role's default dashboard route, not left on the login form — prevents a logged-in Tutor from re-submitting login/registration forms pointlessly |

---

### 9.3 Test Case Detail — auth.store.test.ts

FRs: FR-SP-003. **OWASP: A07:2021 – Identification and Authentication Failures.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `setAuth` stores token + user | fresh store | call `setAuth('token123', { id: '1', role: 'STUDENT' })` | `store.getState().token === 'token123'`, `store.getState().user.role === 'STUDENT'` |
| `logout` clears both fields completely | store pre-populated via `setAuth` | call `logout()` | `token === null`, `user === null` — no stale field left behind (e.g. `user` cleared but a residual `token` string), which the `ProtectedRoute` malformed-state case above depends on never actually happening in practice |
| Persists under the `auth-storage` key, not a generic/default key | call `setAuth(...)` | inspect the persistence layer's write | the persisted key is exactly `auth-storage`, matching frontend conventions §0.3 — a drift here would silently break session restore across a rename |
| Never persists a password or any field beyond token/user identity | call `setAuth(...)` with only `{ id, role, email }` | inspect the full persisted payload | no password, no raw credentials, no PII beyond what `AuthUser` explicitly defines ever lands in `localStorage` — this is the frontend half of NFR-008/NFR-007's intent, since anything written here is a plaintext, script-readable value (OWASP A05:2021 – Security Misconfiguration: sensitive data in browser storage) |
| Rehydration on reload restores a prior session | simulate a pre-existing `auth-storage` value in the storage backend, then re-initialize the store | read `store.getState()` | `token`/`user` match the persisted value — confirms `PublicRoute`'s bounce-if-authenticated logic actually survives a hard refresh, not just an in-memory session |

---

### 9.4 Test Case Detail — axios.test.ts (interceptors)

FRs: FR-SP-003. **OWASP: A07:2021 – Identification and Authentication Failures, A01:2021 – Broken Access Control.**

#### Request interceptor

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Attaches Bearer token when present | `auth.store` has a token | make a request through `api` | outgoing config has `Authorization: Bearer <token>` |
| Omits the header when no token | `auth.store` token is `null` | make a request to a `Public`-labeled endpoint (e.g. `POST /auth/login`) | no `Authorization` header sent — confirms there is no separate unauthenticated client instance silently used incorrectly (frontend conventions §0.4) |

#### Response interceptor — 401 handling

| Case | Setup | Action | Expected result |
|---|---|---|---|
| 401 clears the store and redirects with `returnTo` | mock a `401` response from any endpoint; current path is `/tutor/earnings` | trigger the interceptor | `auth.store.logout()` called; navigation to `/login?returnTo=%2Ftutor%2Fearnings` |
| `returnTo` is never used to build an absolute/off-site URL (open-redirect guard) | mock a `401` while `window.location` reflects a path containing an attacker-influenced fragment, e.g. `/tutor/earnings?x=https://evil.example` | trigger the interceptor | the constructed `returnTo` value is the app-relative pathname+search only (`encodeURIComponent` of `location.pathname + location.search`), never a scheme-qualified URL that could redirect off-domain after a later `navigate(returnTo)` call in `LoginPage` — OWASP A01:2021 (Broken Access Control) / open-redirect class issue if this weren't constrained to a relative path |
| Non-401 errors pass through unmodified | mock a `403`/`500`/`400` response | trigger the interceptor | error is rethrown/propagated to the caller unchanged — no accidental blanket logout on every error class |
| A 401 from the Chapa webhook or another non-Bearer flow is not applicable | n/a | — | out of scope — the webhook is server-to-server and never touches this client-side interceptor at all |

---

### 9.5 Test Case Detail — useAuth.test.ts

FRs: FR-SP-001–005, FR-TU-001–002, FR-AC-002, FR-AC-005. **OWASP: A07:2021, A04:2021 – Insecure Design.**

#### useLogin

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success calls `authStore.setAuth` with response data | mock `POST /auth/login` → `{ accessToken, user }` | `mutate({ identifier, password })` | `authStore.setAuth` called with exactly those two values, nothing else |
| Identical-shape error for wrong password vs. unknown identifier | mock a `401` with the backend's single generic message for both cases | `mutate` with each variant | the hook surfaces the **same** error object/message in both cases — a test asserting these are literally identical (not just "both are some 401") catches a future regression where the frontend accidentally branches on identifier-not-found vs wrong-password and leaks which one it was (this would defeat the backend's own timing/response-shape hardening documented in the backend's `9-1-shared-config.md` §9.18) |

#### useLogout

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success clears local session via `onSettled` | mock `POST /auth/logout` → success | `mutate()` | `authStore.logout()` is called, clearing `token`/`user` to `null` |
| A failed API call still clears local session | mock `POST /auth/logout` to reject (e.g. an already-expired token, network error) | `mutate()` | `authStore.logout()` is still called — confirms the hook uses `onSettled`, not `onSuccess`, so the person is never stuck unable to log out client-side because of a network error (8-1's explicit design intent) |
| Clears session before/without an imperative `navigate()` call | mock success | `mutate()` | no `navigate()` call is asserted from within the hook itself — the redirect to `/login` is left to `ProtectedRoute`'s next render reacting to `token` becoming `null`, not a side effect the hook performs directly |

#### useRegister(mode)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `mode='student'` posts to the student endpoint with `grade` | mock `POST /auth/register/student` | `mutate({ ...body, grade: 8 })` | request hits the student-specific endpoint, not a generic `/auth/register` |
| `mode='parent'`/`'tutor'` post to their respective endpoints | mock each endpoint | `mutate(body)` per mode | correct endpoint per mode, confirms the "one hook parameterized by mode, not three files" design (8-1) actually routes correctly |
| 409 (already exists) is surfaced as a distinct error shape from a 400 | mock `409` | `mutate(body)` | error's `status === 409`, distinguishable by the calling page for its field-level (not form-level) rendering |

#### useVerifyContact / useResendVerification

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Verify success | mock `POST /auth/verify-contact` success | `mutate({ userId, code })` | resolves; no client-side assumption about what happens next (that's the page's job) |
| Invalid/expired code | mock `400`/`410`-class error | `mutate({ userId, code })` | error propagates, distinguishable from a network failure |
| Resend has no built-in cooldown of its own | mock `POST /auth/resend-verification` | call `mutate(userId)` twice in immediate succession | **both calls actually fire from the hook's perspective** — the hook itself does not throttle; the mocked backend call in this test does not simulate the real server-side rate limit, since that's enforced by `rateLimiter.middleware.ts` (Doc 02 NFR-013, resolved — 3/hour per account), not by anything in this hook or its mock. `VerifyContactPage`'s own 30s UI-disable (tested separately in §9.6) is a UX courtesy layered *in front of* the real server-side control, not a replacement for it. This case exists to confirm the two client-side layers (hook, page) aren't confused with each other or with the (real, server-side, out of this file's scope) rate limit — see §9.10 |

#### useForgotPassword / useResetPassword

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Forgot-password always resolves the same way regardless of identifier validity | mock the backend's generic always-success response for both a real and a fake identifier | `mutate(identifier)` for each | **identical** resolved value/shape in both cases — the hook must not itself introduce a branch (e.g. by inspecting a debug header) that would let `ForgotPasswordPage` render a different message for the two cases, which would undo the backend's deliberate non-disclosure design (8-1 §1.7) |
| Reset-password invalid/expired code | mock `400`/`410`-class error | `mutate({ userId, code, newPassword })` | error propagates as field-level, not form-level |

---

### 9.6 Test Case Detail — LoginPage.test.tsx, RegisterPage.test.tsx, VerifyContactPage.test.tsx, ForgotPasswordPage.test.tsx, ResetPasswordPage.test.tsx

**OWASP: A03:2021 – Injection (input rendering), A07:2021, A04:2021.**

#### LoginPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Client-side validation blocks an empty submit | render page, leave both fields blank | click submit | `useLogin().mutate` is **never called** — Zod catches it first |
| `returnTo` redirect on success | render at `/login?returnTo=%2Ftutor%2Fearnings`, mock `useLogin` to succeed | submit valid credentials | `navigate('/tutor/earnings')` called — the decoded path, not the raw encoded string |
| No `returnTo` — falls back to role-default route | render at plain `/login`, mock success with `user.role = 'PARENT'` | submit | `navigate(roleDefaultRoute('PARENT'))` |
| 401 renders a form-level banner, form re-enabled | mock `useLogin` to reject with the generic 401 | submit | banner visible; inputs and submit button are not left disabled/stuck |
| Submitting state disables the button and shows a spinner | mock `useLogin` as pending | render | submit button `disabled`, spinner visible — prevents a double-submit race, distinct from any backend idempotency concern |

#### RegisterPage (parameterized over `mode`)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `mode='student'` requires `grade` in the 6–12 range client-side | render with `mode="student"` | enter `grade=3` and submit | rejected client-side before any `mutate` call — mirrors the backend's `register*Schema` per role (8-1); this is a UX guard, and a mismatched or looser client check here would let a person submit a request the backend will reject anyway with a worse error experience, not a security bypass, since the backend re-validates independently |
| `termsAccepted` is mandatory across all three modes | render with any `mode` | submit with `termsAccepted: false` | blocked client-side |
| 409 renders as a field-level error on email/phone, not a full-page error | mock `useRegister` to reject with `409` | submit | error appears anchored to the identifier field specifically |
| Success with `verificationRequired: true` navigates to verify-contact carrying `userId` in route state | mock success response | submit | `navigate('/verify-contact', { state: { userId } })` |
| Success without verification required navigates straight to login | mock success response, `verificationRequired: false` | submit | `navigate('/login')` |

#### VerifyContactPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `userId` resolved from route state first, query param fallback second | render with route `state: { userId }` vs. render with only `?userId=` in the URL (simulating a bookmarked SMS link, per 8-1) | submit a code | both cases call `mutate({ userId, code })` with the correct value — confirms neither path is silently dropped |
| Resend button disables for 30s after each press (fake timers) | mock `useResendVerification`, use `vi.useFakeTimers()` | click "Resend" | button becomes disabled immediately; `vi.advanceTimersByTime(29_999)` → still disabled; `vi.advanceTimersByTime(1)` → re-enabled — this is asserted explicitly as a **client-side UX courtesy layered in front of the real server-side rate limit** (Doc 02 NFR-013, resolved), not a substitute for it; see §9.10 |
| Success redirects with a toast | mock success | submit | `navigate('/login')`, success toast rendered |

#### ForgotPasswordPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Always shows the same generic success message | mock `useForgotPassword` to resolve (the hook itself never distinguishes outcomes, per §9.5) | submit with any identifier | the exact same generic copy renders every time — **there is no error-state branch to test here at all**, and a test suite that added one testing "identifier not found shows an error" would itself be reintroducing the disclosure bug this page is designed to prevent (OWASP A01:2021-adjacent: user enumeration via password-reset response difference) |

#### ResetPasswordPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Invalid/expired code renders a field-level error | mock rejection | submit | error anchored to the code field |
| Success redirects to login | mock success | submit | `navigate('/login')` |

---

### 9.7 Test Case Detail — PolicyPage.test.tsx

FRs: FR-SC-001. **OWASP: A03:2021 – Injection (XSS via rendered markdown).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid `:type` fetches and renders | render at `/policies/PRIVACY` | — | `usePolicy('PRIVACY')` called; content rendered |
| Invalid `:type` never fires the request | render at `/policies/NOT_A_TYPE` | — | `usePolicy` is **not** called at all — the route param is validated against the 5-value enum client-side before querying, per 8-1; `NotFoundPage` content renders inline instead |
| Rendered markdown content is sanitized before insertion | mock `usePolicy` to return content containing a `<script>`/`onerror=`-bearing payload (simulating a compromised or malformed `PolicyDocument.content` value) | render | the markdown renderer strips/escapes the executable payload — no `<script>` tag or inline event handler survives into the rendered DOM; asserted via `container.innerHTML` inspection or a mocked renderer call showing sanitization was applied, not merely that the page "didn't crash." This is the one place in the app rendering admin-authored rich content to every visitor including logged-out users, making it the highest-value XSS test in this feature (OWASP A03:2021) |

---

### 9.8 Test Case Detail — NotificationsPage.test.tsx, NotificationBell.test.tsx, AnnouncementsPage.test.tsx

**OWASP: A01:2021 – Broken Access Control (Admin-only announcements composer).**

#### NotificationsPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Empty list renders `EmptyState`, not an error | mock `useMyNotifications` → `{ notifications: [] }` | render | `EmptyState`, no error banner — matches API convention §0.3 |
| Opening an unread row marks it read as a side effect | mock an unread notification row | click the row (not the explicit "mark read" button) | `useMarkRead().mutate(id)` called — confirms the side-effect-on-open behavior from 8-1, not only the explicit button path |
| Unread filter toggles the query param | render, toggle "unread only" | — | `useMyNotifications(true, page)` re-invoked with the flag flipped |

#### NotificationBell

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Polls every 30s | mock `useMyNotifications`, fake timers | mount, advance 30_000ms | query re-fires (or `refetchInterval: 30_000` is asserted on the hook call options) |
| Badge count caps display at "9+" | mock unread count = 14 | render | badge text is exactly `"9+"`, not `"14"` |
| Clicking a dropdown entry marks read then navigates per notification type | mock a `NEW_MESSAGE`-type entry | click it | `useMarkRead().mutate(id)` called, then `navigate('/messaging')` — via the `notificationTypeToRoute` util, tested here through the component's observable navigation rather than by importing the util directly (both are acceptable; this row asserts the integration) |

#### AnnouncementsPage (Admin)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Route requires Admin (defense already covered by ProtectedRoute §9.2, re-asserted at the page level for defense-in-depth) | render `AnnouncementsPage` directly with a non-Admin `auth.store` state, bypassing the router | — | the page itself renders nothing sensitive / redirects rather than assuming the router-level guard is the only defense — a genuine defense-in-depth check, not a redundant duplicate, since a future routing refactor that accidentally drops the `ProtectedRoute` wrapper should not silently expose this page |
| `audienceRoles` cannot be empty on submit | render, select a title/body but leave `audienceRoles` unselected | click submit | submit blocked client-side — mirrors 8-1's explicit note that the backend would otherwise silently accept a no-op send |
| Successful creation invalidates the announcement list | mock `useCreateAnnouncement` success | submit | `[ANNOUNCEMENTS]` query key invalidated, list re-fetches |

---

### 9.9 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `ProtectedRoute`'s `returnTo` value is asserted as URL-encoded and path-relative in the actual navigation call args, not inferred from "the redirect happened."
- [ ] The `useLogin` wrong-password vs. unknown-identifier test asserts identical error objects (deep-equal), not just "both truthy/both 401."
- [ ] `ForgotPasswordPage`'s test suite contains **no** test asserting different behavior for a valid vs. invalid identifier — presence of such a test is itself a regression signal, not a coverage gain.
- [ ] `auth.store.test.ts`'s persistence check inspects the actual serialized payload written to the storage backend, not just the in-memory store state, to catch a future persist-middleware misconfiguration that writes more than intended.
- [ ] `PolicyPage`'s XSS case uses a real sanitization-bypass-shaped payload (a `<script>` tag or an `onerror` attribute), not a plain string, so the test would actually fail if sanitization were removed.
- [ ] `VerifyContactPage`'s resend-throttle test and `useAuth.test.ts`'s resend-hook test are not merged into one — they intentionally test two different layers (UI courtesy vs. hook has-no-throttle-of-its-own), and collapsing them would hide a regression in either.
- [ ] `NotificationBell`'s "9+" cap is tested with a count strictly greater than 9 (not exactly 9 or 10 only) to rule out an off-by-one in the cap condition.

---

### 9.10 Out of Scope for Automated Testing (and why)

- **`PublicLayout.tsx`, `AuthLayout.tsx`, `DashboardLayout.tsx`** — no conditional logic of their own beyond composing children and selecting a sidebar component by `role` (a single property lookup, not a decision worth its own suite); the sidebar-selection behavior itself is exercised indirectly by every `ProtectedRoute` test that renders through a real layout tree, and directly by each feature's own sidebar file where relevant.
- **`TopNavBar.tsx`, `Footer.tsx`** — presentational; active-link styling is a CSS-class derivation from `useLocation()` with no branch that could silently break a business rule.
- **Client-side resend-verification throttle as a security control** — this 30-second UI disable in `VerifyContactPage` is tested (§9.6) as a UX courtesy only. On its own it provides **no actual rate-limiting** — a person can trivially call the underlying endpoint directly (devtools, a script, a second tab) to bypass it entirely, and §9.5's `useResendVerification` test deliberately proves the hook itself has no throttle. **Correction (Phase 0, testing-redesign):** an earlier version of this note claimed the backend also has no server-side rate limit here, mirroring a supposed gap in `9-1-shared-config.md` §9.19. That claim was wrong. Doc 02 NFR-013 is resolved and specifies exact server-side limits — 3/hour per account for resend-verification, 3/hour per account+IP for forgot-password, 5 attempts/15 min per identifier+IP for login — enforced by `rateLimiter.middleware.ts` and covered by real test cases in `9-1-shared-config.md` §9.19. The backend doc was correct; this doc's prior claim of an open vulnerability was not. This frontend doc's UI-disable test is still worth keeping exactly as specified — it is a genuine UX layer, distinct from and in front of the real server-side control — it just must not be read as evidence that no server-side control exists.
- **Real markdown-rendering library internals** — `PolicyPage`'s sanitization test (§9.7) verifies the *integration* (a malicious payload does not survive into the DOM); it does not re-test the sanitization library's own correctness, which is that library's own test suite's responsibility.
- **Visual regression / pixel-level styling** — out of scope for Vitest/RTL; if introduced later, that's a separate tool (e.g. Storybook + Chromatic), not this test suite.
- **Real Chapa/Jitsi/Geez SMS/Brevo network behavior** — not applicable to this feature file; see `9-7-payments-earnings.md` and `9-4-class-delivery-library.md` for the features that actually touch those integrations client-side (link submission, checkout redirect).
- **CSRF** — not applicable; stateless Bearer-JWT, no cookie session, matching the backend test doc's same note.

---

**Next:** proceed to → [9-2. Frontend Test Documentation: Accounts & Guardianship]
