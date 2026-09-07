## Project: AKEWTutor — Frontend Test Documentation: Gamification & Engagement
**Links back to:** [05b §6], [07-06], [8-6. Frontend Function-Level Spec: Gamification & Engagement]
**Conventions:** see `9-1-shared-config.md` for shared setup/mock patterns and OWASP category definitions.

**Depends on:** Accounts & Guardianship (hard); soft-integrates with Class Delivery & Library (server-side XP trigger only — no client-facing coupling, per 8-6).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered |
|---|---|
| hooks/useGamification.ts (useMyProgress, useLeaderboard, useMyBadges, useActiveChallenges, useMyChallengeProgress) | FR-SP-039–040, FR-GA-002–006 |
| hooks/useAdminGamification.ts (useAllBadges, useCreateBadge, useAdjustBadge, useAdjustStudentXP) | FR-GA-005, FR-AD-004, FR-AD-018 — **I2 fix** |
| hooks/useChallenges.ts (useCreateChallenge) | FR-SP-040, FR-GA-004 |
| components/StreakFlame.tsx | FR-GA-006 (streak independence from XP/badges) |
| components/LeaderboardTable.tsx | FR-GA-002 (first-name+last-initial only) |
| components/ChallengeCard.tsx | FR-GA-004 |
| pages/student/AchievementsPage.tsx, ChallengesPage.tsx | FR-SP-014 (H3 fix — Parent-reachable) |

---

### 9.1 Test File Map

