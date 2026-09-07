## Project: AKEWTutor — Frontend Specification
**Feature:** Gamification & Engagement
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `06-gamification-engagement-api.md`

**Depends on:** Accounts & Guardianship (hard); soft-integrates with Class Delivery & Library (XP-award trigger on class-attended — no client-facing coupling at all, since the award happens entirely server-side).

**Links back to:** [0. Frontend Conventions], [06-api/06-gamification-engagement-api.md], [05b. Frontend Folder & File Structure §6]
**Links forward to:** [8-6. Frontend Function-Level Spec: Gamification & Engagement]

---

### 6.1 Routes

```
/student/achievements  → ProtectedRoute('any') → DashboardLayout → AchievementsPage
/student/leaderboard    → ProtectedRoute(['STUDENT','PARENT']) → DashboardLayout → LeaderboardPage
/student/challenges      → ProtectedRoute('any') → DashboardLayout → ChallengesPage
/admin/badges             → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → BadgeManagementPage
/admin/challenges          → ProtectedRoute(['ADMIN']) → DashboardLayout(AdminSidebar) → ChallengeManagementPage
```

**I2 fix:** `BadgeManagementPage` now includes a "Create badge" action (`useCreateBadge`) alongside the existing criteria/active-status adjustment (`useAdjustBadge`), and an "Adjust student XP" panel (`useAdjustStudentXP`, `studentId` entered directly since there is no dedicated per-student admin detail page elsewhere in the spec to launch this from — UC-88) — no new route was added; both capabilities live on the existing `/admin/badges` page since it is already the general Admin-gamification surface.

