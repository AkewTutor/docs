# AKEWTutor — Backend Folder & File Structure

**Project:** AKEWTutor — Online Tutoring Platform

**Links back to:** [04. Database & Data Model], [Feature Decomposition]
**Links forward to:** [06. API Specification], [08. Function-Level Specification]

**Base template:** Node/Express (same convention as the reference project's `template-node-express`) — `src/{schemas,services,controllers,routes,middlewares,utils,jobs}`, Prisma ORM, Zod validation, Vitest for tests.

**Structuring principle:** Files are grouped by the 8 features defined in `feature-decomposition.md`, not by technical layer alone — every feature gets its own slice through `schemas/`, `services/`, `controllers/`, `routes/`, and `jobs/`, mirroring the parallelizable build order so a team can work feature-by-feature with minimal file collisions. Cross-cutting foundations that no single feature owns are listed separately in Section 0.

**Standing rule (carried over from the reference project):** every service file gets a mirrored test file under `tests/`, same relative path.

---

## 0. Cross-Cutting Foundations

These exist before any feature and are not owned by one feature's team.

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/middlewares/auth.middleware.ts` | Middleware | verifies JWT, attaches `req.user`; role check via a `requireRole(...roles)` wrapper — no separate `adminOnly.middleware.ts` needed since the wrapper covers it | jwt utils |
| `src/middlewares/error.middleware.ts` | Middleware | centralized error → HTTP response mapping | — |
| `src/middlewares/validate.middleware.ts` | Middleware | Zod schema validation wrapper for request body/query/params | zod |
| `src/middlewares/rateLimiter.middleware.ts` | Middleware | In-memory per-endpoint rate limiting (NFR-013) — login, resend-verification, forgot-password, payment initiation, send-message | express-rate-limit |
| `src/config/rateLimits.ts` | Config | named threshold constants consumed by `rateLimiter.middleware.ts` call sites | — |
| `src/utils/jwt.ts` | Util | sign/verify access tokens (30-min TTL, NFR-014) | env config |
| `src/utils/refreshToken.ts` | Util | generate/hash opaque refresh tokens (30-day TTL, rotation + reuse detection, NFR-014/015) | crypto (node built-in) |
| `src/utils/password.ts` | Util | bcrypt hash/compare | — |
| `src/config/db.ts` | Config | shared Prisma client instance, Prisma 7 driver-adapter pattern (pg `Pool` → `@prisma/adapter-pg` → `PrismaClient`), per template §5.2 — moved here from `src/utils/prisma.ts` to match the template's folder contract (`src/config/` = "external connections — DB and validated env vars") | prisma, pg, @prisma/adapter-pg, env config |
| `src/utils/pagination.ts` | Util | shared list/pagination helper used across every controller | — |
| `src/jobs/scheduler.ts` | Util | registers all cron/interval jobs listed under each feature below | node-cron (or equivalent) |
| `prisma/schema.prisma` | Schema | full schema from Doc 04 (40 entities, all enums) | — |
| `prisma/seed.ts` | Seed script | bootstraps the first Admin user, the base `Subject` catalog, and a default `PricingConfig` row per `TutoringFormat` (`ONE_TO_ONE`/`ONE_TO_THREE`/`ONE_TO_FIVE`) — **fix (audit):** without this, `matching-cohorts` has no active price to read on a fresh clone/test DB, since `payments-earnings` (which owns `PricingConfig`) is built and seeded last in build order but is read from as early as the first `Cohort` reaching `PENDING_PAYMENT` | prisma |
| `tests/utils/*.test.ts`, `tests/middlewares/*.test.ts` | Test | mirrors each cross-cutting util/middleware | — |

---

## 1. Feature: `shared-config`

**Owns:** User, Notification, PolicyDocument, RefreshToken. **Depends on:** nothing (foundation feature).

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/schemas/auth.schema.ts` | Zod schema | registerStudentSchema, registerParentSchema, registerTutorSchema, loginSchema, passwordResetRequestSchema, passwordResetSchema, verifyContactSchema | — |
| `src/services/auth.service.ts` | Service | registerUser (role-aware), login, refreshAccessToken, logout, logoutAll, requestPasswordReset, resetPassword, verifyContact, resendVerification | prisma, password.ts, jwt.ts, refreshToken.ts, sms/email clients |
| `src/controllers/auth.controller.ts` | Controller | register, login, refresh, logout, logoutAll, forgotPassword, resetPassword, verify handlers | auth.service.ts |
| `src/routes/auth.routes.ts` | Route | mounts `/auth/*` — public | auth.controller.ts |
| `src/utils/providers/sms.client.ts` | Util | wraps Geez SMS (verification codes, SMS notification channel) | env config |
| `src/utils/providers/email.client.ts` | Util | wraps Brevo (verification emails, email notification channel) | env config |
| `src/schemas/notification.schema.ts` | Zod schema | listNotificationsQuerySchema | — |
| `src/services/notification.service.ts` | Service | dispatchNotification (routes to sms/email/push client by `preferredNotificationChannel`), listForUser, markRead, retryFailed | prisma, sms.client.ts, email.client.ts |
| `src/controllers/notification.controller.ts` | Controller | listMyNotifications, markAsRead handlers | notification.service.ts |
| `src/routes/notification.routes.ts` | Route | mounts `/notifications/*`; `authMiddleware` on all | notification.controller.ts |
| `src/services/adminAnnouncement.service.ts` | Service | composePlatformAnnouncement, adjustNotificationRules (FR-AD-019) | notification.service.ts |
| `src/controllers/adminAnnouncement.controller.ts` | Controller | createAnnouncement, listAnnouncements handlers | adminAnnouncement.service.ts |
| `src/routes/adminAnnouncement.routes.ts` | Route | mounts `/admin/announcements/*`; `authMiddleware` + `requireRole(ADMIN)` | adminAnnouncement.controller.ts |
| `src/schemas/policy.schema.ts` | Zod schema | publishPolicySchema | — |
| `src/services/policy.service.ts` | Service | getCurrentPolicy (by type, latest version), publishNewVersion | prisma |
| `src/controllers/policy.controller.ts` | Controller | getPolicy (public), publishPolicy (admin) handlers | policy.service.ts |
| `src/routes/policy.routes.ts` | Route | mounts `/policies/*` (public GET) and `/admin/policies/*` (admin POST) | policy.controller.ts |
| `src/jobs/notificationRetry.job.ts` | Job | retries `Notification.status = FAILED` rows on an interval | notification.service.ts |
| `tests/services/auth.service.test.ts` | Test | mirrors auth.service.ts — includes generic-error-on-invalid-credentials case | — |
| `tests/services/notification.service.test.ts` | Test | mirrors notification.service.ts | — |
| `tests/services/policy.service.test.ts` | Test | mirrors policy.service.ts | — |

---

## 2. Feature: `accounts-guardianship`

**Owns:** StudentProfile, ParentProfile, TutorProfile, ParentStudentRelationship, Subject, TutorSubjectRanking, AvailabilitySlot. **Depends on:** `shared-config`.

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/schemas/studentProfile.schema.ts` | Zod schema | updateAcademicProfileSchema (grade, school, subjects, goals, language, schedule, teachingStyle, budget, format) | — |
| `src/services/studentProfile.service.ts` | Service | getProfile, updateAcademicProfile, updateBasicProfile (photo/name) | prisma |
| `src/controllers/studentProfile.controller.ts` | Controller | getMyProfile, updateAcademicProfile, updateBasicProfile handlers | studentProfile.service.ts |
| `src/routes/studentProfile.routes.ts` | Route | mounts `/students/me/*`; `authMiddleware` | studentProfile.controller.ts |
| `src/schemas/guardianship.schema.ts` | Zod schema | addStudentSchema (grade required), inviteGuardianSchema, revokeRelationshipSchema | — |
| `src/services/guardianship.service.ts` | Service | addStudentAndInvite, resendOrRegenerateInvite, activateInvite, inviteOptionalGuardian, revokeOrModifyRelationship, handleSoleGuardianRemoval (→ `GUARDIAN_REQUIRED_HOLD`) | prisma, notification.service.ts |
| `src/controllers/guardianship.controller.ts` | Controller | addStudent, resendInvite, activateInvite, inviteGuardian, revokeRelationship handlers | guardianship.service.ts |
| `src/routes/guardianship.routes.ts` | Route | mounts `/guardianship/*`; `authMiddleware` | guardianship.controller.ts |
| `src/jobs/inviteReminder.job.ts` | Job | sends the day-7 activation reminder for pending invites | guardianship.service.ts, notification.service.ts |
| `src/schemas/tutorProfile.schema.ts` | Zod schema | updateTutorProfileSchema, rankSubjectsSchema (max 2, enforced) | — |
| `src/services/tutorProfile.service.ts` | Service | getProfile, updateProfile, rankSubjects (hard cap at 2), submitForVerification | prisma |
| `src/controllers/tutorProfile.controller.ts` | Controller | getMyProfile, updateProfile, rankSubjects handlers | tutorProfile.service.ts |
| `src/routes/tutorProfile.routes.ts` | Route | mounts `/tutors/me/*`; `authMiddleware` | tutorProfile.controller.ts |
| `src/schemas/availability.schema.ts` | Zod schema | createSlotSchema, deleteSlotSchema | — |
| `src/services/availability.service.ts` | Service | setSlots, listSlots, removeSlot (blocked if a confirmed session depends on it) | prisma |
| `src/controllers/availability.controller.ts` | Controller | listMyAvailability, addSlot, removeSlot handlers | availability.service.ts |
| `src/routes/availability.routes.ts` | Route | mounts `/tutors/me/availability/*`; `authMiddleware` | availability.controller.ts |
| `src/services/subject.service.ts` | Service | listSubjects (public), createSubject, deactivateSubject (admin) | prisma |
| `src/controllers/subject.controller.ts` | Controller | listSubjects, createSubject, deactivateSubject handlers | subject.service.ts |
| `src/routes/subject.routes.ts` | Route | mounts `/subjects/*` (public GET) and `/admin/subjects/*` (admin mutate) | subject.controller.ts |
| `src/services/adminTutorVerification.service.ts` | Service | listPendingTutors, approveTutor, rejectTutor | prisma, notification.service.ts |
| `src/controllers/adminTutorVerification.controller.ts` | Controller | listPending, approve, reject handlers | adminTutorVerification.service.ts |
| `src/routes/adminTutorVerification.routes.ts` | Route | mounts `/admin/tutors/*`; `authMiddleware` + `requireRole(ADMIN)` | adminTutorVerification.controller.ts |
| `src/services/adminPeople.service.ts` | Service | listUsers (student/parent/tutor), suspendAccount, restrictAccount, manageRelationshipRecords | prisma |
| `src/controllers/adminPeople.controller.ts` | Controller | listUsers, suspendAccount, editRelationship handlers | adminPeople.service.ts |
| `src/routes/adminPeople.routes.ts` | Route | mounts `/admin/people/*`; `authMiddleware` + `requireRole(ADMIN)` | adminPeople.controller.ts |
| `tests/services/studentProfile.service.test.ts` | Test | mirrors studentProfile.service.ts | — |
| `tests/services/guardianship.service.test.ts` | Test | mirrors guardianship.service.ts — includes 14-day expiry/reset and sole-guardian-hold cases | — |
| `tests/services/tutorProfile.service.test.ts` | Test | mirrors tutorProfile.service.ts — includes third-subject-rejected case | — |
| `tests/services/availability.service.test.ts` | Test | mirrors availability.service.ts — includes slot-in-use-cannot-delete case | — |
| `tests/services/adminTutorVerification.service.test.ts` | Test | mirrors adminTutorVerification.service.ts | — |
| `tests/services/adminPeople.service.test.ts` | Test | mirrors adminPeople.service.ts | — |

---

## 3. Feature: `matching-cohorts`

**Owns:** MatchRequest, TutorExclusion, Cohort, CohortMembership, FormatSwitchRequest. **Depends on:** `accounts-guardianship`.

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/schemas/matching.schema.ts` | Zod schema | searchTutorsQuerySchema (1-to-1 filters), selectTutorSchema, noExactMatchSchema | — |
| `src/services/matching.service.ts` | Service | searchOneToOneTutors, recommendTutorsWithMatchPercent, selectTutor (Path A), triggerNoExactMatch (Path B), requestGroupFormat (Path C entry) | prisma, cohort.service.ts |
| `src/controllers/matching.controller.ts` | Controller | searchTutors, getRecommendations, selectTutor, noExactMatch, requestGroupFormat handlers | matching.service.ts |
| `src/routes/matching.routes.ts` | Route | mounts `/matching/*`; `authMiddleware` | matching.controller.ts |
| `src/services/cohort.service.ts` | Service | formOrJoinCohort (Path C grouping), approveCohort, rejectCohort (re-queue logic per path), tutorExitContinuity (drop-out/suspension), endCohort | prisma, tutorExclusion logic |
| `src/controllers/cohort.controller.ts` | Controller | getMyCohort, getCohortMembers (name+photo only for group formats) handlers | cohort.service.ts |
| `src/routes/cohort.routes.ts` | Route | mounts `/cohorts/*`; `authMiddleware` | cohort.controller.ts |
| `src/services/adminMatching.service.ts` | Service | listPendingApprovals (overdue-flagged), approveBooking, rejectBooking, manuallyAssignTutor (Path B), manuallyAssembleGroup (Path C double-fail) | prisma, cohort.service.ts, notification.service.ts |
| `src/controllers/adminMatching.controller.ts` | Controller | listQueue, approve, reject, manualAssign handlers | adminMatching.service.ts |
| `src/routes/adminMatching.routes.ts` | Route | mounts `/admin/matching/*`; `authMiddleware` + `requireRole(ADMIN)` | adminMatching.controller.ts |
| `src/schemas/formatSwitch.schema.ts` | Zod schema | requestFormatSwitchSchema | — |
| `src/services/formatSwitch.service.ts` | Service | requestSwitch (cancel current membership, spawn new MatchRequest, trigger refund) | cohort.service.ts, matching.service.ts, refund.service.ts (payments-earnings) |
| `src/controllers/formatSwitch.controller.ts` | Controller | requestSwitch handler | formatSwitch.service.ts |
| `src/routes/formatSwitch.routes.ts` | Route | mounts `/format-switch/*`; `authMiddleware` | formatSwitch.controller.ts |
| `src/jobs/groupFormationWindow.job.ts` | Job | closes a `Cohort.groupFormationWindowExpiresAt`, forms class at reached size (Section 7 partial-formation rule) | cohort.service.ts |
| `src/jobs/zeroMatchEscalation.job.ts` | Job | detects 48h continuous zero-match `MatchRequest` rows, auto-escalates to Path B | matching.service.ts |
| `src/jobs/staleApproval.job.ts` | Job | flags 48h-overdue approvals, escalates + notifies student at 5 days | adminMatching.service.ts, notification.service.ts |
| `tests/services/matching.service.test.ts` | Test | mirrors matching.service.ts — includes budget/language hard-filter cases | — |
| `tests/services/cohort.service.test.ts` | Test | mirrors cohort.service.ts — includes partial-group-fixed-price and tutor-exit-group-continuity cases | — |
| `tests/services/adminMatching.service.test.ts` | Test | mirrors adminMatching.service.ts — includes rejection re-routing (Path A exclusion vs. Path C re-queue) cases | — |
| `tests/services/formatSwitch.service.test.ts` | Test | mirrors formatSwitch.service.ts — includes cohort-mates-unaffected case | — |

---

## 4. Feature: `class-delivery-library`

**Owns:** ScheduledSession, RescheduleRequest, SessionMiss, RecordingConsent, Recording, LibraryMaterial, WeeklyAssessment. **Depends on:** `matching-cohorts`.

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/schemas/session.schema.ts` | Zod schema | provideJitsiLinkSchema | — |
| `src/services/session.service.ts` | Service | generateSessionsForCohort, provideJitsiLink (≥30min-before check), markCompleted | prisma |
| `src/controllers/session.controller.ts` | Controller | listMySessions, provideLink, getSession handlers | session.service.ts |
| `src/routes/session.routes.ts` | Route | mounts `/sessions/*`; `authMiddleware` | session.controller.ts |
| `src/schemas/recordingConsent.schema.ts` | Zod schema | acknowledgeConsentSchema | — |
| `src/services/recordingConsent.service.ts` | Service | getConsentStatus, acknowledgeAsStudentOrParent, acknowledgeAsTutor (blocks first recorded session until both set) | prisma |
| `src/controllers/recordingConsent.controller.ts` | Controller | getStatus, acknowledge handlers | recordingConsent.service.ts |
| `src/routes/recordingConsent.routes.ts` | Route | mounts `/recording-consent/*`; `authMiddleware` | recordingConsent.controller.ts |
| `src/utils/providers/storage.client.ts` | Util | wraps Cloudflare R2 (upload, signed URL generation) | env config |
| `src/services/recording.service.ts` | Service | uploadRecording (720p, sets 90-day `expiresAt`), getSignedUrl, keepPermanently, flagMissing, escalateMissing | prisma, storage.client.ts |
| `src/controllers/recording.controller.ts` | Controller | upload, getMyRecordings, getSignedUrl, keepPermanently handlers | recording.service.ts |
| `src/routes/recording.routes.ts` | Route | mounts `/recordings/*`; `authMiddleware` (access scoped to own cohort membership) | recording.controller.ts |
| `src/schemas/library.schema.ts` | Zod schema | uploadMaterialSchema | — |
| `src/services/library.service.ts` | Service | uploadMaterial, listCohortMaterials, adminManageLibrary (retention/access overrides) | prisma, storage.client.ts |
| `src/controllers/library.controller.ts` | Controller | upload, listForCohort, adminOverride handlers | library.service.ts |
| `src/routes/library.routes.ts` | Route | mounts `/library/*` and `/admin/library/*`; `authMiddleware` | library.controller.ts |
| `src/schemas/reschedule.schema.ts` | Zod schema | requestRescheduleSchema | — |
| `src/services/reschedule.service.ts` | Service | requestReschedule (classifies ≥12h free vs. <12h same-day-miss), enforceMonthlyCap (2 free/month) | prisma, sessionMiss.service.ts |
| `src/controllers/reschedule.controller.ts` | Controller | requestReschedule handler | reschedule.service.ts |
| `src/routes/reschedule.routes.ts` | Route | mounts `/reschedule/*`; `authMiddleware` | reschedule.controller.ts |
| `src/services/sessionMiss.service.ts` | Service | recordTutorCausedMiss (queues free make-up within 7 days, 50% rate flag), recordStudentCausedMiss (no make-up), checkTutorEscalation (2+/30-day rolling) | prisma, notification.service.ts |
| `src/controllers/sessionMiss.controller.ts` | Controller | reportMiss (system/admin-triggered), listMisses handlers | sessionMiss.service.ts |
| `src/routes/sessionMiss.routes.ts` | Route | mounts `/session-miss/*`; `authMiddleware` | sessionMiss.controller.ts |
| `src/schemas/weeklyAssessment.schema.ts` | Zod schema | submitAssessmentSchema | — |
| `src/services/weeklyAssessment.service.ts` | Service | submitAssessment (tutor), getAssessmentsForStudent | prisma |
| `src/controllers/weeklyAssessment.controller.ts` | Controller | submit, listForMembership handlers | weeklyAssessment.service.ts |
| `src/routes/weeklyAssessment.routes.ts` | Route | mounts `/assessments/*`; `authMiddleware` | weeklyAssessment.controller.ts |
| `src/jobs/classReminder.job.ts` | Job | 1-hour-before reminder to tutor + student | notification.service.ts |
| `src/jobs/recordingMissingCheck.job.ts` | Job | flags `MISSING` at 2h, escalates at 24h | recording.service.ts |
| `tests/services/session.service.test.ts` | Test | mirrors session.service.ts | — |
| `tests/services/recordingConsent.service.test.ts` | Test | mirrors recordingConsent.service.ts — includes first-session-blocked-without-consent case | — |
| `tests/services/recording.service.test.ts` | Test | mirrors recording.service.ts — includes cross-student-access-denied case | — |
| `tests/services/reschedule.service.test.ts` | Test | mirrors reschedule.service.ts — includes 12h-boundary and monthly-cap cases | — |
| `tests/services/sessionMiss.service.test.ts` | Test | mirrors sessionMiss.service.ts — includes reduced-rate-flag and escalation-threshold cases | — |

---

## 5. Feature: `messaging`

**Owns:** MessageThread, Message. **Depends on:** `matching-cohorts`.

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/schemas/messaging.schema.ts` | Zod schema | sendMessageSchema | — |
| `src/services/messaging.service.ts` | Service | getThreadForCohort, sendMessage (blocked if not an active member), listMessages | prisma, notification.service.ts |
| `src/controllers/messaging.controller.ts` | Controller | getThread, sendMessage, listMessages handlers | messaging.service.ts |
| `src/routes/messaging.routes.ts` | Route | mounts `/messaging/*`; `authMiddleware` | messaging.controller.ts |
| `src/services/adminMessaging.service.ts` | Service | viewThreadForDispute, closeThread | prisma |
| `src/controllers/adminMessaging.controller.ts` | Controller | viewThread, closeThread handlers | adminMessaging.service.ts |
| `src/routes/adminMessaging.routes.ts` | Route | mounts `/admin/messaging/*`; `authMiddleware` + `requireRole(ADMIN)` | adminMessaging.controller.ts |
| `src/jobs/archiveMessageThreads.job.ts` | Job | archives threads 90 days after their Cohort's `endedAt` | messaging.service.ts |
| `tests/services/messaging.service.test.ts` | Test | mirrors messaging.service.ts — includes cohort-shared-thread vs. 1-to-1-private-thread cases | — |
| `tests/services/adminMessaging.service.test.ts` | Test | mirrors adminMessaging.service.ts | — |

---

## 6. Feature: `gamification-engagement`

**Owns:** XPLedgerEntry, Badge, StudentBadge, TutorBadge, Streak, Challenge, ChallengeProgress. **Depends on:** `accounts-guardianship` (hard); soft-integrates with `class-delivery-library` (XP-award trigger on class-attended).

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/services/xp.service.ts` | Service | awardXP (event-based), getLeaderboard (per-grade, first-name+last-initial, computed from ledger), adminAdjustXP (`reason: OTHER`, caller-supplied amount) — **I2 fix** | prisma |
| `src/controllers/xp.controller.ts` | Controller | getMyProgress, getLeaderboard, adminAdjust handlers — **I2 fix** | xp.service.ts |
| `src/schemas/xp.schema.ts` | Zod schema | adjustXPSchema — **I2 fix** | — |
| `src/routes/xp.routes.ts` | Route | mounts `/gamification/xp/*`, `/gamification/leaderboard`, `/admin/students/:studentId/xp-adjustments` (Admin-only) — **I2 fix**; `authMiddleware` | xp.controller.ts |
| `src/services/badge.service.ts` | Service | createBadge (admin) — **I2 fix**, awardStudentBadge, awardTutorBadge, adminManageBadges (criteria review, never rating-based) | prisma |
| `src/controllers/badge.controller.ts` | Controller | listMyBadges, adminListAll, adminCreate, adminAdjust handlers — **I2 fix** | badge.service.ts |
| `src/schemas/badge.schema.ts` | Zod schema | createBadgeSchema — **I2 fix** | — |
| `src/routes/badge.routes.ts` | Route | mounts `/gamification/badges/*` and `/admin/badges/*` (GET, POST, PATCH — **I2 fix**); `authMiddleware` | badge.controller.ts |
| `src/services/streak.service.ts` | Service | updateStreakOnActivity, resetStreakOnGap (internal, triggered by activity events) | prisma |
| `tests/services/streak.service.test.ts` | Test | mirrors streak.service.ts | — |
| `src/schemas/challenge.schema.ts` | Zod schema | createChallengeSchema | — |
| `src/services/challenge.service.ts` | Service | createChallenge (admin), listActiveChallenges, trackProgress | prisma |
| `src/controllers/challenge.controller.ts` | Controller | listActive, adminCreate, getMyProgress handlers | challenge.service.ts |
| `src/routes/challenge.routes.ts` | Route | mounts `/gamification/challenges/*` and `/admin/challenges/*`; `authMiddleware` | challenge.controller.ts |
| `tests/services/xp.service.test.ts` | Test | mirrors xp.service.ts — includes per-grade-scoping and name-display cases | — |
| `tests/services/badge.service.test.ts` | Test | mirrors badge.service.ts — includes no-rating-field-exists case | — |
| `tests/services/challenge.service.test.ts` | Test | mirrors challenge.service.ts | — |

---

## 7. Feature: `payments-earnings`

**Owns:** PricingConfig, Payment, PaymentPause, Refund, TutorEarning, Payout, PromotionCode. **Depends on:** `matching-cohorts`, `class-delivery-library` (TutorEarning → ScheduledSession).

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/utils/providers/chapa.client.ts` | Util | wraps Chapa payment initiation + webhook verification | env config |
| `src/schemas/payment.schema.ts` | Zod schema | initiatePaymentSchema, applyPromotionSchema | — |
| `src/services/payment.service.ts` | Service | initiatePayment, handleChapaWebhook, getPaymentHistory, setBillingCycleAnchor (first successful payment) | prisma, chapa.client.ts, cohort.service.ts (matching-cohorts) |
| `src/controllers/payment.controller.ts` | Controller | initiate, webhook, getHistory handlers | payment.service.ts |
| `src/routes/payment.routes.ts` | Route | mounts `/payments/*`; webhook route unauthenticated (signature-verified instead), rest `authMiddleware` | payment.controller.ts |
| `src/services/paymentPause.service.ts` | Service | pauseForNonPayment, resumeOnPayment, rescheduleSessionsDuringPause (no miss/refund) | prisma, session.service.ts (class-delivery-library) |
| `src/controllers/paymentPause.controller.ts` | Controller | getPauseStatus handler | paymentPause.service.ts |
| `src/routes/paymentPause.routes.ts` | Route | mounts `/payment-pause/*`; `authMiddleware` | paymentPause.controller.ts |
| `src/schemas/pricing.schema.ts` | Zod schema | updatePricingConfigSchema | — |
| `src/services/pricing.service.ts` | Service | getActiveConfig (per format), createAndActivateConfig (versioned, deactivates old) | prisma |
| `src/controllers/pricing.controller.ts` | Controller | getActive, adminUpdate handlers | pricing.service.ts |
| `src/routes/pricing.routes.ts` | Route | mounts `/pricing/*` (public GET) and `/admin/pricing/*` (admin mutate) | pricing.controller.ts |
| `src/services/refund.service.ts` | Service | calculateProration (sessions-delivered formula), createPendingRefund, approveRefund, rejectRefund — **I1 fix** | prisma |
| `src/schemas/refund.schema.ts` | Zod schema | rejectRefundSchema — **I1 fix** | — |
| `src/controllers/refund.controller.ts` | Controller | adminReview, adminApprove, adminReject handlers — **I1 fix** | refund.service.ts |
| `src/routes/refund.routes.ts` | Route | mounts `/admin/refunds/*`; `authMiddleware` + `requireRole(ADMIN)` | refund.controller.ts |
| `src/services/earning.service.ts` | Service | creditEarning (FULL or REDUCED_MAKEUP rate), getEarningsForTutor | prisma |
| `src/controllers/earning.controller.ts` | Controller | getMyEarnings handler | earning.service.ts |
| `src/routes/earning.routes.ts` | Route | mounts `/tutors/me/earnings/*`; `authMiddleware` | earning.controller.ts |
| `src/services/payout.service.ts` | Service | generateMonthlyPayouts (batches unpaid TutorEarning rows), markPaid, adminAdjust | prisma |
| `src/controllers/payout.controller.ts` | Controller | adminList, adminMarkPaid handlers | payout.service.ts |
| `src/routes/payout.routes.ts` | Route | mounts `/admin/payouts/*`; `authMiddleware` + `requireRole(ADMIN)` | payout.controller.ts |
| `src/schemas/promotion.schema.ts` | Zod schema | createPromotionSchema | — |
| `src/services/promotion.service.ts` | Service | createPromotion (admin), listActivePromotions, applyToPayment | prisma |
| `src/controllers/promotion.controller.ts` | Controller | adminCreate, listActive handlers | promotion.service.ts |
| `src/routes/promotion.routes.ts` | Route | mounts `/promotions/*` (public GET active) and `/admin/promotions/*` (admin mutate) | promotion.controller.ts |
| `src/jobs/paymentReminder.job.ts` | Job | sends the 3-day-before reminder, anchored per-student | payment.service.ts, notification.service.ts |
| `src/jobs/monthlyPayout.job.ts` | Job | runs `payout.service.generateMonthlyPayouts` on a fixed monthly cycle | payout.service.ts |
| `tests/services/payment.service.test.ts` | Test | mirrors payment.service.ts | — |
| `tests/services/paymentPause.service.test.ts` | Test | mirrors paymentPause.service.ts — includes no-miss-no-refund-during-pause case | — |
| `tests/services/pricing.service.test.ts` | Test | mirrors pricing.service.ts — includes versioning/next-booking-only case | — |
| `tests/services/refund.service.test.ts` | Test | mirrors refund.service.ts — includes sessions-delivered proration formula case | — |
| `tests/services/earning.service.test.ts` | Test | mirrors earning.service.ts | — |
| `tests/services/payout.service.test.ts` | Test | mirrors payout.service.ts — includes no-manual-request-step case | — |
| `tests/services/promotion.service.test.ts` | Test | mirrors promotion.service.ts | — |

---

## 8. Feature: `support-trust-admin`

**Owns:** ComplaintReport. **Depends on:** `shared-config`, `messaging`, `class-delivery-library` (hard, via `relatedThreadId`/`relatedSessionId`); soft-integrates with `payments-earnings` (refund action from a dispute, via `relatedPaymentId`/`relatedCohortId`) and `accounts-guardianship` (suspension action from a dispute).

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/schemas/complaint.schema.ts` | Zod schema | fileComplaintSchema | — |
| `src/services/complaint.service.ts` | Service | fileComplaint, getMyComplaints | prisma |
| `src/controllers/complaint.controller.ts` | Controller | file, listMine handlers | complaint.service.ts |
| `src/routes/complaint.routes.ts` | Route | mounts `/complaints/*`; `authMiddleware` | complaint.controller.ts |
| `src/services/adminDispute.service.ts` | Service | listQueue, reviewComplaint (pulls linked thread/session), resolveComplaint (may call refund/suspend/re-match from other features) | prisma, adminMessaging.service.ts, refund.service.ts, adminPeople.service.ts |
| `src/controllers/adminDispute.controller.ts` | Controller | listQueue, review, resolve handlers | adminDispute.service.ts |
| `src/routes/adminDispute.routes.ts` | Route | mounts `/admin/disputes/*`; `authMiddleware` + `requireRole(ADMIN)` | adminDispute.controller.ts |
| `src/services/adminReporting.service.ts` | Service | getPlatformStatistics, getActivityHistory, getTutorPerformanceHistory | prisma |
| `src/controllers/adminReporting.controller.ts` | Controller | getStats, getActivity, getTutorPerformance handlers | adminReporting.service.ts |
| `src/routes/adminReporting.routes.ts` | Route | mounts `/admin/reports/*`; `authMiddleware` + `requireRole(ADMIN)` | adminReporting.controller.ts |
| `tests/services/complaint.service.test.ts` | Test | mirrors complaint.service.ts | — |
| `tests/services/adminDispute.service.test.ts` | Test | mirrors adminDispute.service.ts | — |
| `tests/services/adminReporting.service.test.ts` | Test | mirrors adminReporting.service.ts | — |

---

## 9. Schema/Config Changes

| File | Change |
|---|---|
| `.env.example` | Add `CHAPA_API_KEY`, `GEEZ_SMS_API_KEY`, `BREVO_API_KEY`, `CLOUDFLARE_R2_ACCESS_KEY`, `CLOUDFLARE_R2_SECRET_KEY`, `CLOUDFLARE_R2_BUCKET`, `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD`. (`JWT_SECRET` is already present in the template's `.env.example` — not re-added.) Change `JWT_EXPIRES_IN`'s default from the template's `7d` to `30m`, per NFR-014's 30-minute access-token TTL. |
| `src/config/env.ts` | **Fix (was missing):** extend `envSchema` with all eight new keys above (`CHAPA_API_KEY`, `GEEZ_SMS_API_KEY`, `BREVO_API_KEY`, `CLOUDFLARE_R2_ACCESS_KEY`, `CLOUDFLARE_R2_SECRET_KEY`, `CLOUDFLARE_R2_BUCKET`, `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD`), all `z.string()` and required (none should default-empty). Listing a var in `.env.example` alone is not enough — `envSchema` uses `z.object()`, which strips any key not explicitly declared, so an unlisted var reads as `undefined` via `env.X` even when present in `.env`. Also update `JWT_EXPIRES_IN`'s schema default to `'30m'` to match the `.env.example` change above. |
| `prisma/schema.prisma` | Add all 40 models + enums from Doc 04 |
| `package.json` | Add dependencies: Prisma client, Zod, bcrypt, jsonwebtoken, a Chapa SDK/HTTP client, an R2-compatible S3 client (`@aws-sdk/client-s3`), a cron/job scheduler |
| `src/routes/index.ts` | Register every feature's routers, grouped by feature comment blocks in build order (Section 10) |

---

## 10. File Creation Order

Follows the parallelizable build order from `feature-decomposition.md`. Steps within the same numbered phase can be built concurrently by different people.

**Phase 0 — Foundation**
1. `prisma/schema.prisma` (migrate + generate)
2. `prisma/seed.ts`
3. Section 0 cross-cutting files (middleware, jwt/password utils, `src/config/db.ts`, scheduler)

**Phase 1 — `shared-config`**
4. `auth.schema.ts` → `auth.service.ts` → `auth.controller.ts` → `auth.routes.ts`
5. `sms.client.ts`, `email.client.ts`
6. `notification.service.ts` → `notification.controller.ts` → `notification.routes.ts`
7. `adminAnnouncement.*`
8. `policy.schema.ts` → `policy.service.ts` → `policy.controller.ts` → `policy.routes.ts`
9. `notificationRetry.job.ts`

**Phase 2 — `accounts-guardianship`**
10. `studentProfile.*`, `tutorProfile.*`
11. `guardianship.*` → `inviteReminder.job.ts`
12. `availability.*`
13. `subject.*`
14. `adminTutorVerification.*`, `adminPeople.*`

**Phase 3 — `matching-cohorts`**
15. `cohort.service.ts` (built before `matching.service.ts`, which depends on it)
16. `matching.*`
17. `adminMatching.*`
18. `formatSwitch.*`
19. `groupFormationWindow.job.ts`, `zeroMatchEscalation.job.ts`, `staleApproval.job.ts`

**Phase 4 — Parallel: `class-delivery-library` / `messaging` / `gamification-engagement`**
20a. `session.*`, `recordingConsent.*`, `storage.client.ts` → `recording.*`, `library.*`, `reschedule.*`, `sessionMiss.*`, `weeklyAssessment.*`, then `classReminder.job.ts` / `recordingMissingCheck.job.ts`
20b. `messaging.*`, `adminMessaging.*`, `archiveMessageThreads.job.ts`
20c. `xp.*`, `badge.*`, `streak.service.ts`, `challenge.*`

**Phase 5 — `payments-earnings` and `support-trust-admin`**
21. `chapa.client.ts` → `payment.*` → `paymentPause.*`
22. `pricing.*`, `refund.*`, `earning.*`, `payout.*`, `promotion.*`
23. `paymentReminder.job.ts`, `monthlyPayout.job.ts`
24. `complaint.*` → `adminDispute.*` (depends on messaging + refund + people-suspension being complete)
25. `adminReporting.*`
26. `src/routes/index.ts` (register everything — last)
27. Remaining test files, per the standing test-mirrors-service rule

---

**Next:** proceed to → [05b. Frontend Structure]
