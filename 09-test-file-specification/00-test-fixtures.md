## Project: AKEWTutor — Test Fixture & Factory Convention

**Fixes:** Review §3.7 (no test data/fixture strategy specified anywhere). **Referenced by:** every `09-N-*.md` file (backend and frontend) and `00-agent-rules.md`.

**Rule:** every unit, Integration (HTTP contract), Integration (persistence), and E2E test builds its entity data through the factory functions defined here — never through an inline ad-hoc object literal. If a test needs an entity shape not yet covered below, the factory is added here first, in the same PR, not inlined "just this once."

---

### 0. Location & shape

Factories live under `tests/factories/`, one file per owning module (mirroring the 8 backend `09-N` modules — a module's factories cover the entities it owns per that module's `09-N` header, regardless of whether the consuming test is backend or frontend):

| Factory file | Entities |
|---|---|
| `tests/factories/shared-config.factory.ts` | User, RefreshToken, Notification, PolicyDocument |
| `tests/factories/accounts-guardianship.factory.ts` | StudentProfile, ParentProfile, TutorProfile, ParentStudentRelationship, Subject, TutorSubjectRanking, AvailabilitySlot |
| `tests/factories/matching-cohorts.factory.ts` | MatchRequest, TutorExclusion, Cohort, CohortMembership, FormatSwitchRequest |
| `tests/factories/class-delivery-library.factory.ts` | ScheduledSession, RescheduleRequest, SessionMiss, RecordingConsent, Recording, LibraryMaterial, WeeklyAssessment |
| `tests/factories/messaging.factory.ts` | MessageThread, Message |
| `tests/factories/payments-earnings.factory.ts` | PricingConfig, Payment, PaymentPause, Refund, TutorEarning, Payout, PromotionCode |
| `tests/factories/gamification-engagement.factory.ts` | XPLedgerEntry, Badge, StudentBadge, TutorBadge, Streak, Challenge, ChallengeProgress |
| `tests/factories/support-trust-admin.factory.ts` | ComplaintReport |

Every factory has the shape `buildX(overrides?: Partial<X>): X` — takes a sparse partial, returns a fully-populated, schema-valid object, `overrides` shallow-merged last so any field can be pinned for a given test case.

Frontend tests import the same factories (via a shared package/path alias, not a duplicate frontend copy) so that a fixture shape can never drift between what a backend integration test and a frontend component test believe an entity looks like.

---

### 1. Cross-cutting conventions (apply to every factory below)

1. **IDs:** every `id` and FK field defaults to a fresh `crypto.randomUUID()` unless overridden. A default FK value is a syntactically valid UUID with **no guarantee a row with that ID exists** — Unit and Integration (HTTP contract) tests (service layer / DB mocked) can rely on defaults freely; Integration (persistence) and E2E tests **must** override every FK with the real ID of a row actually created in that test's setup, or the test will fail against real foreign-key constraints (this is intentional — a persistence test that "passes" against a dangling FK default is exactly the false-positive Review §7 rule 5 warns about for mocks, and the same logic applies here).
2. **Money fields (`Decimal` columns — e.g., `Payment.amount`, `PricingConfig.pricePerStudentPerHour`, `Refund.amount`, `TutorEarning.amount`, `Payout.totalAmount`, `PromotionCode.discountValue`):** factories return these as a decimal-safe string (e.g., `"120.00"`), matching `frontend/9-7-payments-earnings.md`'s Decimal-as-string convention — never a raw JS `number`. Backend tests that need arithmetic use the project's decimal library on the factory's output, not `parseFloat`/`Number()` coercion, for the same precision-loss reason `9-7`'s existing convention note gives.
3. **Timestamps:** `createdAt`/`updatedAt`-style fields default to `new Date()` at call time. Fields with a documented offset (e.g., `RefreshToken.expiresAt` = `createdAt` + 30 days, `Recording.expiresAt` = `createdAt` + 90 days) default using that same offset, not an arbitrary future date, so a test overriding only `createdAt` still gets an internally consistent object.
4. **Enums:** default to the schema's own stated default where one exists (e.g., `Cohort.status` → `FORMING`, `Payment.status` → `PENDING`); where the schema has no stated default, default to the "happy path" first-listed value (e.g., `SessionMiss.missType` → `NO_SHOW`). Tests exercising a non-default state pass it explicitly via `overrides` — this keeps every factory call's non-default state visibly intentional in the test body rather than buried in the factory.
5. **Nullable fields:** default to `null`/`undefined` (whichever the surrounding type layer expects) unless the entity is meaningless without it for most test purposes (e.g., `WeeklyAssessment.tutorFeedback` is required, not nullable, so it always gets a placeholder string).
6. **Composability:** factories do not call each other automatically (e.g., `buildCohortMembership()` does not internally call `buildCohort()`) — a test composes the graph it needs explicitly, passing real IDs down, so the test body itself documents which entities are actually related in that scenario.

