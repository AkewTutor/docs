# AKEWTutor — Frontend Folder & File Structure

**Project:** AKEWTutor — Online Tutoring Platform

**Links back to:** [04. Database & Data Model], [Feature Decomposition], [05a. Backend Structure]
**Links forward to:** [06. API Specification], [07. Frontend Specification], [10. UI Foundation Spec]

**Base template:** React (same convention as the reference project's `template-react`) — `src/{pages,components,hooks,store,routes,lib,types,constants,styles}`, React Query for data fetching, Zustand for client state, Vitest for hook tests.

**Structuring principle:** Mirrors the backend's 8 features 1:1 (Section 2 onward) so the same pair can own a feature end-to-end. Feature 0, `ui-foundation`, has no backend equivalent — see the note in `feature-decomposition.md` §3.

**See `10-ui-foundation-spec.md`** for the full design-token, common-component, and layout spec — this file only lists what exists where; that document defines what each foundation piece contains and why.

**Standing rule (carried over from the reference project):** every hook file gets a mirrored test file under `tests/`.

> **Spacing namespace rule (carried over unchanged):** project-specific spacing tokens use the `space-*` namespace (`mt-space-sm`, `gap-space-md`, `p-space-lg`, `px-space-gutter`, `py-space-margin-mobile`) to avoid colliding with Tailwind's built-in `sm/md/lg/xl` scale. Standard Tailwind utilities (`md:flex`, `gap-4`, `px-6`, `max-w-md`, etc.) are untouched and must not be renamed.

---

## 0. Feature: `ui-foundation` (frontend-only, no backend equivalent)

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/styles/globals.css` | Styles | design tokens + namespaced `space-*` scale | — |
| `index.html` | Config | font links | — |
| `src/lib/utils.ts` | Util | `cn()` class-merge helper | — |
| `src/components/ui/Button.tsx`, `Card.tsx`, `Input.tsx`, `Label.tsx`, `Checkbox.tsx`, `Badge.tsx` | Component | base primitives per `10-ui-foundation-spec.md` | — |
| `src/components/common/StatusBadge.tsx` | Component | consolidated status pill (approved/pending/overdue/missing/etc.) | — |
| `src/components/common/EmptyState.tsx` | Component | shared empty-state pattern (no matches yet, no payments yet, etc.) | — |
| `src/components/common/CountdownTimer.tsx` | Component | shared countdown used by class reminders, payment reminders, invite expiry | — |
| `tests/components/*.test.tsx` | Test | mirrors each `ui/` primitive that has non-trivial logic | — |

---

## 1. Feature: `shared-config`

**Owns UI for:** registration/login, notifications, policy pages, platform announcements.

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/LoginPage.tsx` | Page | role-aware login form | useAuth |
| `src/pages/RegisterPage.tsx` | Page | student/parent/tutor registration entry | useAuth |
| `src/pages/VerifyContactPage.tsx` | Page | email/phone verification code entry | useAuth |
| `src/pages/ForgotPasswordPage.tsx`, `ResetPasswordPage.tsx` | Page | password reset flow | useAuth |
| `src/pages/PolicyPage.tsx` | Page | renders one policy type by route param (Privacy/Terms/Safety/Refund/Rules) | usePolicy |
| `src/pages/NotificationsPage.tsx` | Page | notification center | useNotifications |
| `src/pages/admin/AnnouncementsPage.tsx` | Page | admin composes/lists platform announcements | useAdminAnnouncements |

### Layouts & Route Guards

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/components/layouts/PublicLayout.tsx` | Layout | public chrome, no auth awareness | TopNavBar, Footer |
| `src/components/layouts/AuthLayout.tsx` | Layout | login/register chrome | — |
| `src/components/layouts/DashboardLayout.tsx` | Layout | role-aware app chrome, composes the correct sidebar per role, renders `<Outlet />` | StudentSidebar/TutorSidebar/AdminSidebar (§2, §8) |
| `src/routes/index.tsx` | Routes | mounts PublicLayout (ungated), AuthLayout/PublicRoute, DashboardLayout/ProtectedRoute (role-checked) | all layouts/guards |
| `src/routes/ProtectedRoute.tsx`, `PublicRoute.tsx` | Guard | auth/role gating | authStore |

### Store, Hooks, Components

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/store/auth.store.ts` | Store | Zustand, persisted `auth-storage` — token + user identity/role | — |
| `src/hooks/useAuth.ts` | Hook | useRegister, useLogin, useLogout, useVerifyContact, usePasswordReset | src/lib/axios.ts, auth.store.ts |
| `tests/hooks/useAuth.test.ts` | Test | mirrors useAuth.ts | — |
| `src/hooks/useNotifications.ts` | Hook | useMyNotifications, useMarkRead | src/lib/axios.ts |
| `tests/hooks/useNotifications.test.ts` | Test | mirrors useNotifications.ts | — |
| `src/hooks/usePolicy.ts` | Hook | usePolicy(type) | src/lib/axios.ts |
| `tests/hooks/usePolicy.test.ts` | Test | mirrors usePolicy.ts | — |
| `src/hooks/useAdminAnnouncements.ts` | Hook | useCreateAnnouncement, useListAnnouncements | src/lib/axios.ts |
| `tests/hooks/useAdminAnnouncements.test.ts` | Test | mirrors useAdminAnnouncements.ts | — |
| `src/components/common/TopNavBar.tsx` | Component | public/dashboard nav, active-link styling via `useLocation()` | ROUTES |
| `src/components/common/Footer.tsx` | Component | static public footer | — |
| `src/components/common/NotificationBell.tsx` | Component | header notification indicator + dropdown | useNotifications |

---

## 2. Feature: `accounts-guardianship`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/student/AcademicProfilePage.tsx` | Page | grade/school/subjects/goals/language/schedule/budget/format form | useStudentProfile |
| `src/pages/parent/AddStudentPage.tsx` | Page | grade entry + invite send | useGuardianship |
| `src/pages/InviteActivationPage.tsx` | Page | student activates a parent-issued invite | useGuardianship |
| `src/pages/parent/GuardianSettingsPage.tsx` | Page | manage/revoke relationships, invite optional guardian (6-12) | useGuardianship |
| `src/pages/tutor/TutorProfilePage.tsx` | Page | qualifications, experience, education | useTutorProfile |
| `src/pages/tutor/SubjectRankingPage.tsx` | Page | rank up to 2 subjects | useTutorProfile |
| `src/pages/tutor/AvailabilityPage.tsx` | Page | manage availability slots | useAvailability |
| `src/pages/admin/TutorVerificationPage.tsx` | Page | review/approve/reject pending tutors | useAdminTutorVerification |
| `src/pages/admin/PeopleManagementPage.tsx` | Page | manage students/parents/tutors, suspend/restrict | useAdminPeople |
| `src/pages/admin/SubjectManagementPage.tsx` | Page | CRUD subjects | useSubjects |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/accounts/GuardianInviteStatusCard.tsx` | Component | invite state (invited/expiring/active) + resend action |
| `src/components/accounts/SubjectRankingForm.tsx` | Component | drag-to-rank UI, hard-blocks a 3rd subject |
| `src/components/accounts/AvailabilityCalendar.tsx` | Component | slot add/remove grid |
| `src/components/accounts/TutorVerificationCard.tsx` | Component | one pending tutor's review card |
| `src/components/common/StudentSidebar.tsx`, `TutorSidebar.tsx` | Component | role-specific dashboard nav (used by `DashboardLayout`) |

### Store, Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useStudentProfile.ts` | Hook | useMyStudentProfile, useUpdateAcademicProfile | src/lib/axios.ts |
| `tests/hooks/useStudentProfile.test.ts` | Test | mirrors useStudentProfile.ts | — |
| `src/hooks/useGuardianship.ts` | Hook | useAddStudent, useResendInvite, useActivateInvite, useInviteGuardian, useRevokeRelationship | src/lib/axios.ts |
| `tests/hooks/useGuardianship.test.ts` | Test | mirrors useGuardianship.ts | — |
| `src/hooks/useTutorProfile.ts` | Hook | useMyTutorProfile, useUpdateTutorProfile, useRankSubjects | src/lib/axios.ts |
| `tests/hooks/useTutorProfile.test.ts` | Test | mirrors useTutorProfile.ts | — |
| `src/hooks/useAvailability.ts` | Hook | useMyAvailability, useAddSlot, useRemoveSlot | src/lib/axios.ts |
| `tests/hooks/useAvailability.test.ts` | Test | mirrors useAvailability.ts | — |
| `src/hooks/useSubjects.ts` | Hook | useSubjects, useCreateSubject, useDeactivateSubject | src/lib/axios.ts |
| `tests/hooks/useSubjects.test.ts` | Test | mirrors useSubjects.ts | — |
| `src/hooks/useAdminTutorVerification.ts` | Hook | usePendingTutors, useApproveTutor, useRejectTutor | src/lib/axios.ts |
| `tests/hooks/useAdminTutorVerification.test.ts` | Test | mirrors useAdminTutorVerification.ts | — |
| `src/hooks/useAdminPeople.ts` | Hook | useUsers, useSuspendAccount, useEditRelationship | src/lib/axios.ts |
| `tests/hooks/useAdminPeople.test.ts` | Test | mirrors useAdminPeople.ts | — |

---

## 3. Feature: `matching-cohorts`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/student/FindTutorPage.tsx` | Page | 1-to-1 search + filters | useMatching |
| `src/pages/student/TutorRecommendationsPage.tsx` | Page | match-percentage list, "No Exact Match" trigger | useMatching |
| `src/pages/student/TutorProfileViewPage.tsx` | Page | full 1-to-1 profile view | useMatching |
| `src/pages/student/GroupFormatStatusPage.tsx` | Page | Path C — waiting/assigned state, name+photo only | useCohort |
| `src/pages/student/FormatSwitchPage.tsx` | Page | request a format change | useFormatSwitch |
| `src/pages/admin/MatchingQueuePage.tsx` | Page | pending bookings/auto-matches, overdue-flagged | useAdminMatching |
| `src/pages/admin/ManualAssignmentPage.tsx` | Page | Path B/C manual assembly | useAdminMatching |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/matching/TutorSearchFilters.tsx` | Component | subject/grade/schedule/budget/language filters (1-to-1 only) |
| `src/components/matching/TutorRecommendationCard.tsx` | Component | match % + summary card |
| `src/components/matching/GroupAssignmentCard.tsx` | Component | name+photo-only assigned-tutor card for group formats |
| `src/components/matching/NoExactMatchButton.tsx` | Component | manual Path B trigger + zero-match countdown display |
| `src/components/admin-matching/ApprovalQueueTable.tsx` | Component | pending approvals, overdue rows highlighted |
| `src/components/admin-matching/ManualAssignmentForm.tsx` | Component | Admin picks a tutor / assembles a group |

### Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useMatching.ts` | Hook | useSearchTutors, useRecommendations, useSelectTutor, useNoExactMatch, useRequestGroupFormat | src/lib/axios.ts |
| `tests/hooks/useMatching.test.ts` | Test | mirrors useMatching.ts | — |
| `src/hooks/useCohort.ts` | Hook | useMyCohort, useCohortMembers | src/lib/axios.ts |
| `tests/hooks/useCohort.test.ts` | Test | mirrors useCohort.ts | — |
| `src/hooks/useAdminMatching.ts` | Hook | useApprovalQueue, useApprove, useReject, useManualAssign | src/lib/axios.ts |
| `tests/hooks/useAdminMatching.test.ts` | Test | mirrors useAdminMatching.ts | — |
| `src/hooks/useFormatSwitch.ts` | Hook | useRequestFormatSwitch | src/lib/axios.ts |
| `tests/hooks/useFormatSwitch.test.ts` | Test | mirrors useFormatSwitch.ts | — |

---

## 4. Feature: `class-delivery-library`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/student/UpcomingClassesPage.tsx` | Page | scheduled sessions + countdown + join link | useSessions |
| `src/pages/tutor/ConductClassPage.tsx` | Page | provide Jitsi link, mark completed, upload recording | useSessions, useRecordings |
| `src/pages/LibraryPage.tsx` | Page | student's recordings + materials | useLibrary, useRecordings |
| `src/pages/RecordingConsentPage.tsx` | Page | one-time consent acknowledgment modal/page | useRecordingConsent |
| `src/pages/RequestReschedulePage.tsx` | Page | reschedule request form | useReschedule |
| `src/pages/tutor/WeeklyAssessmentPage.tsx` | Page | submit weekly feedback per student | useWeeklyAssessment |
| `src/pages/student/ProgressPage.tsx` | Page | assessment history + tutor feedback | useWeeklyAssessment |
| `src/pages/admin/RecordingComplianceQueuePage.tsx` | Page | "recording missing" escalation queue | useRecordings |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/class-delivery/SessionCard.tsx` | Component | one scheduled session, countdown + join button |
| `src/components/class-delivery/RecordingIndicatorBanner.tsx` | Component | persistent in-session recording-in-progress indicator |
| `src/components/class-delivery/RecordingPlayer.tsx` | Component | signed-URL playback |
| `src/components/class-delivery/MaterialUploadForm.tsx` | Component | tutor uploads PDF/note/book |
| `src/components/class-delivery/RescheduleForm.tsx` | Component | new-time picker, shows free-vs-same-day-miss classification live |
| `src/components/class-delivery/AssessmentForm.tsx` | Component | tutor feedback + score summary entry |

### Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useSessions.ts` | Hook | useUpcomingSessions, useProvideLink, useMarkCompleted | src/lib/axios.ts |
| `tests/hooks/useSessions.test.ts` | Test | mirrors useSessions.ts | — |
| `src/hooks/useRecordingConsent.ts` | Hook | useConsentStatus, useAcknowledgeConsent | src/lib/axios.ts |
| `tests/hooks/useRecordingConsent.test.ts` | Test | mirrors useRecordingConsent.ts | — |
| `src/hooks/useRecordings.ts` | Hook | useUpload, useMyRecordings, useSignedUrl, useKeepPermanently, useComplianceQueue | src/lib/axios.ts |
| `tests/hooks/useRecordings.test.ts` | Test | mirrors useRecordings.ts | — |
| `src/hooks/useLibrary.ts` | Hook | useCohortMaterials, useUploadMaterial | src/lib/axios.ts |
| `tests/hooks/useLibrary.test.ts` | Test | mirrors useLibrary.ts | — |
| `src/hooks/useReschedule.ts` | Hook | useRequestReschedule | src/lib/axios.ts |
| `tests/hooks/useReschedule.test.ts` | Test | mirrors useReschedule.ts — includes 12h-boundary display case | — |
| `src/hooks/useWeeklyAssessment.ts` | Hook | useSubmitAssessment, useAssessmentsForStudent | src/lib/axios.ts |
| `tests/hooks/useWeeklyAssessment.test.ts` | Test | mirrors useWeeklyAssessment.ts | — |

---

## 5. Feature: `messaging`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/MessagingPage.tsx` | Page | thread view — private pair (1-to-1) or shared cohort thread (group) | useMessaging |
| `src/pages/admin/MessageThreadReviewPage.tsx` | Page | dispute-linked thread viewer, close/report action | useAdminMessaging |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/messaging/MessageThreadView.tsx` | Component | scrollable message list, renders differently for pair vs. cohort thread |
| `src/components/messaging/MessageComposer.tsx` | Component | text-only input (no attachment UI, matching the text-only rule) |

### Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useMessaging.ts` | Hook | useThread, useSendMessage, useMessages | src/lib/axios.ts |
| `tests/hooks/useMessaging.test.ts` | Test | mirrors useMessaging.ts | — |
| `src/hooks/useAdminMessaging.ts` | Hook | useReviewThread, useCloseThread | src/lib/axios.ts |
| `tests/hooks/useAdminMessaging.test.ts` | Test | mirrors useAdminMessaging.ts | — |

---

## 6. Feature: `gamification-engagement`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/student/AchievementsPage.tsx` | Page | XP, badges, streak display | useGamification |
| `src/pages/student/LeaderboardPage.tsx` | Page | per-grade leaderboard | useGamification |
| `src/pages/student/ChallengesPage.tsx` | Page | active weekly/monthly challenges | useChallenges |
| `src/pages/admin/BadgeManagementPage.tsx` | Page | manage badge criteria/awards | useAdminGamification |
| `src/pages/admin/ChallengeManagementPage.tsx` | Page | create/manage challenges | useChallenges |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/gamification/XPProgressBar.tsx` | Component | current XP / level display |
| `src/components/gamification/BadgeGrid.tsx` | Component | earned badges |
| `src/components/gamification/StreakFlame.tsx` | Component | current/longest streak indicator |
| `src/components/gamification/LeaderboardTable.tsx` | Component | grade-scoped ranking, first-name + last-initial only |
| `src/components/gamification/ChallengeCard.tsx` | Component | one active challenge + progress |

### Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useGamification.ts` | Hook | useMyProgress, useLeaderboard | src/lib/axios.ts |
| `tests/hooks/useGamification.test.ts` | Test | mirrors useGamification.ts | — |
| `src/hooks/useAdminGamification.ts` | Hook | useAllBadges, useAdjustBadge | src/lib/axios.ts |
| `tests/hooks/useAdminGamification.test.ts` | Test | mirrors useAdminGamification.ts | — |
| `src/hooks/useChallenges.ts` | Hook | useActiveChallenges, useMyChallengeProgress, useCreateChallenge (admin) | src/lib/axios.ts |
| `tests/hooks/useChallenges.test.ts` | Test | mirrors useChallenges.ts | — |

---

## 7. Feature: `payments-earnings`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/PaymentPage.tsx` | Page | Chapa checkout hand-off after Admin approval | usePayments |
| `src/pages/PaymentHistoryPage.tsx` | Page | past payments | usePayments |
| `src/pages/PaymentPausedPage.tsx` | Page | "Pay for this month" blocking screen | usePayments |
| `src/pages/tutor/EarningsPage.tsx` | Page | earnings + upcoming payout, reduced-rate sessions itemized | useEarnings |
| `src/pages/admin/PricingConfigPage.tsx` | Page | manage price/revenue-split per format | usePricing |
| `src/pages/admin/RefundReviewPage.tsx` | Page | review/approve refunds | useRefunds |
| `src/pages/admin/PayoutManagementPage.tsx` | Page | monthly payout oversight | usePayouts |
| `src/pages/admin/PromotionManagementPage.tsx` | Page | CRUD promo codes | usePromotions |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/payments/ChapaCheckoutButton.tsx` | Component | initiates Chapa redirect/hand-off |
| `src/components/payments/PaymentHistoryTable.tsx` | Component | past transactions |
| `src/components/payments/PaymentReminderBanner.tsx` | Component | 3-day countdown, per-student anchored |
| `src/components/admin-payments/PricingConfigForm.tsx` | Component | format price/split editor |
| `src/components/admin-payments/RefundCard.tsx` | Component | one refund case, proration breakdown shown |
| `src/components/admin-payments/PayoutBatchTable.tsx` | Component | payout batches, itemized full vs. reduced-rate |
| `src/components/admin-payments/PromotionForm.tsx` | Component | create/edit a promo code |

### Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/usePayments.ts` | Hook | useInitiatePayment, useMyPayments, usePauseStatus | src/lib/axios.ts |
| `tests/hooks/usePayments.test.ts` | Test | mirrors usePayments.ts | — |
| `src/hooks/useEarnings.ts` | Hook | useMyEarnings | src/lib/axios.ts |
| `tests/hooks/useEarnings.test.ts` | Test | mirrors useEarnings.ts | — |
| `src/hooks/usePricing.ts` | Hook | useActivePricing, useUpdatePricing | src/lib/axios.ts |
| `tests/hooks/usePricing.test.ts` | Test | mirrors usePricing.ts | — |
| `src/hooks/useRefunds.ts` | Hook | useRefundQueue, useApproveRefund | src/lib/axios.ts |
| `tests/hooks/useRefunds.test.ts` | Test | mirrors useRefunds.ts | — |
| `src/hooks/usePayouts.ts` | Hook | usePayoutBatches, useMarkPaid | src/lib/axios.ts |
| `tests/hooks/usePayouts.test.ts` | Test | mirrors usePayouts.ts | — |
| `src/hooks/usePromotions.ts` | Hook | useActivePromotions, useCreatePromotion | src/lib/axios.ts |
| `tests/hooks/usePromotions.test.ts` | Test | mirrors usePromotions.ts | — |

---

## 8. Feature: `support-trust-admin`

### Pages

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/SubmitComplaintPage.tsx` | Page | file a complaint/claim/report | useComplaints |
| `src/pages/SupportContactPage.tsx` | Page | support channel + emergency contact (phone/Telegram) | — (static content) |
| `src/pages/admin/DisputeQueuePage.tsx` | Page | complaint/dispute review queue | useAdminDisputes |
| `src/pages/admin/PlatformReportsPage.tsx` | Page | statistics, activity history, tutor performance | useAdminReporting |

### Feature Components

| File | Type | Purpose |
|---|---|---|
| `src/components/support/ComplaintForm.tsx` | Component | complaint/report submission |
| `src/components/admin-support/DisputeCard.tsx` | Component | one dispute, linked thread/session context, resolve actions |
| `src/components/admin-support/AdminSidebar.tsx` | Component | admin-wide sidebar nav (used by `DashboardLayout` for the Admin role) |
| `src/components/admin-support/PlatformStatsGrid.tsx` | Component | headline platform stats |
| `src/components/admin-support/TutorPerformanceTable.tsx` | Component | tutor performance/badge history |

### Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useComplaints.ts` | Hook | useFileComplaint, useMyComplaints | src/lib/axios.ts |
| `tests/hooks/useComplaints.test.ts` | Test | mirrors useComplaints.ts | — |
| `src/hooks/useAdminDisputes.ts` | Hook | useDisputeQueue, useReviewDispute, useResolveDispute | src/lib/axios.ts |
| `tests/hooks/useAdminDisputes.test.ts` | Test | mirrors useAdminDisputes.ts | — |
| `src/hooks/useAdminReporting.ts` | Hook | usePlatformStats, useActivityHistory, useTutorPerformance | src/lib/axios.ts |
| `tests/hooks/useAdminReporting.test.ts` | Test | mirrors useAdminReporting.ts | — |

---

## 9. Schema/Config Changes

| File | Change |
|---|---|
| `src/types/index.ts` | Add all types across the 8 features (StudentProfile, TutorProfile, Cohort, ScheduledSession, MessageThread, XPLedgerEntry, Payment, ComplaintReport, etc.) |
| `src/constants/index.ts` | Add `QUERY_KEYS` for every hook above; add `ROUTES` for every page above, grouped by feature comment blocks |
| `src/lib/axios.ts` | 401 interceptor redirects to the correct login route based on the failing request's role prefix (`/students/`, `/tutors/`, `/admin/`) rather than a single hardcoded admin redirect, since AKEWTutor has three authenticated roles, not one |
| `.env.example` | Confirm `VITE_API_URL` includes `/api/v1` |

---

## 10. File Creation Order

**Phase 0 — UI Foundation**
See `10-ui-foundation-spec.md` for full reasoning; must complete before Phase 1.
1. `src/styles/globals.css`, `index.html`, `src/lib/utils.ts`
2. `src/components/ui/*`
3. `src/components/common/StatusBadge.tsx`, `EmptyState.tsx`, `CountdownTimer.tsx`

**Phase 1 — Shared Config**
Routing/state must exist before any nav/sidebar can build ROUTES-aware active-link logic.
4. `src/types/index.ts`, `src/constants/index.ts`
5. `src/store/auth.store.ts`
6. `src/lib/axios.ts`
7. `useAuth.ts` → `LoginPage.tsx`, `RegisterPage.tsx`, `VerifyContactPage.tsx`, `ForgotPasswordPage.tsx`, `ResetPasswordPage.tsx`
8. `usePolicy.ts` → `PolicyPage.tsx`
9. `useNotifications.ts` → `NotificationBell.tsx`, `NotificationsPage.tsx`
10. `useAdminAnnouncements.ts` → `AnnouncementsPage.tsx`
11. `TopNavBar.tsx`, `Footer.tsx` → `PublicLayout.tsx`, `AuthLayout.tsx`
12. `ProtectedRoute.tsx`, `PublicRoute.tsx`

**Phase 2 — `accounts-guardianship`**
13. `useStudentProfile.ts`, `useTutorProfile.ts`, `useAvailability.ts`, `useSubjects.ts`
14. `StudentSidebar.tsx`, `TutorSidebar.tsx` → `DashboardLayout.tsx` (modify) → `src/routes/index.tsx` (first pass)
15. `AcademicProfilePage.tsx`, `TutorProfilePage.tsx`, `SubjectRankingPage.tsx`, `AvailabilityPage.tsx`
16. `useGuardianship.ts` → `AddStudentPage.tsx`, `InviteActivationPage.tsx`, `GuardianSettingsPage.tsx`
17. `useAdminTutorVerification.ts`, `useAdminPeople.ts` → `TutorVerificationPage.tsx`, `PeopleManagementPage.tsx`, `SubjectManagementPage.tsx`

**Phase 3 — `matching-cohorts`**
18. `useMatching.ts`, `useCohort.ts` → `FindTutorPage.tsx`, `TutorRecommendationsPage.tsx`, `TutorProfileViewPage.tsx`, `GroupFormatStatusPage.tsx`
19. `useAdminMatching.ts` → `MatchingQueuePage.tsx`, `ManualAssignmentPage.tsx`
20. `useFormatSwitch.ts` → `FormatSwitchPage.tsx`

**Phase 4 — Parallel: `class-delivery-library` / `messaging` / `gamification-engagement`**
21a. `useRecordingConsent.ts`, `useSessions.ts`, `useRecordings.ts`, `useLibrary.ts`, `useReschedule.ts`, `useWeeklyAssessment.ts` → their pages/components
21b. `useMessaging.ts`, `useAdminMessaging.ts` → `MessagingPage.tsx`, `MessageThreadReviewPage.tsx`
21c. `useGamification.ts`, `useChallenges.ts`, `useAdminGamification.ts` → their pages/components

**Phase 5 — `payments-earnings` and `support-trust-admin`**
22. `usePayments.ts`, `useEarnings.ts`, `usePricing.ts`, `useRefunds.ts`, `usePayouts.ts`, `usePromotions.ts` → their pages/components
23. `useComplaints.ts`, `useAdminDisputes.ts`, `useAdminReporting.ts` → `SubmitComplaintPage.tsx`, `SupportContactPage.tsx`, `AdminSidebar.tsx`, `DisputeQueuePage.tsx`, `PlatformReportsPage.tsx`
24. `src/routes/index.tsx` (final pass — every page registered)
25. Remaining hook test files, per the standing test-mirrors-hook rule

---

**Next:** proceed to → [06. API Specification]
