## Project: AKEWTutor — Frontend Specification
**Links back to:** [02. Requirements], [03. Use Cases], [04. Database & Data Model], [05b. Frontend Folder & File Structure], [06. API Specification], [10. UI Foundation Spec]
**Built on:** template-react (React 19, Vite, TypeScript, Tailwind v4, shadcn/ui, TanStack Query, Axios, Zustand, React Router v7, React Hook Form + Zod) — the same base template Doc 05b names, so nothing about the template's plumbing (Axios contract shape, TanStack Query patterns, RHF+Zod forms) is re-litigated here; only AKEWTutor-specific adaptations are.

Split into feature-based files, mirroring the same 8-feature grouping used throughout Docs 04, 05a/05b, 06, and Feature Decomposition:

- **0-frontend-conventions.md** — this file
- **01-shared-config-frontend.md** — Shared Config (Auth, Notifications, Policies, Announcements)
- **02-accounts-guardianship-frontend.md** — Accounts & Guardianship
- **03-matching-cohorts-frontend.md** — Matching & Cohorts
- **04-class-delivery-library-frontend.md** — Class Delivery, Recording & Library
- **05-messaging-frontend.md** — In-Platform Messaging
- **06-gamification-engagement-frontend.md** — Gamification & Engagement
- **07-payments-earnings-frontend.md** — Payments & Earnings
- **08-support-trust-admin-frontend.md** — Support, Trust & Admin Reporting

**`ui-foundation` is deliberately not a 9th numbered file here.** Doc 05b §0 already states design tokens, primitives, and common components are fully specified in `10-ui-foundation-spec.md`, and 05b only lists *what* foundation files exist, not what they contain. This folder does the equivalent job for features 1–8 (hook signatures, route wiring, component responsibilities, form/validation behavior) — it does not restate the foundation layer a second time.

---

### 0.1 The Core Adaptation: Auth-First, Four Roles — Not Public-First, Single-Role

The base template assumes a binary world: logged in or not, one kind of authenticated user. AKEWTutor is closer to that than a fully public app, but with a real wrinkle the template doesn't anticipate: **four distinct authenticated roles** (Student, Parent, Tutor, Admin) sharing one login mechanism, plus a thin public layer in front of all of it (Doc 01 §1.5 — marketing site, standalone informational pages).

**Four route categories, not the template's two:**

| Category | Guard | Layout | Applies to |
|---|---|---|---|
| Public (no auth concept) | None | `PublicLayout` | Landing/marketing pages, `GET /policies/:type` pages (Privacy/Terms/Safety/Refund/Rules) |
| Guest action (unauthenticated, sometimes token-scoped) | `PublicRoute` (redirect to caller's dashboard if already logged in) | `AuthLayout` | Login, Register (student/parent/tutor — three separate registration pages, one shared login page), Verify Contact, Forgot/Reset Password, Guardian Invite Activation |
| Authenticated — role-exclusive | `ProtectedRoute(roles: Role[])` | `DashboardLayout` | Pages under a single role's own area (Student, Parent, Tutor, or Admin) |
| Authenticated — shared across roles | `ProtectedRoute(roles: 'any')` | `DashboardLayout` | Notifications, Messaging, Library, Payments, Support/Complaints — reachable by more than one role, gated only on "logged in," not a specific role |

**One deliberate divergence from the sibling reference project's pattern:** that project's Admin area used its own `AdminLoginPage` at `/admin/login` because it was the *only* authenticated role in that app. AKEWTutor has one shared `LoginPage.tsx` (Doc 05b §1) backing `POST /auth/login`, which authenticates any of the four roles (01-shared-config-api.md §1.2) — there is no separate admin login route. `ProtectedRoute` determines *what the person can see after* logging in; it doesn't determine *where they log in*.

---

### 0.2 Routing Structure

```
/                                → PublicLayout → LandingPage
/policies/:type                 → PublicLayout → PolicyPage
/login                           → PublicRoute → AuthLayout → LoginPage
/register/student                → PublicRoute → AuthLayout → RegisterPage (grade 6-12 flow)
/register/parent                 → PublicRoute → AuthLayout → RegisterPage (parent flow)
/register/tutor                  → PublicRoute → AuthLayout → RegisterPage (tutor flow)
/verify-contact                  → PublicRoute → AuthLayout → VerifyContactPage
/forgot-password                 → PublicRoute → AuthLayout → ForgotPasswordPage
/reset-password                  → PublicRoute → AuthLayout → ResetPasswordPage
/invite/:token/activate          → PublicRoute → AuthLayout → InviteActivationPage
/student/*                       → ProtectedRoute(['STUDENT']) → DashboardLayout(StudentSidebar)
/parent/*                        → ProtectedRoute(['PARENT'])  → DashboardLayout(ParentSidebar)
/tutor/*                         → ProtectedRoute(['TUTOR'])   → DashboardLayout(TutorSidebar)
/admin/*                         → ProtectedRoute(['ADMIN'])   → DashboardLayout(AdminSidebar)
/notifications                   → ProtectedRoute('any')       → DashboardLayout → NotificationsPage
/messaging                       → ProtectedRoute('any')       → DashboardLayout → MessagingPage
/library                         → ProtectedRoute('any')       → DashboardLayout → LibraryPage
/payments, /payments/history     → ProtectedRoute('any')       → DashboardLayout → PaymentPage, PaymentHistoryPage
/support, /complaints            → ProtectedRoute('any')       → DashboardLayout → SupportContactPage, SubmitComplaintPage
*                                → NotFoundPage (no layout, per template default)
```

Per the template's own rule (already carried into Doc 05b's structuring principle): every route is nested under the correct guard **and** the correct layout — never bare, and never a role-exclusive page reachable through a shared-path route.

