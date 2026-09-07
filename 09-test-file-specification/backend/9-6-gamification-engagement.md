## Project: AKEWTutor — Backend Test Documentation: Gamification & Engagement
**Links back to:** [05a. Backend Folder & File Structure §6], [8-6. Function-Level Spec: Gamification & Engagement]
**Conventions:** see `00-api-conventions.md` §0.1–0.7. See also Doc 02 Section 10's v3.2 XP point value table, streak milestone definitions, and V1 badge list.

Per the standing rule: test file mirrors `src/` exactly under `tests/`. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

**Owns:** XPLedgerEntry, Badge, StudentBadge, TutorBadge, Streak, Challenge, ChallengeProgress. **Depends on:** `accounts-guardianship` (hard); soft-integrates with `class-delivery-library` (XP-award trigger on class-attended — no FK, event-based only).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| xp.service.ts | FR-SP-039, FR-GA-002, Section 10 v3.2 XP point values | — |
| badge.service.ts | FR-GA-003, FR-GA-005, FR-AD-004, FR-AD-018 (admin badge/achievement management) | — |
| streak.service.ts | FR-GA-003, FR-GA-006, Section 10 v3.2 streak milestones | — |
| challenge.schema.ts | FR-SP-040, FR-GA-004 | — |
| challenge.service.ts | FR-SP-040, FR-GA-004, FR-AD-018 (admin leaderboard/challenge management) | — |

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/services/xp.service.ts | tests/services/xp.service.test.ts | Unit (mocked Prisma, mocked `streak.service.updateStreakOnActivity`) | ☐ |
| src/schemas/xp.schema.ts | tests/schemas/xp.schema.test.ts | Unit — **I2 fix** | ☐ |
| src/controllers/xp.controller.ts | tests/controllers/xp.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/xp.routes.ts | tests/routes/xp.routes.test.ts | Integration (supertest) | ☐ |
| src/services/badge.service.ts | tests/services/badge.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/schemas/badge.schema.ts | tests/schemas/badge.schema.test.ts | Unit — **I2 fix** | ☐ |
| src/controllers/badge.controller.ts | tests/controllers/badge.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/badge.routes.ts | tests/routes/badge.routes.test.ts | Integration (supertest) | ☐ |
| src/services/streak.service.ts | tests/services/streak.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/schemas/challenge.schema.ts | tests/schemas/challenge.schema.test.ts | Unit | ☐ |
| src/services/challenge.service.ts | tests/services/challenge.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/challenge.controller.ts | tests/controllers/challenge.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/challenge.routes.ts | tests/routes/challenge.routes.test.ts | Integration (supertest) | ☐ |

---

### 9.2 Test Case Detail — xp.service.test.ts

FRs: FR-SP-039, FR-GA-002, Section 10 v3.2 XP point values. **OWASP: A01:2021 – Broken Access Control (grade-scoping and Parent→child resolution), A04:2021 – Insecure Design (XP values are business-critical constants; a wrong value is a monetization/fairness-adjacent bug, not just cosmetic).**

#### awardXP