**H3 fix:** `AchievementsPage` and `ChallengesPage` are now reachable by Parent as well as Student (previously Student-only, which left FR-SP-014/UC-14's "Parent can view the student dashboard's XP/streaks/achievements" requirement with no actual page or endpoint to reach). Both pages resolve which student's data to show the same way: `STUDENT` gets their own with no param; `PARENT` resolves an active child via a `studentId` selector (single linked child auto-selected, multiple children get a dropdown — same one-per-child selector pattern used for payments, Doc 07 §7.6) and passes `studentId` through to every hook in this feature. `LeaderboardPage` was already Parent-reachable before this fix and needed no change.

### 6.2 Types (added to src/types/index.ts)

```typescript
export interface XPProgress {
  totalXP: number;
  streak: { currentStreakDays: number; longestStreakDays: number; lastActivityDate: string };
  recentEntries: { amount: number; reason: string; createdAt: string }[];
}

export interface LeaderboardEntry { rank: number; displayName: string; xp: number }

export interface Leaderboard {
  grade: number;
  period: 'WEEKLY' | 'MONTHLY';
  rankings: LeaderboardEntry[];
  callerRank: number;
}

export interface Badge {
  badgeId: string;
  name: string;
  description: string;
  earnedAt: string;
}

export interface AdminBadge {
  id: string;
  name: string;
  category: 'STUDENT' | 'TUTOR';
  criteriaDescription: string;
  isActive: boolean;
}

// I2 fix: previously missing — the resolved-audit shape of a manual Admin XP adjustment (UC-88).
export interface XPAdjustment {
  id: string;
  studentId: string;
  amount: number;
  reason: 'OTHER';
  note: string;
  createdAt: string;
}

export interface Challenge {
  id: string;
  title: string;
  period: 'WEEKLY' | 'MONTHLY';
  startsAt: string;
  endsAt: string;
  targetValue: number;
}

export interface ChallengeProgress { challengeId: string; progressValue: number; completedAt: string | null }
```

### 6.3 Hooks (src/hooks/useGamification.ts)

```typescript
export function useMyProgress(studentId?: string) {
  // H3 fix: studentId is required when called from a Parent-viewed dashboard (FR-SP-014),
  // omitted/ignored server-side for a Student caller — same optional-param shape as useLeaderboard.
  return useQuery({
    queryKey: [QUERY_KEYS.XP_PROGRESS, studentId],
    queryFn: () => api.get<XPProgress>('/gamification/xp/me', { params: { studentId } }).then((r) => r.data),
  });
}

export function useLeaderboard(period: 'WEEKLY' | 'MONTHLY', studentId?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.LEADERBOARD, period, studentId],
    queryFn: () => api.get<Leaderboard>('/gamification/leaderboard', { params: { period, studentId } }).then((r) => r.data),
  });
}
```

### 6.4 Hooks (src/hooks/useBadges.ts)

```typescript
export function useMyBadges(studentId?: string) {
  // H3 fix: same Parent/studentId shape as useMyProgress.
  return useQuery({
    queryKey: [QUERY_KEYS.MY_BADGES, studentId],
    queryFn: () => api.get<{ badges: Badge[] }>('/gamification/badges/me', { params: { studentId } }).then((r) => r.data),
  });
}
```

### 6.5 Hooks (src/hooks/useAdminGamification.ts)

```typescript
export function useAllBadges(category?: 'STUDENT' | 'TUTOR', page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.ADMIN_BADGES, category, page],
    queryFn: () => api.get('/admin/badges', { params: { category, page, limit: 20 } }).then((r) => r.data),
  });
}

export function useAdjustBadge() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ badgeId, ...body }: { badgeId: string; criteriaDescription?: string; isActive?: boolean }) =>
      api.patch<AdminBadge>(`/admin/badges/${badgeId}`, body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_BADGES] }),
  });
}

// I2 fix: previously missing — no create path existed alongside useAdjustBadge.
export function useCreateBadge() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { name: string; description: string; category: 'STUDENT' | 'TUTOR'; criteriaDescription: string; isActive?: boolean }) =>
      api.post<AdminBadge>('/admin/badges', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_BADGES] }),
  });
}

// I2 fix: previously missing — awardXP's reason: OTHER (Doc 02 §10 v3.2) had no Admin-facing hook to invoke it,
// and UC-88's leaderboard correction had no real mechanism without this.
export function useAdjustStudentXP() {
  return useMutation({
    mutationFn: ({ studentId, amount, note }: { studentId: string; amount: number; note: string }) =>
      api.post<XPAdjustment>(`/admin/students/${studentId}/xp-adjustments`, { amount, note }).then((r) => r.data),
    // Deliberately does not invalidate QUERY_KEYS.LEADERBOARD or QUERY_KEYS.XP_PROGRESS here — this action is
    // taken from an Admin-only surface that doesn't hold either query; the affected student's own next visit
    // to their dashboard/leaderboard naturally re-fetches fresh data (Section 6.8 non-optimistic-read note).
  });
}
```

### 6.6 Hooks (src/hooks/useChallenges.ts)

```typescript
export function useActiveChallenges() {
  return useQuery({
    queryKey: [QUERY_KEYS.CHALLENGES],
    queryFn: () => api.get<{ challenges: Challenge[] }>('/gamification/challenges').then((r) => r.data),
  });
}

export function useMyChallengeProgress(studentId?: string) {
  // H3 fix: same Parent/studentId shape as useMyProgress/useMyBadges.
  return useQuery({
    queryKey: [QUERY_KEYS.CHALLENGE_PROGRESS, studentId],
    queryFn: () => api.get<{ progress: ChallengeProgress[] }>('/gamification/challenges/me', { params: { studentId } }).then((r) => r.data),
  });
}

export function useCreateChallenge() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { title: string; description: string; period: 'WEEKLY' | 'MONTHLY'; startsAt: string; endsAt: string; targetValue: number }) =>
      api.post<Challenge>('/admin/challenges', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.CHALLENGES] }),
  });
}
```

### 6.7 Components

| File | Responsibility |
|---|---|
| `XPProgressBar.tsx` | Current XP / level display from `useMyProgress` |
| `BadgeGrid.tsx` | Earned badges from `useMyBadges`; `EmptyState` (foundation component) when none earned yet |
| `StreakFlame.tsx` | Current/longest streak; **must not visually imply lost XP or lost badges on a streak reset** — `totalXP` is a separate field and is never reduced by `currentStreakDays` resetting (per the API's explicit note) |
| `LeaderboardTable.tsx` | Grade-scoped ranking; renders `displayName` exactly as returned (first name + last-initial) — **never appends or reconstructs a full last name client-side**, even if the caller happens to know it (e.g. a Parent viewing their own child's row) |
| `ChallengeCard.tsx` | One active challenge + the caller's `progressValue`/`targetValue`, cross-referencing `useActiveChallenges` and `useMyChallengeProgress` by `challengeId` |
| `BadgeForm.tsx` (admin) — **I2 fix** | `name`/`description`/`category`/`criteriaDescription` fields; wired to `useCreateBadge` on submit, and reused (pre-filled, `criteriaDescription`/`isActive` only editable) for `useAdjustBadge` |
| `XPAdjustmentForm.tsx` (admin) — **I2 fix** | `studentId`/`amount`/`note` fields; wired to `useAdjustStudentXP`. `note` is required client-side (matches the API's required `note`, UC-88) and `amount` accepts a signed, non-zero integer — the form does not attempt to preview the resulting leaderboard rank, since the leaderboard is a live server-computed view (Section 6.8) |

### 6.8 Notes on Data Freshness

The leaderboard is computed live from the XP ledger server-side, never denormalized (per the API's design note) — the frontend should treat `useLeaderboard` as a plain, non-optimistic read (no local cache mutation on XP-earning actions elsewhere in the app) since there's no client-side way to predict the *rank* impact of a new XP entry, only the raw amount.

---

**Next:** proceed to → [07. Payments & Earnings Frontend]
