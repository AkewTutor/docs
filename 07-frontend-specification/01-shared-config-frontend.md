## Project: AKEWTutor — Frontend Specification
**Feature:** Shared Config (Auth, Notifications, Policies, Announcements)
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `01-shared-config-api.md`

**Owns UI for:** registration/login, notifications, policy pages, platform announcements (Doc 05b §1). No backend dependency (foundation feature).

---

### 1.1 Routes

```
/                    → PublicLayout → LandingPage
/policies/:type      → PublicLayout → PolicyPage
/login               → PublicRoute → AuthLayout → LoginPage
/register/student    → PublicRoute → AuthLayout → RegisterPage (mode="student")
/register/parent     → PublicRoute → AuthLayout → RegisterPage (mode="parent")
/register/tutor      → PublicRoute → AuthLayout → RegisterPage (mode="tutor")
/verify-contact      → PublicRoute → AuthLayout → VerifyContactPage
/forgot-password     → PublicRoute → AuthLayout → ForgotPasswordPage
/reset-password      → PublicRoute → AuthLayout → ResetPasswordPage
/notifications       → ProtectedRoute('any') → DashboardLayout → NotificationsPage
/admin/announcements → ProtectedRoute(['ADMIN']) → DashboardLayout → AnnouncementsPage
```

`RegisterPage.tsx` is one component parameterized by route (`mode` derived from the path), not three separate files — the three registration bodies differ only in a few fields (grade for student, none extra for parent, none extra for tutor) and all three hit `POST /auth/register/:role` (01-shared-config-api.md §1.2).

### 1.2 Types (added to src/types/index.ts)

```typescript
export type Role = 'STUDENT' | 'PARENT' | 'TUTOR' | 'ADMIN';

export interface AuthUser {
  id: string;
  role: Role;
  email: string | null;
  phone: string | null;
}

export interface LoginResponse {
  accessToken: string;
  user: AuthUser;
}

export interface RegisterResponse {
  userId: string;
  role: Role;
  // one of, depending on role:
  studentProfileId?: string;
  parentProfileId?: string;
  tutorProfileId?: string;
  grade?: number;               // student only
  accountStatus?: string;       // student only
  onboardingStatus?: string;    // parent only
  verificationStatus?: string;  // tutor only
  verificationRequired: boolean;
}

export interface AppNotification {
  id: string;
  type: string; // CLASS_REMINDER | NEW_MESSAGE | COMPLAINT_RESOLVED | etc.
  payload: Record<string, unknown>;
  channel: 'PUSH' | 'EMAIL' | 'SMS';
  status: string;
  sentAt: string | null;
  readAt: string | null;
  createdAt: string;
}

export interface PolicyDocument {
  type: 'PRIVACY' | 'TERMS' | 'SAFETY' | 'REFUND' | 'RULES';
  version: number;
  content: string; // markdown
  publishedAt: string;
}

export interface Announcement {
  id: string;
  title: string;
  audienceRoles: Role[];
  createdAt: string;
}
```

### 1.3 Store (src/store/auth.store.ts)

Per conventions §0.3 — the single Zustand store for all four roles:

```typescript
interface AuthState {
  token: string | null;
  user: AuthUser | null;
  setAuth: (token: string, user: AuthUser) => void;
  logout: () => void;
}
```