| Case | Setup | Action | Expected result |
|---|---|---|---|
| CLASS_ATTENDED awards exactly 20 | — | call `awardXP(studentId, undefined, 'CLASS_ATTENDED')` — amount derived internally, not caller-supplied for this reason | inserted `XPLedgerEntry.amount === 20` |
| ASSESSMENT_COMPLETED awards exactly 15 | — | call `awardXP(studentId, undefined, 'ASSESSMENT_COMPLETED')` | `amount === 15` |
| STREAK_MILESTONE awards exactly 50 | — | call `awardXP(studentId, undefined, 'STREAK_MILESTONE')` | `amount === 50` |
| CHALLENGE_COMPLETED awards 30 for a weekly challenge | mock the completed `Challenge.period === 'WEEKLY'` | call `awardXP(studentId, undefined, 'CHALLENGE_COMPLETED')` | `amount === 30` |
| CHALLENGE_COMPLETED awards 100 for a monthly challenge | mock `Challenge.period === 'MONTHLY'` | call `awardXP(studentId, undefined, 'CHALLENGE_COMPLETED')` | `amount === 100` — tested as a genuinely different branch from the weekly case, not assumed from one |
| BADGE_AWARDED awards exactly 25 | — | call `awardXP(studentId, undefined, 'BADGE_AWARDED')` | `amount === 25` |
| OTHER accepts a caller-supplied amount | — | call `awardXP(studentId, 12, 'OTHER')` | `amount === 12` — the sole reason where `amount` is caller-supplied rather than a fixed constant per the v3.2 table |
| Never mutates a running total column | spy on all Prisma calls | call `awardXP(studentId, ..., 'CLASS_ATTENDED')` | assert only an `xpLedgerEntry.create` call was made — no `update` targeting any `totalXP`/running-sum field on `StudentProfile` or elsewhere; `totalXP` is always a live sum over the ledger (Doc 04 §4.0) |
| Also updates the streak for the same event | spy on `streak.service.updateStreakOnActivity` | call `awardXP(studentId, ..., 'CLASS_ATTENDED')` | assert it was called once with `(studentId, activityDate)` |

#### getLeaderboard

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Grade-scoping is enforced in the query itself | mock a mixed dataset containing both Grade 8 and Grade 10 students' `XPLedgerEntry` rows | call `getLeaderboard(callerId, 'STUDENT', undefined, 'WEEKLY')` for a Grade 8 caller | assert the underlying query filter includes `grade: 8`, and the resolved `rankings` contains zero Grade 10 entries — tested against an actual cross-grade student present in the fixture, not merely absent from a same-grade-only dataset |
| displayName is always first name + last-initial | mock a student named `"Bethel Molla"` | call `getLeaderboard(...)` | resolves `rankings[].displayName === "Bethel M."` — asserted as a shape check that no `lastName`-bearing field is reachable anywhere in the DTO, not just that one example string happens to look truncated |
| WEEKLY vs MONTHLY use different aggregation windows | mock ledger entries spanning both windows | call `getLeaderboard(..., 'WEEKLY')` then `getLeaderboard(..., 'MONTHLY')` | the two calls aggregate over genuinely different date ranges, verified via the query's date-filter arguments, not just a differently-labeled response |
| Parent caller resolves studentId via an ACTIVE relationship | mock caller is a Parent with an `ACTIVE` `ParentStudentRelationship` to `studentId` | call `getLeaderboard(parentId, 'PARENT', studentId, 'WEEKLY')` | resolves the target student's leaderboard context |
| Parent caller with no relationship to the queried student (IDOR) | mock no `ACTIVE` relationship exists between caller and `studentId` | call `getLeaderboard(parentId, 'PARENT', otherStudentId, 'WEEKLY')` | throws `ApiError(403, ...)` |
| Student caller ignores any studentId override | mock caller is a Student | call `getLeaderboard(studentId, 'STUDENT', someOtherStudentId, 'WEEKLY')` | resolves the caller's own leaderboard context — the supplied `someOtherStudentId` has no effect |
| callerRank present even outside the top ranks | mock caller ranked 47th of 50 in-grade students | call `getLeaderboard(...)` | resolves `callerRank: 47` even though `rankings` itself may only list the top N |

