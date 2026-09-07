## Project: AKEWTutor — Backend Function-Level Spec: Gamification & Engagement
**Conventions:** see `00-api-conventions.md` §0.1–0.7. **API reference:** `06-gamification-engagement-api.md`. **Folder/file reference:** `05a-backend-structure.md` §6.

**Owns:** XPLedgerEntry, Badge, StudentBadge, TutorBadge, Streak, Challenge, ChallengeProgress. **Depends on:** `accounts-guardianship` (hard); soft-integrates with `class-delivery-library` (XP-award trigger on class-attended — no FK, event-based only).

**Links back to:** [06-api/06-gamification-engagement-api.md], [05a. Backend Folder & File Structure §6]
**Links forward to:** [9-6. Backend Test Spec: Gamification & Engagement]

---

### src/services/xp.service.ts (new)

#### awardXP

| Field | Detail |
|---|---|
| Signature | `awardXP(studentId: string, amount: number, reason: XPReason): Promise<XPLedgerEntryDTO>` |
| Purpose | Event-based — called from other features (primarily `class-delivery-library`'s `session.service.ts → markCompleted`, and `weeklyAssessment.service.ts` completion, per the soft-dependency note in Feature Decomposition §1.1) rather than exposed as its own client-facing endpoint. |
| Side effects | Inserts an append-only `XPLedgerEntry` row; never mutates a running total column — `totalXP` is always a live sum over the ledger (Doc 04 §4.0 design decision, matching the leaderboard's own "never a stored/denormalized table" rule below). Also calls `streak.service.ts → updateStreakOnActivity` for the same event. |
| Edge cases | This function has no failure mode that should roll back the triggering action — an XP award failure must never prevent a class from being marked complete. Any error here should be caught and logged by the caller, not propagated as a blocking exception (same philosophy as `notification.service.ts → dispatchNotification`). |

Test file: `tests/services/xp.service.test.ts` — includes per-grade-scoping and name-display cases (verified via `getLeaderboard` below, which is the function that actually surfaces those constraints).

#### getLeaderboard

| Field | Detail |
|---|---|
| Signature | `getLeaderboard(callerId, callerRole, studentId, period: 'WEEKLY' \| 'MONTHLY'): Promise<LeaderboardDTO>` |
| Purpose | Per-grade leaderboard, computed live from the XP ledger — never a stored/denormalized table (Doc 04 §4.0). |
| Side effects | Aggregates `XPLedgerEntry` rows within the period window, grouped by student, filtered to the caller's own `StudentProfile.grade` — grade-scoping is enforced in the query itself, not filtered client-side after a broader fetch (Section 10 Definition of Done #1). |
| Output | `displayName` is always first name + last-initial — never a full last name, computed in this function, never returning a raw last name field for the client to truncate (Section 10 Definition of Done #2). |

Test file: `tests/services/xp.service.test.ts` — includes the per-grade-scoping and name-display cases directly.

#### adminAdjustXP — **I2 fix**

| Field | Detail |
|---|---|
| Signature | `adminAdjustXP(studentId: string, adminId: string, amount: number, note: string): Promise<XPLedgerEntryDTO>` |
| Purpose | The Admin-facing entry point for `XPReason.OTHER` (Doc 02 §10 v3.2 XP Point Values callout). Closes the gap where `OTHER` was defined as "the only reason where `amount` is caller-supplied" but no function actually accepted a caller-supplied amount from an Admin. Also the sole mechanism behind UC-88's leaderboard correction — the leaderboard has no separate stored ranking to edit (Doc 04 §4.0), so correcting it means writing a signed `XPLedgerEntry`. |
| Throws | `ApiError(404, "Student not found")` — no `StudentProfile` matches `studentId`. `ApiError(400, "amount must be a non-zero integer")`. `ApiError(400, "A note is required for a manual XP adjustment")`. |
| Side effects | Calls the same underlying insert path as `awardXP` — an append-only `XPLedgerEntry` row with `reason: OTHER`, `amount` exactly as supplied (positive or negative), `note` always populated (unlike other reasons, where `note` is optional). Does **not** call `streak.service.ts → updateStreakOnActivity` — a manual XP correction is not itself a day of platform activity, so it must never fabricate or extend a streak. |

Test file: `tests/services/xp.service.test.ts`

### src/controllers/xp.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getMyProgress | `xp.service` read: sum of `XPLedgerEntry` for caller-resolved `studentId` (see edge case) + `Streak` row + recent entries | 200 |
| getLeaderboard | `xpService.getLeaderboard(req.user.id, req.user.role, req.query.studentId, req.query.period)` | 200 |
| adminAdjust | `xpService.adminAdjustXP(req.params.studentId, req.user.id, req.body.amount, req.body.note)` — **I2 fix** | 201 |

| Field | Detail |
|---|---|
| getMyProgress edge case | A streak broken by an inactive period resets `currentStreakDays` without deleting previously earned badges/XP — `totalXP` is never reduced by a streak reset (UC-63 alternate flow). |
| getMyProgress auth | **H3 fix:** now `Student\|Parent`, matching the API spec and `getLeaderboard`'s existing pattern below — for `PARENT`, resolves `req.query.studentId` through an `ACTIVE` `ParentStudentRelationship` (same check named in `00-api-conventions.md` §0.2 and used by `profile.service.ts → getProfile`) before reading that student's ledger/streak; for `STUDENT`, `studentId` is always the caller's own, `req.query.studentId` is ignored if present. |

### src/routes/xp.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /gamification/xp/me | `authMiddleware` | getMyProgress |
| GET | /gamification/leaderboard | `authMiddleware` | getLeaderboard |
| POST | /admin/students/:studentId/xp-adjustments | `authMiddleware, requireRole('ADMIN'), validate(adjustXPSchema)` | adminAdjust — **I2 fix** |

Neither of the first two routes restricts by `requireRole` at the middleware layer — both are `Student\|Parent`, with the caller-vs-target-student resolution (and the `ACTIVE ParentStudentRelationship` check for a Parent caller) done inside the service function, identical in shape to `getLeaderboard`'s existing pattern. The Admin route is the one exception, restricted by `requireRole('ADMIN')` at the middleware layer since it has no caller-vs-target ambiguity to resolve.

---

### src/services/badge.service.ts (new)

#### awardStudentBadge / awardTutorBadge

| Field | Detail |
|---|---|
| Signature | `awardStudentBadge(studentId: string, badgeId: string): Promise<StudentBadgeDTO>` · `awardTutorBadge(tutorId: string, badgeId: string): Promise<TutorBadgeDTO>` |
| Purpose | Event-triggered from wherever a badge's criteria is met (e.g. a 5-session streak) — not directly client-facing. |
| Edge cases | `criteriaDescription` on every `Badge` is always experience/performance/achievement-based — there is no `rating`-derived field anywhere in this model (Doc 04 Badge notes, FC-01); this constraint is structural (no such column exists), not something this function needs to additionally validate. |

Test file: `tests/services/badge.service.test.ts` — includes the no-rating-field-exists case (a schema/shape assertion rather than a runtime branch).

#### createBadge — **I2 fix**

| Field | Detail |
|---|---|
| Signature | `createBadge(input: { name, description, category, criteriaDescription, isActive? }): Promise<BadgeDTO>` |
| Purpose | Create a new badge definition (`POST /admin/badges`). Closes the gap where Doc 02 §10's v3.2 V1 Badge List callout promised "Admin can add more later" but only an *edit* endpoint (`PATCH /admin/badges/:badgeId`) existed. |
| Edge cases | Same structural constraint as `awardStudentBadge`/`awardTutorBadge`: no `rating`-derived field exists on `Badge` to accidentally populate (Doc 04 Badge notes, FC-01) — `criteriaDescription` is always free-text. |

Test file: `tests/services/badge.service.test.ts`

#### adminManageBadges

| Field | Detail |
|---|---|
| Signature | `adminManageBadges(page?, limit?, category?): Promise<PaginatedBadgeDTO>` — list mode. `adminManageBadges(badgeId: string, input: { criteriaDescription?, isActive? }): Promise<BadgeDTO>` — adjust mode, per Doc 05a's single-function naming covering both the list and adjust endpoints. |
| Purpose | List all badge definitions and award history for review (`GET /admin/badges`), and adjust criteria/active status (`PATCH /admin/badges/:badgeId`). Creation is handled separately by `createBadge` above (**I2 fix**), not folded into this function, since create/list/adjust have distinct enough signatures that a single overloaded function would be harder to type safely. |

Test file: `tests/services/badge.service.test.ts`

### src/controllers/badge.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listMyBadges | direct read of `StudentBadge` joined to `Badge`, scoped to the caller-resolved `studentId` (H3 fix — see below) | 200 |
| adminListAll | `badgeService.adminManageBadges(req.query.category, req.query.page, req.query.limit)` | 200 |
| adminCreate | `badgeService.createBadge(req.body)` — **I2 fix** | 201 |
| adminAdjust | `badgeService.adminManageBadges(req.params.badgeId, req.body)` | 200 |

| Field | Detail |
|---|---|
| listMyBadges auth | **H3 fix:** now `Student\|Parent`. For `PARENT`, resolves `req.query.studentId` through an `ACTIVE` `ParentStudentRelationship` before scoping the `StudentBadge` read; for `STUDENT`, always the caller's own. Same resolution pattern as `xp.controller.ts → getMyProgress` and `getLeaderboard`. |

### src/routes/badge.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /gamification/badges/me | `authMiddleware` | listMyBadges |
| GET | /admin/badges | `authMiddleware, requireRole('ADMIN')` | adminListAll |
| POST | /admin/badges | `authMiddleware, requireRole('ADMIN'), validate(createBadgeSchema)` | adminCreate — **I2 fix** |
| PATCH | /admin/badges/:badgeId | `authMiddleware, requireRole('ADMIN')` | adminAdjust |

---

### src/services/streak.service.ts (new)

#### updateStreakOnActivity / resetStreakOnGap

| Field | Detail |
|---|---|
| Signature | `updateStreakOnActivity(studentId: string, activityDate: string): Promise<StreakDTO>` · `resetStreakOnGap(studentId: string): Promise<StreakDTO>` |
| Purpose | Internal only — triggered by activity events (class attendance, assessment completion), never exposed as its own endpoint. |
| Side effects | `updateStreakOnActivity` increments `currentStreakDays` if `activityDate` is consecutive with `lastActivityDate`, else resets to `1`; updates `longestStreakDays` if the new current exceeds it. `resetStreakOnGap` is called by a scheduled check (or lazily on next read) for a student who had no qualifying activity for a full day, per the streak-break rule. |

Test file: `tests/services/streak.service.test.ts`

---

### src/schemas/challenge.schema.ts (new)

| Schema | Shape |
|---|---|
| createChallengeSchema | `z.object({ body: z.object({ title: z.string().min(1), description: z.string().min(1), period: z.enum(['WEEKLY','MONTHLY']), startsAt: z.string().datetime(), endsAt: z.string().datetime(), targetValue: z.number().int().positive() }).refine(b => new Date(b.endsAt) > new Date(b.startsAt), "End time must be after start time") })` |

### src/services/challenge.service.ts (new)

#### createChallenge / listActiveChallenges / trackProgress

| Field | Detail |
|---|---|
| Signature | `createChallenge(input, adminId: string): Promise<ChallengeDTO>` · `listActiveChallenges(): Promise<ChallengeDTO[]>` · `trackProgress(studentId: string, challengeId: string, increment: number): Promise<ChallengeProgressDTO>` |
| Throws | (create) `ApiError(400, "End time must be after start time")` — redundant with the schema, retained as the authoritative rule. |
| Side effects | `trackProgress` is event-triggered (e.g. an assessment completion increments progress toward a "complete 3 assessments" challenge) rather than client-callable directly; it upserts `ChallengeProgress`, setting `completedAt` once `progressValue >= targetValue`. |

Test file: `tests/services/challenge.service.test.ts`

### src/controllers/challenge.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listActive | `challengeService.listActiveChallenges()` | 200 |
| adminCreate | `challengeService.createChallenge(req.body, req.user.id)` | 201 |
| getMyProgress | direct read of `ChallengeProgress` for the caller-resolved `studentId` across active challenges (H3 fix — see below) | 200 |

| Field | Detail |
|---|---|
| getMyProgress auth | **H3 fix:** now `Student\|Parent`. Same `ACTIVE ParentStudentRelationship` resolution pattern as `xp.controller.ts → getMyProgress`. |

### src/routes/challenge.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /gamification/challenges | `authMiddleware` | listActive |
| GET | /gamification/challenges/me | `authMiddleware` | getMyProgress |
| POST | /admin/challenges | `authMiddleware, requireRole('ADMIN'), validate(createChallengeSchema)` | adminCreate |

---

**Next:** proceed to → [8-7. Backend: Payments & Earnings]