| Source file | Test file | Notes |
|---|---|---|
| src/hooks/useGamification.ts | tests/hooks/useGamification.test.ts | mandatory (plain reads — per 8-6's note, no full block needed beyond the query-key/params table) |
| src/hooks/useAdminGamification.ts | tests/hooks/useAdminGamification.test.ts | mandatory |
| src/hooks/useChallenges.ts | tests/hooks/useChallenges.test.ts | mandatory |
| src/pages/student/AchievementsPage.tsx | tests/pages/AchievementsPage.test.tsx | non-trivial: Student-vs-Parent studentId resolution — full block below |
| src/components/StreakFlame.tsx | tests/components/StreakFlame.test.tsx | non-trivial: must not conflate streak reset with lost XP — full block below |
| src/components/XPProgressBar.tsx | — | purely cosmetic level derivation, no server reconciliation — see §9.8 |
| src/components/BadgeGrid.tsx | — | presentational list + `EmptyState`; folded into `AchievementsPage`'s test, no separate file |
| src/pages/student/LeaderboardPage.tsx | tests/pages/LeaderboardPage.test.tsx | non-trivial: period toggle, Parent child-selector |
| src/components/LeaderboardTable.tsx | tests/components/LeaderboardTable.test.tsx | non-trivial: privacy-relevant display-name rule — full block below |
| src/pages/student/ChallengesPage.tsx, components/ChallengeCard.tsx | tests/components/ChallengeCard.test.tsx | non-trivial: cross-referencing two lists, progress freeze after completion |
| src/pages/admin/BadgeManagementPage.tsx | tests/pages/BadgeManagementPage.test.tsx | non-trivial: category filter |
| src/pages/admin/ChallengeManagementPage.tsx | tests/pages/ChallengeManagementPage.test.tsx | non-trivial: `endsAt > startsAt` validation |

---

### 9.2 Test Case Detail — AchievementsPage.test.tsx (full block)

FRs: FR-SP-014 (H3 fix). **OWASP: A01:2021 – Broken Access Control (role-resolved studentId must not allow one Parent to view a different family's child).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Route reachable by Student or Parent | render as `STUDENT`, then as `PARENT` | — | page renders for both — confirms the H3 fix (`ProtectedRoute('any')`, not `['STUDENT']`) is actually reflected in the rendered route guard, not just documented |
| Student role calls hooks with no `studentId` (resolves to self server-side) | render as `STUDENT` | — | `useMyProgress()`/`useMyBadges()` called with no `studentId` argument |
| Parent with exactly one linked child auto-resolves, no selector shown | render as `PARENT` with one `ACTIVE` relationship | — | `useMyProgress(studentId)` called with that child's id automatically; no dropdown rendered |
| Parent with multiple children shows a selector | render as `PARENT` with two+ `ACTIVE` relationships | — | dropdown shown; selecting a different child re-invokes both hooks with the newly selected `studentId` — this is the security-relevant case: a Parent must only ever be able to select among their **own** linked children, never an arbitrary id, so the selector's options are asserted to come exclusively from `useMyRelationships()`'s own list, never a free-text/arbitrary id field |

---

### 9.3 Test Case Detail — StreakFlame.test.tsx (full block)

FRs: FR-GA-006. **OWASP: none directly — a data-integrity/UX-honesty concern, not a security one, but included per this project's "test everything that should be tested" standard.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders `currentStreakDays` and `longestStreakDays` as independent stats | render with `{ currentStreakDays: 0, longestStreakDays: 12 }`, `totalXP: 4500` | render | both streak numbers shown as given |
| A reset streak (`currentStreakDays: 0`) does not alter or hide `totalXP` | render with a reset streak and a large `totalXP` | — | `totalXP` renders unchanged, sourced from its own separate prop — no shared derivation between the two |
| No "you lost your streak" framing that could be misread as lost points | render with `currentStreakDays: 0` | inspect all rendered text | no copy anywhere on the component implies XP or badges were lost — this is the exact edge case 8-6 flags explicitly (these are genuinely independent fields server-side, and a poorly-worded reset message would misinform the student about their actual standing) |

---

### 9.4 Test Case Detail — LeaderboardTable.test.tsx (full block)

FRs: FR-GA-002. **OWASP: A01:2021 – Broken Access Control / privacy — leaderboard identity minimization.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders `displayName` exactly as given, no reconstruction | render with `displayName: "Abebe K."` | — | that exact string renders, unmodified |
| **Never appends or reconstructs a full last name, even when the caller has that information available** | render `LeaderboardTable` from within a Parent's own view of their own child's row (a scenario where the calling Parent genuinely knows the child's full last name from elsewhere in the app) | inspect rendered output and the component's prop type | only `displayName` (the opaque first-name+last-initial string) is ever rendered — the component's prop type carries no separate `lastName` field to splice in, and this test explicitly renders it in the one scenario where a well-meaning "just show the full name since it's their own kid" shortcut might tempt a future contributor; asserting the type-level absence (no `lastName` prop exists at all) makes this a compile-time-reinforced guarantee, not just a runtime check (8-6 / privacy-by-design for the per-grade leaderboard) |
| Caller's own row is highlighted via `rank === callerRank` | render with `callerRank: 3` | — | the row with `rank: 3` has the highlight class; no other row does |

---

### 9.5 Test Case Detail — ChallengeCard.test.tsx (component) / ChallengesPage cross-referencing

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `progress` is looked up by `challengeId`, `undefined` when not started | mock `useActiveChallenges` with 2 challenges, `useMyChallengeProgress` with progress for only 1 | render | the challenge with no matching progress entry renders `progressValue: 0` via `progress?.progressValue ?? 0`, not a crash on `undefined.progressValue` |
| Completed challenge freezes visual progress at 100%, does not continue animating | render with `progress.completedAt` set and a `progressValue` that would exceed `targetValue` if taken literally | — | the bar renders capped/completed, not overshooting past its completed state — 8-6's explicit note that a completed challenge's bar must not continue moving from later unrelated activity |
| Empty `challenges: []` renders the page's own empty state, not a per-card issue | mock `useActiveChallenges` → `[]` | render `ChallengesPage` | page-level empty state ("no active challenges this period"), not an error |

---

### 9.6 Test Case Detail — BadgeManagementPage.test.tsx, ChallengeManagementPage.test.tsx

**OWASP: A01:2021 – Broken Access Control (Admin-only).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Category filter re-queries | change filter to `TUTOR` | — | `useAllBadges('TUTOR', page)` re-invoked |
| Edit form wires to `useAdjustBadge` with only the changed fields | open edit, change `isActive` | submit | `useAdjustBadge().mutate({ id, ...changes })` called |
| Create form wires to `useCreateBadge` — **I2 fix** | open "Create badge," fill `name`/`description`/`category`/`criteriaDescription` | submit | `useCreateBadge().mutate({ name, description, category, criteriaDescription })` called; on success the badge list re-renders (via `[ADMIN_BADGES]` invalidation) |
| XP adjustment form requires a note — **I2 fix** | open "Adjust student XP," enter `studentId`/`amount` but leave `note` empty | attempt submit | `useAdjustStudentXP().mutate` is not called; a validation message is shown |
| XP adjustment form wires through signed amount and note — **I2 fix** | fill `studentId: "s1"`, `amount: -20`, `note: "Correction"` | submit | `useAdjustStudentXP().mutate({ studentId: "s1", amount: -20, note: "Correction" })` called |
| `endsAt` must be after `startsAt`, validated client-side | set `endsAt` earlier than `startsAt` | submit | blocked before `useCreateChallenge().mutate` fires |
| List reuses `useActiveChallenges` (flagged gap acknowledged) | render the page | — | the currently-active list renders as documented; this test doc does not fabricate a "past challenges" test against an endpoint that, per 8-6, may not yet exist — see §9.8 |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `AchievementsPage`'s Parent-selector test asserts the dropdown's option set is built exclusively from `useMyRelationships()`'s own returned children — not merely that *some* dropdown renders — since the entire point of the check is preventing an arbitrary/off-list `studentId` from ever being selectable.
- [ ] `LeaderboardTable`'s "never reconstructs a full name" case is written as a real rendering assertion against a scenario where the temptation to do so is highest (Parent viewing their own child), not a generic pass-through test with an already-short name that wouldn't reveal the bug either way.
- [ ] `StreakFlame`'s "no lost-XP framing" check inspects the actual rendered text content for the *absence* of implicated phrasing, not just that `totalXP` numerically renders correctly — a component could render the right number while still using alarming copy elsewhere on the same card.
- [ ] `ChallengeCard`'s freeze-at-completion case uses a `progressValue` that would visibly exceed 100% if the freeze weren't applied, not a value already capped by coincidence.

---

### 9.8 Out of Scope for Automated Testing (and why)

- **`XPProgressBar.tsx`'s level derivation** — 8-6 is explicit this is a purely cosmetic bucket function with no server-side "level" field to reconcile against; testing its exact thresholds would be testing an arbitrary design choice, not a business rule. A single smoke test confirming it renders `totalXP` without crashing is sufficient and is folded into `AchievementsPage`'s suite rather than given its own file.
- **`BadgeGrid.tsx`** — a presentational grid + `EmptyState`; its empty-state behavior is exercised as part of `AchievementsPage`'s test, not duplicated.
- **Admin "past/ended challenges" listing** — per 8-6, `06-gamification-engagement-api.md` does not currently expose a dedicated endpoint for this; this test doc does not invent test cases against a `GET /admin/challenges` endpoint that may not exist, mirroring the function-level spec's own flag rather than silently assuming the gap has been resolved.
- **XP-award trigger correctness** — entirely server-side, invisible to the frontend by design (8-6's header note); covered by the backend's `9-6-gamification-engagement.md`.

---

**Next:** proceed to → [9-7. Frontend Test Documentation: Payments & Earnings]