---

### 2. Factory reference by module

Each row gives the factory's required-override fields (i.e., fields with no sensible test-agnostic default — the test must supply them) and any default worth calling out. Every other field on the entity gets a schema-valid default per §1 and is not re-listed here field-by-field; see `04-database-and-data-model.md §4.2` for the full field list of each entity.

#### shared-config

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildUser(overrides)` | none (`role` defaults `STUDENT`) | `passwordHash` a fixed bcrypt-shaped dummy hash, never a real hash; `termsAcceptedAt` defaults to now |
| `buildRefreshToken(overrides)` | `userId` | `familyId` defaults to a fresh UUID (new chain); `revokedAt`/`replacedByTokenId` default `null` (active token) |
| `buildNotification(overrides)` | `userId`, `type` (no sensible cross-cutting default given the L2 canonical-type table) | `status` defaults `QUEUED`, `channel` defaults `PUSH` |
| `buildPolicyDocument(overrides)` | `type`, `publishedById` | `version` defaults `1`, `content` a short placeholder Markdown string |

#### accounts-guardianship

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildStudentProfile(overrides)` | `userId` | `grade` defaults `8`; `accountStatus` derived from `grade` per FR-AC-002/005 (defaults `ACTIVE`, since default `grade` is 6–12) — pass `grade: <1-5>` to get the `PENDING_ACTIVATION` path |
| `buildParentProfile(overrides)` | `userId` | `onboardingStatus` defaults `PENDING` |
| `buildTutorProfile(overrides)` | `userId` | `verificationStatus` defaults `PENDING`; `experienceDescription` a placeholder string |
| `buildParentStudentRelationship(overrides)` | `parentId`, `studentId` | `relationshipType` defaults `MANDATORY_GUARDIAN`; `status` defaults `INVITED`; `inviteExpiresAt` defaults `invitedAt` + 14 days |
| `buildSubject(overrides)` | none | `name` defaults a unique placeholder (`Subject ${randomUUID()}`) to satisfy the unique constraint across repeated calls |
| `buildTutorSubjectRanking(overrides)` | `tutorId`, `subjectId` | `rank` defaults `1` |
| `buildAvailabilitySlot(overrides)` | `tutorId` | `isRecurring` defaults `false`; `startTime`/`endTime` default a 1-hour window starting now |

#### matching-cohorts

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildMatchRequest(overrides)` | `studentId`, `subjectId` | `format` defaults `ONE_TO_ONE`; `path` defaults `PATH_A`; `status` defaults `SEARCHING` |
| `buildTutorExclusion(overrides)` | `studentId`, `tutorId` | `reason` defaults `ADMIN_REJECTED` |
| `buildCohort(overrides)` | `tutorId`, `subjectId` | `format` defaults `ONE_TO_ONE`; `status` defaults `PENDING_ADMIN_APPROVAL` (the 1-to-1 default per schema) — pass `format: 'ONE_TO_THREE'` / `'ONE_TO_FIVE'` and `status: 'FORMING'` explicitly for the group path, since the schema's default is format-conditional and the factory cannot infer intent silently |
| `buildCohortMembership(overrides)` | `cohortId`, `studentId` | `status` defaults `PENDING_PAYMENT` |
| `buildFormatSwitchRequest(overrides)` | `studentId`, `fromMembershipId`, `fromFormat`, `toFormat` | `requestedAt` defaults now |

#### class-delivery-library

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildScheduledSession(overrides)` | `cohortId` | `status` defaults `SCHEDULED`; `recordingStatus` defaults `PENDING`; `scheduledStart`/`scheduledEnd` default a 1-hour window starting 24h from now (avoids accidental past-dated defaults tripping "must be ≥30 min before start" style assertions) |
| `buildRescheduleRequest(overrides)` | `sessionId`, `requestedById`, `requestedNewStart` | `noticeHours` defaults computed from `requestedNewStart` vs. now if not passed; `classification` defaults `FREE_RESCHEDULE` when the computed/passed `noticeHours` ≥ 12, else `SAME_DAY_MISS` |
| `buildSessionMiss(overrides)` | `sessionId` | `causedBy` defaults `TUTOR`; `missType` defaults `NO_SHOW` |
| `buildRecordingConsent(overrides)` | `tutorId`, `studentId` | both acknowledgment timestamps default `null` (incomplete) — pass both explicitly to get a consent-complete fixture |
| `buildRecording(overrides)` | `sessionId` | `encoding` defaults `"720p"`; `expiresAt` defaults `createdAt` + 90 days; `keepPermanently` defaults `false` |
| `buildLibraryMaterial(overrides)` | `cohortId`, `uploadedByTutorId` | `fileType` defaults `PDF` |
| `buildWeeklyAssessment(overrides)` | `cohortMembershipId`, `submittedByTutorId` | `tutorFeedback` a placeholder string (required, not nullable) |

