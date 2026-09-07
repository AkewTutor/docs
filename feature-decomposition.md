# AKEWTutor — Feature Decomposition

**Project:** AKEWTutor — Online Tutoring Platform

**Links back to:** [04. Database & Data Model]
**Links forward to:** [05. Folder & File Structure], [08. Function-Level Specification]

**Purpose:** Docs 06 and above (API, frontend spec, function-level spec, test spec) are too large to write or build as single monolithic files. This document defines the feature boundaries those later docs will be split along, so that each feature can be specified, built, and tested independently by a different person/pair with minimal cross-feature coordination.

**Method:** Features are drawn directly from the entity ownership and foreign-key relationships in Doc 04 — not from a conceptual/thematic grouping. A dependency is only called **hard** if a real foreign key crosses feature boundaries; everything else is **soft** (a service call, a triggered side effect, an admin action that touches another feature's data) and does not constrain build order.

---

## 1. Backend Features

| # | Feature | Owns (entities) | Hard dependency (schema FK) | Soft dependency (workflow/integration only) |
|---|---|---|---|---|
| 1 | **shared-config** | User, Notification, PolicyDocument, RefreshToken | — | — |
| 2 | **accounts-guardianship** | StudentProfile, ParentProfile, TutorProfile, ParentStudentRelationship, Subject, TutorSubjectRanking, AvailabilitySlot | shared-config | — |
| 3 | **matching-cohorts** | MatchRequest, TutorExclusion, Cohort, CohortMembership, FormatSwitchRequest | accounts-guardianship | — |
| 4 | **class-delivery-library** | ScheduledSession, RescheduleRequest, SessionMiss, RecordingConsent, Recording, LibraryMaterial, WeeklyAssessment | matching-cohorts | — |
| 5 | **messaging** | MessageThread, Message | matching-cohorts | — |
| 6 | **gamification-engagement** | XPLedgerEntry, Badge, StudentBadge, TutorBadge, Streak, Challenge, ChallengeProgress | accounts-guardianship | class-delivery-library (XP-award trigger on class-attended; no FK) |
| 7 | **payments-earnings** | PricingConfig, Payment, PaymentPause, Refund, TutorEarning, Payout, PromotionCode | matching-cohorts, class-delivery-library (TutorEarning → ScheduledSession) | — |
| 8 | **support-trust-admin** | ComplaintReport | shared-config, messaging, class-delivery-library | payments-earnings (refund action from a dispute), accounts-guardianship (suspension action from a dispute) |

### 1.1 Feature Notes

- **shared-config** is the only feature with no dependency of any kind — it's the foundation every other feature's auth and notification calls sit on.
- **accounts-guardianship** bundles the Subject catalog and tutor availability alongside identity/guardianship, since matching can't function without a tutor's ranked subjects and open slots existing first — these are read-heavy, rarely-changing reference data owned by the same team that owns tutor profiles.
- **matching-cohorts** is the pivot feature: everything downstream (class delivery, messaging, payments) keys off a `Cohort`, so this feature must be functionally complete before 4, 5, or 6 can be meaningfully tested end-to-end (though they can still be *coded* against a stubbed Cohort).
- **payments-earnings** owns `PricingConfig` even though pricing is introduced early in the SRS (Section 07, before matching) — pricing is administratively a money concern (grouped with payouts/refunds in SRS Section 11.3), and matching only *reads* the active price, it doesn't own or mutate it. **Fix (audit):** because `payments-earnings` is built last (step 7) while `matching-cohorts` (step 3) already needs to read an active price the moment a `Cohort` reaches `PENDING_PAYMENT`, `prisma/seed.ts` (Doc 05a §0) must seed a default `PricingConfig` row per `TutoringFormat` alongside the Admin user and Subject catalog — otherwise `matching-cohorts` has nothing to read on a fresh clone or in tests, regardless of build/parallelization order.
- **support-trust-admin** is the one feature that is soft-coupled to almost everything, by design — disputes can be about a message thread, a session, a payment, or a tutor's standing. This mirrors why it's built last: it's an integration surface, not a domain of its own data.

---

## 2. Parallelizable Build Order

```
1. shared-config
2. accounts-guardianship
        ├─→ 3. matching-cohorts
        │        ├─→ 4. class-delivery-library ─┐
        │        └─→ 5. messaging               │
        └─→ 6. gamification-engagement          │
                                                 ├─→ 7. payments-earnings
                                        4 + 5 ───┴─→ 8. support-trust-admin
```

**How to read this:**
1. **shared-config** must be built first — nothing else compiles or authenticates without it.
2. **accounts-guardianship** must follow immediately — it's the hard dependency for both `matching-cohorts` and `gamification-engagement`.
3. Once **matching-cohorts** lands, **class-delivery-library**, **messaging**, and **gamification-engagement** have no remaining hard dependency on each other — three different people can build all three at the same time.
4. **payments-earnings** and **support-trust-admin** are the two integration points that necessarily come last, since each has a real FK or direct data dependency reaching into features built in step 3.

---

## 3. Frontend Features

Frontend mirrors the same 8 feature names 1:1, so a backend feature and its corresponding frontend feature can be built by the same pair without needing to track two different naming schemes.

**The one difference:** frontend adds a 9th, zero-numbered prerequisite — **`ui-foundation`** (design tokens, common/shared components, layouts, route guards) — built before any of the 8 numbered features. This has no backend equivalent, since the backend has no analogous "foundation" layer that every feature's UI needs to inherit from before it can render anything.

```
0. ui-foundation   (frontend only — no backend equivalent)
1. shared-config
2. accounts-guardianship
   ... (same order as Section 2) ...
```

---

**Next:** proceed to → [05. Folder & File Structure]