Persisted under key `auth-storage`. `logout()` clears both fields; the actual `POST /auth/logout` call and the store clear happen together in `useLogout` below (mirroring the pattern already used for the sibling reference project's `useAdminLogout` — API call and local-state clear live in one hook, not split across two call sites).

### 1.4 Hooks (src/hooks/useAuth.ts)

```typescript
export function useRegister(role: 'student' | 'parent' | 'tutor') {
  return useMutation({
    mutationFn: (body: Record<string, unknown>) =>
      api.post<RegisterResponse>(`/auth/register/${role}`, body).then((r) => r.data),
  });
}

export function useLogin() {
  const setAuth = useAuthStore((s) => s.setAuth);
  return useMutation({
    mutationFn: (creds: { identifier: string; password: string }) =>
      api.post<LoginResponse>('/auth/login', creds).then((r) => r.data),
    onSuccess: (data) => setAuth(data.accessToken, data.user),
  });
}

export function useLogout() {
  const logout = useAuthStore((s) => s.logout);
  return useMutation({
    mutationFn: () => api.post('/auth/logout'),
    onSettled: () => logout(), // clear local state even if the call itself fails
  });
}

export function useVerifyContact() {
  return useMutation({
    mutationFn: (body: { userId: string; code: string }) => api.post('/auth/verify-contact', body),
  });
}

export function useResendVerification() {
  return useMutation({
    mutationFn: (userId: string) => api.post('/auth/resend-verification', { userId }),
  });
}

export function useForgotPassword() {
  return useMutation({
    mutationFn: (identifier: string) => api.post('/auth/forgot-password', { identifier }),
  });
}

export function useResetPassword() {
  return useMutation({
    mutationFn: (body: { userId: string; code: string; newPassword: string }) =>
      api.post('/auth/reset-password', body),
  });
}
```

### 1.5 Hooks (src/hooks/useNotifications.ts)

```typescript
export function useMyNotifications(unreadOnly = false, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.NOTIFICATIONS, unreadOnly, page],
    queryFn: () => api.get<{ notifications: AppNotification[]; page: number; limit: number; total: number }>(
      '/notifications', { params: { unreadOnly, page, limit: 20 } }
    ).then((r) => r.data),
  });
}

export function useMarkRead() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => api.patch(`/notifications/${id}/read`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.NOTIFICATIONS] }),
  });
}
```
`NotificationBell.tsx` polls `useMyNotifications(true)` (unread only) at a light interval (`refetchInterval: 30_000`) rather than a websocket, matching the template's existing polling convention used elsewhere for job-driven state (e.g. the sibling reference project's `usePipelineSources`).

### 1.6 Hooks (src/hooks/usePolicy.ts, src/hooks/useAdminAnnouncements.ts)

```typescript
export function usePolicy(type: PolicyDocument['type']) {
  return useQuery({
    queryKey: [QUERY_KEYS.POLICY, type],
    queryFn: () => api.get<PolicyDocument>(`/policies/${type}`).then((r) => r.data),
  });
}

export function useCreateAnnouncement() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { title: string; body: string; audienceRoles: Role[] }) =>
      api.post<Announcement>('/admin/announcements', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ANNOUNCEMENTS] }),
  });
}

export function useAnnouncements(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.ANNOUNCEMENTS, page],
    queryFn: () => api.get<{ announcements: Announcement[]; page: number; limit: number; total: number }>(
      '/admin/announcements', { params: { page, limit: 20 } }
    ).then((r) => r.data),
  });
}
```

### 1.7 Components & Pages

| File | Responsibility |
|---|---|
| `LoginPage.tsx` | Single shared login form for all four roles; on success, redirects to `returnTo` (§0.6) or a role-based default landing (`/student`, `/parent`, `/tutor`, `/admin`) |
| `RegisterPage.tsx` | Mode-driven registration form; student mode adds a `grade` field (6–12 only — the API itself rejects 1–5, per `01-shared-config-api.md`'s 400 case, but the form should pre-empt it client-side too) |
| `VerifyContactPage.tsx` | Code entry + "resend" action wired to `useResendVerification` |
| `ForgotPasswordPage.tsx`, `ResetPasswordPage.tsx` | Standard two-step reset flow; `ForgotPasswordPage` always shows a generic success message regardless of whether the identifier matched an account, mirroring the API's deliberate non-disclosure (01-shared-config-api.md §1.2) |
| `PolicyPage.tsx` | Renders `usePolicy(type)`'s markdown `content`; `type` comes from the route param, validated against the five allowed values before the request fires |
| `NotificationsPage.tsx` | Full notification list with pagination + mark-read; `NotificationBell.tsx` is the header-level unread indicator/dropdown version of the same data |
| `AnnouncementsPage.tsx` (admin) | Compose form (`title`, `body`, `audienceRoles` multi-select) + list of previously sent announcements |

### 1.8 Form Validation Notes

- `RegisterPage` Zod schema mirrors each backend `register*Schema` (min-8-char password, `termsAccepted: true` required) so the common case is caught client-side — but a `409` ("account already exists") is a legitimate server-side-only case and must render as a field-level or form-level error, not be assumed unreachable.
- `LoginPage`'s `401` ("Invalid email/phone or password") is deliberately generic per the API — the form must not attempt to guess or display which field was wrong.

---

**Next:** proceed to → [02. Accounts & Guardianship Frontend]