> ✅ **M2 fix — resolved, not just flagged:** Doc 05b's file inventory now lists `ParentSidebar.tsx` alongside `StudentSidebar.tsx`/`TutorSidebar.tsx`/`AdminSidebar.tsx` as a `DashboardLayout` dependency. Parent is a fourth distinct role with materially different pages (guardianship management, a different payment-history scope, no subject-ranking or availability screens), so it gets its own sidebar file rather than routing Parent through `StudentSidebar` with conditional items — specified in `02-accounts-guardianship-frontend.md` where the Parent-exclusive pages live.

---

### 0.3 State Management Split

- **Zustand:** only `auth.store.ts` (Doc 05b §1) — a single store for all four roles, not one per role, persisted under key `auth-storage`. It holds the JWT and the caller's `{ id, role, email/phone }`, from which every `ProtectedRoute` and every sidebar's role-check reads. No other Zustand store exists — feature-local UI state (a selected tab, an open panel, a filter) is `useState`/URL params per page, matching the same principle already established for the reference project.
- **TanStack Query:** every server call, per the template's standard pattern — one hook file per feature (`useAuth.ts`, `useMatching.ts`, etc.), matching Doc 05b's hook inventory exactly. Query keys are centralized in `src/constants/index.ts` `QUERY_KEYS`, one entry per queryable resource (Doc 05b §9).

---

### 0.4 API Layer

Uses the template's `src/lib/axios.ts`, largely unchanged — AKEWTutor's `SuccessResponse`/`ErrorResponse` envelope (00-api-conventions.md §0.1) matches the template's expected contract exactly, so the interceptor's unwrap logic requires no modification.

`VITE_API_URL` is set to the backend's base path, e.g. `http://localhost:3000/api/v1`, matching `00-api-conventions.md`'s `/api/v1` base path.

**One addition beyond the base template, and a divergence from the sibling reference project:** in that project, only Admin calls carried a Bearer token (everything else was public). Here, the *majority* of endpoints require a token — the interceptor attaches it from `auth.store.ts` on every request when present, and only the small `Public`-labeled set (registration, login, policy reads, the Chapa webhook) is ever called without one. There is no separate unauthenticated Axios instance; the same `api` client is used everywhere, since an absent token is a non-issue on the genuinely public endpoints.

---

### 0.5 Types & Query Keys

`src/types/index.ts` gets one interface block per feature, field names and types copied directly from each `06-api/0X-*.md` file's example JSON, not re-derived — e.g. `StudentProfile`, `TutorProfile`, `Cohort`, `ScheduledSession`, `MessageThread`, `XPLedgerEntry`, `Payment`, `ComplaintReport` (full list per feature in Doc 05b §9).

`src/constants/index.ts` `ROUTES` gets one entry per page listed in Doc 05b, grouped by feature comment blocks, matching the routing structure in §0.2 above exactly — a page's `ROUTES` constant and its actual route path must never drift, since sidebars build their nav links from `ROUTES`, not from hardcoded strings.

---

### 0.6 401 Handling — Divergence from Both the Base Template and the Sibling Reference Project

The base template hard-redirects to `/login` on any `401`. The sibling reference project narrowed that to "only redirect if the failing URL is under `/admin/`" because it had exactly one authenticated role. **AKEWTutor has four**, so neither rule fits as-is.

Because there is a single shared `LoginPage.tsx` (see §0.1) rather than per-role login pages, the interceptor's job on a `401` is not to pick *which* login page to send someone to — there is only one. Its job is to:
1. Clear `auth.store.ts` (token + user).
2. Redirect to `/login?returnTo=<currentPath>`, so `LoginPage` can send the person back to whatever role-scoped or shared page they were on after re-authenticating.

**M8 fix — resolved, not just flagged:** Doc 05b's file inventory previously described this as "redirects to the correct login route based on the failing request's role prefix," which read as multiple login routes. Corrected directly in `05b-frontend-structure.md`: there's no branching between multiple login *routes*, only a `returnTo` param carried through the single route that exists — matching the behavior described here exactly. Both docs now describe the same single `/login?returnTo=` design.

---

### 0.7 Component Split (per Doc 05b, restated as the working convention for Docs 01–08 below)

- `components/ui/` — shadcn/ui primitives, untouched, no app logic
- `components/common/` — shared, feature-agnostic pieces (`StatusBadge`, `EmptyState`, `CountdownTimer`, `TopNavBar`, `Footer`, `NotificationBell`) — per `10-ui-foundation-spec.md` and Doc 05b §1
- `components/layouts/` — `PublicLayout`, `AuthLayout`, `DashboardLayout` (the last composing `StudentSidebar` / `ParentSidebar` / `TutorSidebar` / `AdminSidebar` based on `auth.store.ts`'s `role` — `ParentSidebar` per the M2 fix in §0.2 above, no longer a flagged open question)
- `components/<feature>/` and `components/admin-<feature>/` — one folder per feature for feature-specific and admin-specific components respectively (Doc 05b already separates these, e.g. `components/matching/` vs `components/admin-matching/`), carried forward unchanged in Docs 01–08 below.

Each of Docs 01–08 below specifies, per its feature: exact routes, TypeScript interfaces, hook signatures (query keys, mutation shapes, `enabled`/`refetchInterval` behavior), component responsibilities, and any form validation that mirrors a backend business rule beyond basic Zod shape — the same level of detail already modeled in Docs 05-messaging-api.md and 06-gamification-engagement-api.md's endpoint-detail sections, translated to the frontend side of the same contract.

---

**Next:** proceed to → [01. Shared Config Frontend]
