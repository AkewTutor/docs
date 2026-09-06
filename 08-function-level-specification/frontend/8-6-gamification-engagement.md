## Project: AKEWTutor — Frontend Function-Level Spec: Gamification & Engagement
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `06-gamification-engagement-api.md`. **Frontend spec reference:** `06-gamification-engagement-frontend.md`.

**Depends on:** Accounts & Guardianship (hard); soft-integrates with Class Delivery & Library (XP-award trigger, entirely server-side — no client-facing coupling at all, per frontend spec header).

---

### Shared Pattern: Simple Query Hook

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useMyProgress | GET /gamification/xp/me | [XP_PROGRESS] | — |
| useLeaderboard | GET /gamification/leaderboard | [LEADERBOARD, period, studentId] | `studentId` optional (Parent) |
| useMyBadges | GET /gamification/badges/me | [MY_BADGES] | — |
| useAllBadges (admin) | GET /admin/badges | [ADMIN_BADGES, category, page] | `category` optional filter |
| useActiveChallenges | GET /gamification/challenges | [CHALLENGES] | — |
| useMyChallengeProgress | GET /gamification/challenges/me | [CHALLENGE_PROGRESS] | — |

### Shared Pattern: Simple Mutation + Invalidation

| Hook | Endpoint | Invalidates |
|---|---|---|
| useAdjustBadge (admin) | PATCH /admin/badges/:id | [ADMIN_BADGES] |
| useCreateChallenge (admin) | POST /admin/challenges | [CHALLENGES] |

No hook in this feature needs a full block beyond these tables — every one is a plain read or a plain write-then-invalidate, per frontend spec §6.8's note that the leaderboard (and by extension the rest of this feature's data) is a non-optimistic read with no client-side prediction logic anywhere.

---

### src/pages/student/AchievementsPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/achievements` — **H3 fix:** `ProtectedRoute('any')` (was `['STUDENT']`) + `DashboardLayout` — reachable by Student or Parent per FR-SP-014/UC-14, matching the existing shared-route pattern used by `/payments`, `/library`, etc. (Doc 07 §0.2). Sidebar entry point differs by role (`StudentSidebar` links directly; `ParentSidebar` links to the same route with a child selector when the Parent has more than one linked student — same one-per-child pattern as `PaymentReminderBanner`, Doc 07 §7.6). |
| Behavior | 1. For `STUDENT`, calls `useMyProgress()` / `useMyBadges()` with no `studentId` (resolves to self server-side). 2. For `PARENT`, resolves the active child via a `studentId` selector (defaulting to the first linked `ACTIVE` `ParentStudentRelationship` if only one exists, otherwise a dropdown — same selector pattern already needed for multi-child payment views) and passes that `studentId` through to `useMyProgress(studentId)` / `useMyBadges(studentId)`. 3. `useMyProgress()` renders `XPProgressBar` + `StreakFlame`. 4. `useMyBadges()` renders `BadgeGrid`. |

**States:** loading · success (empty sub-states handled per-component below, not at the page level)

### src/components/XPProgressBar.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ totalXP: number }` |
| Behavior | Renders `totalXP` and a level derived from it via a simple, purely presentational bucket function (e.g. `Math.floor(totalXP / levelThreshold)`) — this derivation is cosmetic only; there is no server-side "level" field to reconcile against; if the design later wants levels to be authoritative, that belongs in the backend spec, not invented here. |