#### adminAdjustXP — **I2 fix**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Writes an XPLedgerEntry with reason OTHER and the exact supplied amount | — | call `adminAdjustXP(studentId, adminId, -20, "Reversing an erroneous award")` | inserted `XPLedgerEntry` has `amount: -20`, `reason: 'OTHER'`, `note: "Reversing an erroneous award"` |
| Accepts a negative amount (correction) as well as positive (goodwill) | — | call `adminAdjustXP(studentId, adminId, 10, "Goodwill")`, then `adminAdjustXP(studentId, adminId, -10, "Correction")` | both succeed; the ledger contains both signed entries, and a `SUM(amount)` read nets to zero |
| Rejects a zero amount | — | call `adminAdjustXP(studentId, adminId, 0, "note")` | throws `ApiError(400, "amount must be a non-zero integer")` |
| Rejects a missing/empty note | — | call `adminAdjustXP(studentId, adminId, 10, "")` | throws `ApiError(400, "A note is required for a manual XP adjustment")` |
| Student not found | mock no matching `StudentProfile` | call `adminAdjustXP(studentId, adminId, 10, "note")` | throws `ApiError(404, "Student not found")` |
| Never calls updateStreakOnActivity | spy on `streak.service.updateStreakOnActivity` | call `adminAdjustXP(studentId, adminId, 10, "note")` | assert it was NOT called — a manual correction must never fabricate or extend a streak, unlike `awardXP` |
| Feeds into the leaderboard exactly like any other ledger entry | create an adjustment, then call `getLeaderboard` for the same grade/period | call `getLeaderboard(...)` | the adjustment's `amount` is reflected in the student's aggregated total — confirms UC-88's leaderboard-correction claim is actually true end-to-end, not just that a row was written |

---

### 9.3 Test Case Detail — xp.controller.test.ts / xp.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Both public-facing routes require auth | no Authorization header | request `GET /gamification/xp/me`, `GET /gamification/leaderboard` | both `401` |
| getMyProgress is Student\|Parent (H3 fix) | mock controller layer, Student token vs. Parent token with `?studentId=` | call each | both reach the handler; a Parent without `?studentId=` is rejected — flagged since Doc 8-6 requires it for Parent but Doc 06's query description marks it "required for Parent," so the controller must enforce this itself if the schema doesn't |
| Parent resolution uses the ACTIVE ParentStudentRelationship pattern | mock service | call controller as Parent | `getMyProgress`'s underlying read resolves `req.query.studentId` through the same relationship check as `xp.service.ts → getLeaderboard`, not a raw pass-through |
| A streak reset never reduces totalXP | mock `Streak.currentStreakDays` reset to 1 after a gap, `totalXP` ledger sum unchanged | call `getMyProgress` | resolves the same `totalXP` as before the reset — badges/XP already earned are untouched (UC-63 alternate flow) |
| getLeaderboard requires `period` | mock controller layer, valid token, no `?period=` | request `GET /gamification/leaderboard` | rejected — `period` is a required enum per Doc 06 §6.2 |
| adminAdjust requires Admin — **I2 fix** | no Authorization header, then a Parent token | request `POST /admin/students/:studentId/xp-adjustments` | `401` then `403` |
| adminAdjust passes req.user.id as the adjusting admin — **I2 fix** | mock service; valid Admin token | call controller with `{ amount: 10, note: "..." }` | `adminAdjustXP` called with `req.user.id`, never a client-suppliable admin id |
| adminAdjust validates body — **I2 fix** | valid Admin token | request with `amount: 0` or a missing `note` | rejected by `validate(adjustXPSchema)`, controller never called |

---

### 9.4 Test Case Detail — badge.service.test.ts

FRs: FR-GA-003, FR-GA-005, FR-AD-004, FR-AD-018 (`adminManageBadges` is the Admin-facing "manage achievements" half of FR-AD-018). **OWASP: A01:2021 – Broken Access Control, A04:2021 – Insecure Design (no-rating-derived field is a structural design safety control per FC-01, not an incidental omission).**