#### messaging

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildMessageThread(overrides)` | `cohortId` | `status` defaults `ACTIVE` |
| `buildMessage(overrides)` | `threadId`, `senderId` | `body` a placeholder string |

#### payments-earnings

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildPricingConfig(overrides)` | `createdById` | `format` defaults `ONE_TO_ONE`; `isActive` defaults `false` (pass `true` explicitly — mirrors the "only one active per format" invariant being a deliberate act, not a default) |
| `buildPayment(overrides)` | `cohortMembershipId` | `provider` defaults `CHAPA`; `status` defaults `PENDING`; `billingPeriodStart`/`End` default a 30-day window starting now |
| `buildPaymentPause(overrides)` | `cohortMembershipId` | `reason` defaults `NONPAYMENT`; `endedAt` defaults `null` (open pause) |
| `buildRefund(overrides)` | `paymentId`, `reason`, `sessionsRemaining`, `totalSessionsBilled` | `status` defaults `PENDING` per the I1 fix; `amount` **is not auto-derived** by the factory (the proration formula is production logic under test, not fixture logic) — callers pass the expected `amount` explicitly |
| `buildTutorEarning(overrides)` | `tutorId`, `sessionId` | `rateType` defaults `FULL` |
| `buildPayout(overrides)` | `tutorId` | `status` defaults `PENDING`; `totalAmount` defaults `"0.00"` |
| `buildPromotionCode(overrides)` | `createdById` | `code` defaults a unique placeholder; `discountType` defaults `PERCENT`; `isActive` defaults `true` |

#### gamification-engagement

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildXPLedgerEntry(overrides)` | `studentId`, `amount` | `reason` defaults `CLASS_ATTENDED` |
| `buildBadge(overrides)` | none | `category` defaults `STUDENT`; `isActive` defaults `true` |
| `buildStudentBadge(overrides)` | `studentId`, `badgeId` | `earnedAt` defaults now |
| `buildTutorBadge(overrides)` | `tutorId`, `badgeId` | `earnedAt` defaults now |
| `buildStreak(overrides)` | `studentId` | `currentStreakDays`/`longestStreakDays` default `0` |
| `buildChallenge(overrides)` | `createdById`, `targetValue` | `period` defaults `WEEKLY`; `startsAt`/`endsAt` default the current calendar week |
| `buildChallengeProgress(overrides)` | `studentId`, `challengeId` | `progressValue` defaults `0` |

#### support-trust-admin

| Factory | Required overrides | Notable defaults |
|---|---|---|
| `buildComplaintReport(overrides)` | `reporterId`, `category`, `description` | `status` defaults `OPEN`; for `category: 'TUTOR_CONDUCT'`, the caller must also pass `relatedCohortId` and/or `relatedSessionId` per the tutor-resolution rule in `04-database-and-data-model.md §4.2.9` — the factory does not silently populate one to paper over an omitted override |

---

### 3. What this does *not* replace

Fixture factories build **entity shapes**, not test doubles for services/clients (Prisma client mocks, the Chapa client mock, `vi.mock('@/lib/axios')`, etc.) — those remain defined per-test/per-suite as today. A factory-built object is what you hand *to* a mock (e.g., `prisma.cohort.findUnique.mockResolvedValue(buildCohort({ id: cohortId }))`), not a replacement for the mock itself.