### src/components/StreakFlame.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ streak: XPProgress['streak']; totalXP: number }` |
| Behavior | Renders `currentStreakDays` (flame icon, filled proportionally or just numerically) and `longestStreakDays` as a secondary stat. |
| Edge cases | **Must not visually imply lost XP or lost badges when `currentStreakDays` resets to 0** — the component renders `totalXP` (passed as a separate prop, not derived from streak data) unchanged alongside a reset streak, and there is no "you lost your streak" framing that could be misread as "you lost your points" (§6.7's explicit note, since these are genuinely independent fields server-side). |
| Test file | `tests/components/StreakFlame.test.tsx` |

### src/components/BadgeGrid.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ badges: Badge[] }` |
| Behavior | Grid of earned badges (name, description, `earnedAt`); renders `EmptyState` (foundation component) when `badges: []` — "no badges earned yet," not an error. |

---

### src/pages/student/LeaderboardPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/leaderboard` — `ProtectedRoute(['STUDENT','PARENT'])` + `DashboardLayout` |
| Local state | `period: 'WEEKLY' \| 'MONTHLY'` (toggle, default `WEEKLY`); `studentId` passed through only when the caller is a Parent with more than one child (a child-selector dropdown above the table in that case). |
| Behavior | `useLeaderboard(period, studentId)` renders `LeaderboardTable`, with `callerRank` highlighted as the caller's own row. |

**States:** loading · success

### src/components/LeaderboardTable.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ rankings: LeaderboardEntry[]; callerRank: number }` |
| Behavior | Renders `displayName` exactly as returned by the API (first name + last-initial, per the backend's own formatting) and highlights the row whose `rank === callerRank`. |
| Edge cases | **Never appends or reconstructs a full last name client-side**, even where the caller happens to know it (e.g. a Parent viewing their own child's row) — the component's prop type only carries `displayName` as a single opaque string, so there is no separate `lastName` field available to accidentally splice in (§6.7). |
| Test file | `tests/components/LeaderboardTable.test.tsx` |

---

### src/pages/student/ChallengesPage.tsx (new), src/components/ChallengeCard.tsx (new — full block)

| Field | Detail |
|---|---|
| Route | `/student/challenges` — **H3 fix:** `ProtectedRoute('any')` (was `['STUDENT']`) + `DashboardLayout` — Parent-reachable with the same child-selector pattern as `AchievementsPage` above. |
| `ChallengeCard` props | `{ challenge: Challenge; progress: ChallengeProgress \| undefined }` |
| Behavior (page) | 1. Resolves `studentId` per the Student/Parent split described under `AchievementsPage` above (`undefined` for Student, selected child for Parent). 2. `useActiveChallenges()` (no `studentId` — same list for everyone) and `useMyChallengeProgress(studentId)` fired together. 3. Cross-references the two lists by `challengeId` — `progress` passed to each `ChallengeCard` is `progressList.find(p => p.challengeId === challenge.id)`, which may be `undefined` if the student hasn't started that challenge yet. |
| Behavior (card) | Renders `progress?.progressValue ?? 0` against `challenge.targetValue` as a progress bar; renders a completed badge/checkmark once `progress?.completedAt` is non-null, and freezes further visual progress updates at that point (a completed challenge's bar does not continue animating past 100% even if some later, unrelated activity happens to bump a value server-side). |
| Test file | `tests/components/ChallengeCard.test.tsx` |

**States (page):** loading (either query pending) · empty (`challenges: []` — no active challenges this period) · success

---

### src/pages/admin/BadgeManagementPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/badges` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | 1. `useAllBadges(category, page)`, filterable by `STUDENT`/`TUTOR` category. 2. Each row's edit action opens an inline form (`criteriaDescription`, `isActive` toggle) wired to `useAdjustBadge`. |

**States:** loading · success

### src/pages/admin/ChallengeManagementPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/challenges` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Local state | React Hook Form — `title`, `description`, `period`, `startsAt`, `endsAt`, `targetValue`. |
| Behavior | 1. `useActiveChallenges()` lists existing challenges (reused rather than a separate admin-list hook, since the shape is identical and Doc 07 does not define a distinct admin listing endpoint for this — **flagging rather than inventing one**: if Admin needs to see *past/ended* challenges too, not just currently-active ones, `06-gamification-engagement-api.md` would need a dedicated `GET /admin/challenges` endpoint, which does not currently exist; worth confirming with whoever owns Doc 06 whether that gap is intentional for V1). 2. Create form submits via `useCreateChallenge`; `endsAt` must be after `startsAt`, client-validated before submit. |

**States:** loading · success (list + create form)

---

**Next:** proceed to → [8-7. Frontend: Payments & Earnings]