#### awardStudentBadge / awardTutorBadge

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Awards a student badge | mock a qualifying event (e.g. streak milestone reached) | call `awardStudentBadge(studentId, badgeId)` | resolves `StudentBadgeDTO`; a `StudentBadge` row created |
| Awards a tutor badge | — | call `awardTutorBadge(tutorId, badgeId)` | resolves `TutorBadgeDTO`; a `TutorBadge` row created |
| No `rating`-derived field exists anywhere on the returned DTO | inspect the resolved `StudentBadgeDTO`/`TutorBadgeDTO` and the joined `Badge` row | call either award function | assert neither object contains a `rating`/`score`-style key — this is a schema/shape assertion (FC-01's structural constraint), not a runtime branch, since no such column exists to begin with |
| Re-awarding a badge the student already has | mock a `StudentBadge` row already exists for `(studentId, badgeId)` | call `awardStudentBadge(studentId, badgeId)` again | **flagged, not hard-asserted:** Doc 8-6 doesn't specify whether this is a silent no-op, an upsert, or a unique-constraint error; this doc requires the implementer's chosen behavior be documented and applied consistently rather than guessing a specific outcome here |

#### createBadge — **I2 fix**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates a new badge definition | — | call `createBadge({ name: "Quarter Champion", description: "...", category: 'STUDENT', criteriaDescription: "Reach a 90-day streak" })` | resolves `BadgeDTO`; a `Badge` row created with `isActive: true` (default) |
| isActive defaults to true when omitted | — | call `createBadge({ ...input, isActive: undefined })` | resolves `isActive: true` |
| No `rating`-derived field exists on the created row | inspect the resolved `BadgeDTO` | call `createBadge(...)` | assert no `rating`/`score`-style key is present — same structural guarantee as `awardStudentBadge`/`awardTutorBadge` above |

#### adminManageBadges

| Case | Setup | Action | Expected result |
|---|---|---|---|
| List mode returns paginated badges filtered by category | mock a mix of `STUDENT` and `TUTOR` badges | call `adminManageBadges(1, 20, 'STUDENT')` | resolves `PaginatedBadgeDTO` containing only `category: STUDENT` rows |
| Adjust mode updates criteria/active status | mock an existing badge | call `adminManageBadges(badgeId, { isActive: false })` | resolves `BadgeDTO` with `isActive: false`; `criteriaDescription` unchanged since it wasn't supplied |
| The two call shapes are dispatched correctly | spy on the underlying Prisma calls | call once with `(page, limit, category)` args and once with `(badgeId, input)` args | the list-mode call never triggers an `update`, and the adjust-mode call never triggers a paginated `findMany` — confirms the dual-purpose function doesn't cross-wire its two modes |

---

### 9.5 Test Case Detail — badge.controller.test.ts / badge.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listMyBadges is Student\|Parent (H3 fix) | mock service | call controller as Student, then as Parent with `?studentId=` | both resolve; Parent resolution follows the same `ACTIVE ParentStudentRelationship` pattern as `xp.controller.ts` |
| adminListAll / adminCreate / adminAdjust require Admin — **I2 fix** | no Authorization header, then a Student token | request `GET /admin/badges`, `POST /admin/badges`, and `PATCH /admin/badges/:id` with (a) no token, (b) a Student token | (a) `401` for each; (b) `403` for each |
| adminCreate validates body — **I2 fix** | valid Admin token | request `POST /admin/badges` with a missing `name` or an invalid `category` | rejected by `validate(createBadgeSchema)`, controller never called |
| adminAdjust forwards only the documented fields | mock service | call controller with `{ criteriaDescription: "...", isActive: true, rating: 5 }` | the underlying service call receives only `criteriaDescription`/`isActive` — no `rating`-style field is ever forwarded, consistent with 9.4's structural guarantee |

---

### 9.6 Test Case Detail — streak.service.test.ts

FRs: FR-GA-003, FR-GA-006, Section 10 v3.2 streak milestones. **OWASP: none specific (internal-only, no client-facing input) — however, milestone/date-math correctness is treated with the same rigor as the reschedule/escalation window tests in 9-4, since a boundary bug here silently breaks badge/XP awarding.**

#### updateStreakOnActivity

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Consecutive day increments the streak | mock `lastActivityDate` = yesterday, `currentStreakDays: 4` | call `updateStreakOnActivity(studentId, today)` | resolves `currentStreakDays: 5` |
| Non-consecutive day resets to 1 | mock `lastActivityDate` = 3 days ago | call `updateStreakOnActivity(studentId, today)` | resolves `currentStreakDays: 1` — a gap breaks the streak rather than partially decrementing it |
| longestStreakDays updates only when exceeded | mock `currentStreakDays` about to become `5`, `longestStreakDays: 12` | call `updateStreakOnActivity(...)` | `longestStreakDays` remains `12`, unaffected |
| longestStreakDays updates when the new current exceeds it | mock `currentStreakDays` about to become `13`, `longestStreakDays: 12` | call `updateStreakOnActivity(...)` | `longestStreakDays` becomes `13` |
| longestStreakDays never decreases on a later reset | mock a prior `longestStreakDays: 30` from an earlier streak, then a gap resets `currentStreakDays` to `1`, then rebuild to `10` | call `updateStreakOnActivity` across this sequence | `longestStreakDays` remains `30` throughout — tested as an explicit reset-then-partial-rebuild sequence, not inferred from a single monotonic-increase case |
| Milestone at exactly 7/30/90 is the trigger point | mock the streak about to reach each threshold in turn | call `updateStreakOnActivity` at 7, then 30, then 90 | each crossing is flagged as a milestone-reached event exactly once; whichever caller wires this into `xp.service.awardXP('STREAK_MILESTONE')`/`badge.service.awardStudentBadge` is **flagged for the implementer to confirm** — Doc 8-6 doesn't specify that `streak.service.ts` itself calls those functions directly, only that the milestone is "the trigger condition" |
| A milestone doesn't re-fire mid-streak | mock `currentStreakDays` already at `10` (past the 7-day milestone) | call `updateStreakOnActivity` incrementing to `11` | no milestone event fires again until the next threshold (30) — milestones fire once per threshold crossing, not on every day past it |

#### resetStreakOnGap

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Resets currentStreakDays without touching badges/XP | mock a student who had no qualifying activity for a full day | call `resetStreakOnGap(studentId)` | `currentStreakDays` reset; spy on `badge.service`/`xp.service` calls asserts neither is invoked — previously earned badges/XP are untouched (UC-63 alternate flow) |
| longestStreakDays is preserved across the reset | mock `longestStreakDays: 30` | call `resetStreakOnGap(studentId)` | `longestStreakDays` unchanged at `30` |

---

### 9.7 Test Case Detail — challenge.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requires endsAt after startsAt | — | parse `{ body: { ..., startsAt: "2026-09-10T00:00:00Z", endsAt: "2026-09-01T00:00:00Z" } }` | fails the `.refine` — "End time must be after start time" |
| Accepts a valid ascending range | — | parse with `endsAt` after `startsAt` | passes |
| targetValue must be a positive integer | — | parse with `targetValue: 0`, then `targetValue: -3` | both fail |
| period restricted to WEEKLY\|MONTHLY | — | parse with `period: "DAILY"` | fails |

---

### 9.8 Test Case Detail — challenge.service.test.ts

FRs: FR-SP-040, FR-GA-004, FR-AD-018 (`createChallenge` is the Admin-facing "manage leaderboards" half of FR-AD-018). **OWASP: A01:2021 – Broken Access Control (Admin-only create), A04:2021 – Insecure Design (date-range business rule enforced redundantly at the service layer).**

#### createChallenge

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Creates a valid challenge | — | call `createChallenge(validInput, adminId)` | resolves `ChallengeDTO` |
| Redundant service-layer date check | bypass the schema (simulate a raw internal call) with `endsAt` before `startsAt` | call `createChallenge(...)` | throws `ApiError(400, "End time must be after start time")` — the authoritative rule, not solely relying on the schema `.refine` |

#### listActiveChallenges

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns only currently-active challenges | mock one challenge not yet started, one currently active, one already ended | call `listActiveChallenges()` | resolves only the currently-active one |
| Empty when none are active | mock all challenges outside their window | call `listActiveChallenges()` | resolves `[]`, not an error |

#### trackProgress

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Upserts progress and sets completedAt once target is reached | mock `targetValue: 3`, no existing `ChallengeProgress` | call `trackProgress(studentId, challengeId, 1)` twice more (total 3) | on the third call, `completedAt` is set for the first time |
| completedAt doesn't change once already set | mock `ChallengeProgress` already `completedAt`-set at `progressValue: 3` | call `trackProgress(studentId, challengeId, 1)` again (`progressValue` now 4) | `completedAt` remains the original timestamp — tested as an explicit past-target-twice sequence, not assumed from the completion case alone |
| Event-triggered, not directly client-callable | — | (documentation-level check) confirm no route in `challenge.routes.ts` maps directly to `trackProgress` | no route exists for it — matching the "internal only" note in Doc 8-6 |

---

### 9.9 Test Case Detail — challenge.controller.test.ts / challenge.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listActive is Student\|Parent (H3-consistency fix) | mock controller layer | call as Student, then Parent | both reach the handler — no `studentId` scoping needed since this is a public-within-auth list of currently-running challenges, not student-specific data |
| getMyProgress is Student\|Parent | mock service | call as Parent with `?studentId=` | resolves the target student's progress via the same `ACTIVE ParentStudentRelationship` resolution pattern as `xp.controller.ts` |
| adminCreate requires Admin | no Authorization header, then a Tutor token | request `POST /admin/challenges` | `401` then `403` |
| adminCreate validates body | valid Admin token | request with `endsAt` before `startsAt` | rejected by `validate(createChallengeSchema)`, controller never called |

---

### 9.10 Coverage Honesty Check (per PR Steward, at review time)

- [ ] All six `XPReason` values (`CLASS_ATTENDED`, `ASSESSMENT_COMPLETED`, `STREAK_MILESTONE`, `CHALLENGE_COMPLETED` weekly/monthly, `BADGE_AWARDED`, `OTHER`) are each tested against the exact v3.2 point-value table — not just a sampled subset, since a wrong constant for even one reason silently under/over-pays XP platform-wide.
- [ ] `getLeaderboard`'s per-grade scoping is tested with an actual cross-grade student present in the mocked dataset who must be excluded — not merely absent from a same-grade-only fixture, which would pass even if the grade filter were silently missing from the query.
- [ ] `displayName`'s never-a-full-last-name guarantee is tested as a shape assertion (no reachable `lastName` field), not only a string-format check on one example name.
- [ ] `longestStreakDays`-never-decreases is tested with an explicit reset-then-partial-rebuild sequence, not only a monotonic-increase case, since the bug this guards against is specifically a reset accidentally lowering the recorded longest streak.
- [ ] `trackProgress`'s `completedAt`-set-once behavior is tested by incrementing progress past the target twice and asserting the timestamp is unchanged on the second call — not just that it becomes non-null once.
- [ ] The no-rating-field-exists check for badges is asserted on both `StudentBadgeDTO` and `TutorBadgeDTO` independently, not only one of the two.

---

### 9.11 Out of Scope for Automated Testing (and why)

- **Admin-editable XP point values** — the v3.2 table is a shared constants file in V1, not an Admin-editable database table (Doc 02 §10 explicit note); there is no endpoint to test for changing these values.
- **Exact wiring between `class-delivery-library`'s session-completion event and `xp.service.awardXP`** — this is a soft, event-based integration with no FK (Feature Decomposition §1.1); which file calls `awardXP('CLASS_ATTENDED')` and when is confirmed at build time, not hard-asserted here beyond `awardXP` itself behaving correctly once called.
- **Real-time leaderboard push/websocket updates** — not part of Docs 02/06/08; the backend contract here is a computed-on-request `GET`, per Doc 04 §4.0's explicit no-stored-table decision.
- **awardXP's own error-swallowing responsibility** — Doc 8-6 notes an XP-award failure must never block the triggering action (e.g. marking a class complete), but this is a responsibility of the *calling* code in another feature (e.g. `session.service.ts`), not something `xp.service.ts` itself needs to catch internally; that caller-side behavior is out of scope for this file and belongs with whichever feature's test doc owns the caller.

---

**Next:** proceed to → [9-7. Backend Test Documentation: Payments & Earnings]
