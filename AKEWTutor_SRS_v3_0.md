# AKEWTutor — Software Requirements Specification

**Online Tutoring Platform — Students, Tutors & Administration**

**Version 3.0 — Final Requirements Baseline**, incorporating the resolution of every item raised across five consistency-audit rounds following the v2.0 client review.

*September 2026 · Confidential — For Project Use Only*

---

## Document Control

| Field | Value |
|---|---|
| Document Title | AKEWTutor — Software Requirements Specification (SRS) |
| Project | AKEWTutor Online Tutoring Platform |
| Version | 3.0 — Final |
| Status | Final requirements baseline — approved for Technical Specification drafting. Section 18 fully closed. |
| Date | September 2026 |
| Prepared For | AKEWTutor Client / Project Owner |
| Source Material | v2.0 client-reviewed SRS, plus the AKEWTutor SRS Resolution Log (5 audit rounds, 26 items) |
| Classification | Confidential — for project use only |

### Revision History

| Version | Date | Description | Author |
|---|---|---|---|
| 1.0 | Aug 2026 | Initial structured SRS derived from client project notes. | Business Analyst |
| 2.0 | Aug 2026 | Incorporated client decisions on all v1.0 clarification items; added Parent/Student account model, refund policy, recording storage architecture; applied two feature changes (ratings/reviews removed; format-based matching split). Two items remained open. | Business Analyst |
| **3.0** | **Sep 2026** | Closed both remaining Section 18 items (video conferencing, third-party providers). Resolved 11 numbered gaps, 3 minor items, an 8-item SRS-level audit, and a 5-item final-check audit across five consistency-audit rounds. Added Messaging (FR-MS), Make-up/Reschedule/Cancellation (FR-MK), and two Safety requirements (FR-SC-008/009). Amended account-model, subject-ranking, payout, leaderboard, and notification requirements. Added per-section Definition of Done blocks to Sections 5–14. Fixed one requirement-ID collision and one cross-section contradiction. No items remain open. | Business Analyst |

### Requirement ID Key

Every functional requirement carries a unique ID in the form **FR-[AREA]-[NUMBER]** (e.g. FR-SP-004). Non-functional requirements use the prefix **NFR-[NUMBER]**. Two functional areas are new in this version: **FR-MS** (in-platform Messaging) and **FR-MK** (Make-up, Reschedule & Cancellation). Retired IDs (e.g. FR-SP-041, FR-TU-020, FR-AD-020) are never reused and remain listed for historical traceability only.

---

## Contents

> Sections 5–14 contain numbered functional requirements (FR-xxx), individually traceable, each closing with a Definition of Done block. Section 15 adds baseline non-functional requirements (NFR-xxx). Boxes marked **RESOLVED** or **NEW IN v3.0** flag every decision closed since v2.0; Section 18 consolidates the full resolution record for client sign-off.

01. Introduction
02. Platform Overview & Purpose
03. Public Website Structure
04. User Roles & Account Model
05. Student & Parent Experience — Functional Requirements
06. Tutor Experience — Functional Requirements
07. Tutoring Formats & Pricing Model
08. Tutor–Student Matching Workflow
09. Class Delivery, Recording & Make-up Management
10. Learning Engagement & Gamification
11. Administrator Dashboard — Functional Requirements
12. Notification Framework
13. Payment & Billing Policy
14. Safety, Trust & Compliance
15. Non-Functional Requirements
16. Third-Party Dependencies & Integrations
17. End-to-End User Journey
18. Resolution Record — v2.0 to v3.0
19. Approval & Sign-off

---

## 01 Introduction

### 1.1 Purpose & Audience

This SRS translates the client's project brief for AKEWTutor into a structured, traceable set of requirements. Version 3.0 is the final requirements baseline: it carries forward every decision made in v2.0 and additionally resolves the two items that remained open in Section 18, together with every gap and inconsistency surfaced across five rounds of consistency auditing (see Section 18 for the full record). It is intended for the client (approval), the design & development team (build reference), and QA (test-case basis).

### 1.2 Scope

Confirmed by client — unchanged from v2.0.

**In scope:**
- Public marketing website (homepage and standalone informational pages)
- Student, Parent, Tutor, and Administrator web-based platforms
- Registration, profile building, tutor discovery, and tutor–student matching
- In-platform text messaging between a student/parent and their matched tutor(s)
- Booking, payment initiation, and schedule confirmation
- Class reminders, make-up/reschedule/cancellation handling, and hand-off to an external video link
- Recording storage and access control within an educational Library
- Gamification layer — XP, badges, streaks, per-grade leaderboards, and challenges
- Administrator console covering all of the above

**Out of scope for this phase:**
- Native mobile applications (iOS / Android)
- Built-in video conferencing — sessions rely on an externally generated Jitsi link (Section 9, Section 18)
- Multi-currency or international payment support beyond ETB
- Support for learners outside Grade 1–12 (possible future expansion)
- Technical integration with school information systems
- Voice, video, or file-attachment messaging (in-platform messaging is text-only, Section 5.11)

### 1.3 Assumptions & Constraints

Confirmed by client — unchanged from v2.0.

- The platform operates primarily in Ethiopia, with pricing and payments in ETB
- Tutoring sessions are conducted via an externally generated video link (Jitsi), not a native video system
- Every booking passes through Admin review — there is no fully automated instant-booking path
- Emergency support contact is manual, via phone number or Telegram
- A tutor is normally linked to one subject; a second subject is a system-triggered exception under a declared shortage (Section 6.2), and no tutor ranks more than two subjects (Section 6.1)
- Pricing and revenue-split figures (Section 7) are final and Admin-configurable post-launch

### 1.4 Related Documents

A separate Technical/System Specification (architecture, database schema, API design, hosting) and UI/UX design files follow from this document and are not covered here. Per-requirement QA test cases remain a separate QA deliverable; this SRS provides per-section Definition of Done blocks (Sections 5–14) as the client sign-off layer, not exhaustive test cases.

---

## 02 Platform Overview & Purpose

AKEWTutor is an online tutoring platform connecting students in Grades 1–12 with qualified tutors. The platform serves three user groups — Students & Parents, Tutors, and the Administrator — connected through a matching engine, a scheduled class system, an in-platform messaging channel, and a shared educational library.

> **Core Promise:** Register → Select learning needs & format → Get matched with a tutor → Admin approval → Payment → Scheduled class → Reminders → Live session → Recording & materials → Weekly assessment → Progress tracking → Continued learning.

---

## 03 Public Website Structure

**Homepage Sections:** Platform introduction · "Find a Tutor" / "Become a Tutor" CTAs · How it works · Benefits · Featured tutors · Popular subjects & grades · Success stories · Promotional offers · FAQ · Contact/support · Login and Registration

**Standalone Public Pages:** About · Find a Tutor · Subjects · Grades 1–12 · Exam Preparation · Become a Tutor · For Parents · For Schools · Pricing · Resources/Blog · FAQ & Contact · Terms & Conditions, Privacy Policy, Safety Policy, Refund Policy

> ✅ **RESOLVED — v2.0 — Refund Policy**
> Refunds apply only when a session/schedule disruption is not caused by the student — namely (a) a tutor drops out of the platform and cannot be replaced, or (b) a platform-side outage prevents a paid session from occurring. Refunds are prorated by sessions delivered (see the v3.0 refinement in Section 13). All refunds require Admin review and approval (FR-AD-012, FR-PB-007).

---

## 04 User Roles & Account Model

### 4.1 Roles at a Glance

| Role | Description |
|---|---|
| Student | Builds an academic profile, gets matched to a tutor, attends classes, messages their tutor, tracks progress, engages with gamified learning. |
| Parent / Guardian | Registers as an independent account linked to one or more students; manages payments and oversees the learning journey. |
| Tutor | Delivers subject teaching, manages availability, uploads materials and recordings, messages assigned students, earns per session. |
| Administrator | Verifies tutors, approves bookings, manages pricing and payments, resolves disputes, monitors the platform. |

### 4.2 Parent–Student Account Model

> ✅ **RESOLVED — v2.0 — Parent/Student Account Model**
> Parents and students hold independent accounts (separate credentials) linked by a parent–student relationship record (parent_id, student_id, relationship_type, status, permissions). One guardian may manage multiple students; a student may have multiple guardians. The relationship can be revoked or modified over time.

**Grades 1–5 — parent-first, guardian mandatory:**
- Parent registers first; the parent account begins in a pending onboarding state.
- Parent must add at least one student, entering the student's grade at that moment (this is what routes the flow into the 1–5 vs. 6–12 path) and creating a student record with status invited; the platform sends the student an activation invite, valid for 14 days, with an automatic reminder at day 7. The parent may resend/regenerate the invite at any time, which restarts the 14-day window.
- Until the student accepts, the parent may log in only to manage or resend the invitation — no student-related access yet.
- Once the student completes account creation: the student account becomes active, the relationship becomes active, and the parent's onboarding completes with full parent functionality unlocked.

**Grades 6–12 — student-first, guardian optional:**
- Student registers and uses the platform independently, setting their own grade via FR-SP-007.
- Student may invite a parent/guardian at any time; the relationship follows the same invite → active flow.
- Students in this band may pay independently via Chapa with no guardian required (Section 13).

> ✨ **NEW IN v3.0 — Sole-Guardian Removal (Grades 1–5)**
> If the sole guardian on a Grades 1–5 account is removed — self-initiated, or Admin-initiated for abuse/dispute reasons — the student account moves to a **"guardian required" hold state**: access is paused (no new bookings, no class access) but no data, progress, XP, or recordings are deleted. The platform prompts for a new guardian invite; once a new guardian accepts, the account reactivates with all history intact. This mirrors the existing invited → active relationship flow rather than introducing a new mechanism.

| ID | Requirement |
|---|---|
| FR-AC-001 | Support independent Parent and Student account types, linked via a parent–student relationship record (relationship_type, status, permissions). |
| **FR-AC-002** *(amended)* | For students in Grades 1–5, require parent-first registration: parent account starts pending; the parent must add a student, entering the student's grade as a required field at that moment, creating an invited student record. This grade entry is what determines routing into the 1–5 vs. 6–12 flow. |
| **FR-AC-003** *(amended)* | Send the invited student an activation invite valid for 14 days, with an automatic reminder at the 7-day mark; restrict the parent to invite-management access only (including resend/regenerate, which restarts the 14-day window) until the student activates. |
| FR-AC-004 | On student activation, set student account and relationship to active, and unlock full parent functionality. |
| FR-AC-005 | For students in Grades 6–12, allow independent student registration with an optional parent/guardian invite at any time; the student sets their own grade via FR-SP-007 with no parent-entry step. |
| FR-AC-006 | Support multiple guardians per student and multiple students per guardian, with per-relationship permissions. |
| FR-AC-007 | Allow a relationship's access to be revoked or modified by an authorized party — the linked guardian(s), or Admin for disputes/abuse. A student cannot unilaterally revoke a mandatory guardian's access under the Grades 1–5 model; for Grades 6–12 relationships the student initiated, the student may also revoke it themselves. |
| **FR-AC-008** *(new)* | If the sole guardian on a Grades 1–5 account is removed, move the student account to a "guardian required" hold state (access paused, no data deleted) until a new guardian accepts an invite. |

---

## 05 Student & Parent Experience — Functional Requirements

### 5.1 Registration & Onboarding

| ID | Requirement |
|---|---|
| FR-SP-001 | Allow student registration via email or phone number, per the account model in Section 4.2. |
| FR-SP-002 | Allow parent/guardian registration via email or phone number, per the account model in Section 4.2. |
| FR-SP-003 | Provide login, logout, and password reset functionality. |
| FR-SP-004 | Verify user email address or phone number during registration (via the SMS/email providers in Section 16). |
| FR-SP-005 | Require acceptance of the Privacy Policy, Terms & Conditions, and Rules & Regulations before registration can be completed. |
| FR-SP-006 | Allow creation and editing of a user profile, including a profile picture. |

### 5.2 Student Academic Profile

| ID | Requirement |
|---|---|
| FR-SP-007 | Allow the student to select their Grade (1–12) and school. |
| FR-SP-008 | Allow the student to select subjects, academic level, and learning goals. |
| FR-SP-009 | Allow the student to set a preferred language, learning schedule, and teaching style. |
| FR-SP-010 | Allow the student to indicate a budget preference and a tutoring-format preference (1-to-1, 1-to-3, or 1-to-5) — this choice determines the matching flow in Section 8. |

> ✨ **NEW IN v3.0 — How Preferences Are Applied in Matching**
> Budget is a **hard filter** for 1-to-1 search (Section 5.4) — tutors priced above the stated budget are excluded from results. Language is a hard filter across all formats. Teaching style is a **soft, scored** factor that feeds the 1-to-1 match percentage (FR-MA-001) but is not used to filter 1-to-3/1-to-5 auto-matching, which optimizes primarily for grade/subject/schedule fit across the group.

### 5.3 Student Dashboard

| ID | Requirement |
|---|---|
| FR-SP-011 | Display a welcome message, upcoming lessons, and today's learning activities. |
| FR-SP-012 | Display recommended lessons (see Section 8 for tutor recommendation, which applies to 1-to-1 only). |
| FR-SP-013 | Display assignments, quizzes, and recent grades. |
| FR-SP-014 | Display learning progress, XP, streaks, achievements, and current league / leaderboard position. |
| FR-SP-015 | Display daily and weekly goals and active learning challenges. |
| FR-SP-016 | Provide access to notifications, messages, and rewards from the dashboard. |

### 5.4 Find a Tutor (Search) — 1-to-1 Only

> ✅ **RESOLVED — v2.0 — Search Filters & Format Split**
> Search-and-choose is available for the 1-to-1 format only. Confirmed filters: subject, grade, schedule/availability, budget (hard filter), and language (hard filter), plus additional reasonable filters (e.g. price) to be added at design time. For 1-to-3 / 1-to-5, students set a preference (FR-SP-010) and the system auto-matches — see Section 8.

| ID | Requirement |
|---|---|
| FR-SP-017 | (1-to-1 only) Allow students/parents to search for tutors by subject. |
| FR-SP-018 | (1-to-1 only) Allow students/parents to search for tutors by grade. |
| FR-SP-019 | (1-to-1 only) Allow students/parents to filter tutor search results by schedule/availability, budget, language, and by additional reasonable filters defined at design time. |

### 5.5 Tutor Recommendation — 1-to-1 Only

| ID | Requirement |
|---|---|
| FR-SP-020 | (1-to-1 only) Recommend tutors based on the student's grade, subject, academic level, schedule, teaching-style fit, and tutor availability. |
| FR-SP-021 | (1-to-1 only) Display a match percentage (e.g. 95%, 80%) for each recommended tutor. Not shown for 1-to-3/1-to-5 (system auto-matched, no student-facing match info). |

### 5.6 Tutor Profile (Student/Parent View)

> ℹ️ **Profile Visibility Differs by Format**
> The full tutor-profile view below (photo, education, subjects, total-students count, available slots) applies to **1-to-1** only. For 1-to-3 / 1-to-5, students see only the assigned tutor's **name and photo** once a class is confirmed — no education detail, no total-students count, and no time-slot picker, consistent with the no-match-information rule in Section 8, Path C.

| ID | Requirement |
|---|---|
| FR-SP-022 | Display tutor name, profile photo, and verification badge. |
| FR-SP-023 | Display tutor education, university, and degree (optional field) — 1-to-1 profile view only. |
| FR-SP-024 | Display subjects and grades taught by the tutor. |
| **FR-SP-025** *(amended)* | Display the total number of **unique** students the tutor has taught to date (distinct student count, not a running session tally) — 1-to-1 profile view only. |
| FR-SP-026 | Display tutor available time slots (1-to-1 search view only). |

### 5.7 Booking, Matching & Payment

See Section 8 for the full matching workflow, split by tutoring format.

| ID | Requirement |
|---|---|
| FR-SP-027 | (1-to-1) Allow the student/parent to view match percentage and full tutor profile before choosing. |
| FR-SP-028 | (1-to-1) Allow the student/parent to choose a preferred tutor when a suitable match is available. |
| FR-SP-029 | (1-to-1) Allow the student/parent to manually select "No Exact Match" to request Admin assignment when no suitable tutor is found. |
| **FR-SP-030** *(amended)* | Display the assigned tutor and confirmed tutoring schedule once set, for all formats — subject to the profile-visibility split in Section 5.6 (full profile for 1-to-1; name and photo only for 1-to-3/1-to-5). |
| FR-SP-031 | Require payment completion after tutor matching/assignment and Admin approval, before the schedule is confirmed, for all formats. |
| FR-SP-032 | Allow the student/parent to view payment history at any time. |

### 5.8 Classes, Recordings & Library

> ✅ **RESOLVED — v2.0 — Recording Storage & Retention**
> Recordings are stored in a dedicated cloud object-storage service (Section 16; outside PostgreSQL) — PostgreSQL/Prisma stores only metadata (student, tutor, session, storage key, file size, created date, expiration date). Default retention is 90 days from upload, then auto-deletion — unless the student selects "Keep permanently," in which case the recording is retained until deleted by the student or platform. Target encoding is 720p, compressed. Recordings are served via authenticated, temporary/signed URLs, never permanent public links. Telegram was evaluated and rejected as a storage backend.

| ID | Requirement |
|---|---|
| FR-SP-033 | Send a countdown notification 1 hour before each scheduled class. |
| FR-SP-034 | Deliver the tutor's session link (Jitsi, Section 9) at least 30 minutes before class. |
| FR-SP-035 | Allow access to scheduled sessions and to personal tutoring recordings after class, via signed temporary URLs. |
| FR-SP-036 | Restrict each student's access to only their own class recordings — never another student's. |
| FR-SP-037 | Provide an educational Library containing PDFs, books, notes, tutor-provided materials, and personal recordings, with a default 90-day retention and a "Keep permanently" option per recording. |

### 5.9 Progress, Feedback & Gamification

| ID | Requirement |
|---|---|
| FR-SP-038 | Provide weekly assessment results and progress view, including tutor feedback. |
| FR-SP-039 | Award achievements, badges, XP / points, and display leaderboard position. |
| FR-SP-040 | Present weekly and monthly learning challenges. |

*Retired: FR-SP-041 (student rating/review of tutors) — removed platform-wide, Feature Change FC-01. This ID is not reused.*

### 5.10 Support

| ID | Requirement |
|---|---|
| FR-SP-042 | Provide a complaint / claim / report system for students and parents. |
| FR-SP-043 | Provide a customer support contact channel. |
| FR-SP-044 | Provide account and privacy settings management. |

### 5.11 In-Platform Messaging (New in v3.0)

> ✨ **NEW IN v3.0 — Direct Messaging with a Matched Tutor**
> AKEWTutor previously offered no channel for a student/parent and their tutor to exchange a quick note (e.g. "running 5 minutes late") outside a live class, short of the full complaint system. v3.0 adds a lightweight, text-only, in-platform messaging thread, scoped strictly to matched pairs, closing this gap while preserving the no-outside-platform-contact rule (FR-SC-004).

| ID | Requirement |
|---|---|
| **FR-MS-001** | Allow a student/parent and their currently assigned tutor(s) to exchange text-only messages in-platform, once a match is confirmed and paid (Section 8). |
| **FR-MS-002** | Restrict messaging threads to matched pairs only — no messaging before a confirmed match, and no messaging with tutors/students outside an active assignment. |
| **FR-MS-003** | Retain message history for the duration of the active assignment plus 90 days, then archive it out of the active thread view. |
| **FR-MS-004** | Allow Admin to view message threads for dispute investigation and to close/report a thread that violates platform rules (ties FR-SC-003, FR-AD-017). |

**Definition of Done — Section 05 — Student & Parent Experience**
1. A Grade 6–12 student can register, build a full academic profile, and pay independently with no guardian on the account.
2. A Grade 1–5 parent cannot access any student-specific screen until that student activates their invite; the invite expires and is resendable per FR-AC-003.
3. 1-to-1 search results never include a tutor priced above the student's stated budget or teaching a different language preference.
4. The 1-to-3/1-to-5 assigned-tutor view shows only name and photo — no education, no total-students count, no match percentage.
5. The total-students count shown on a 1-to-1 tutor profile reflects distinct students, verified against a tutor who has taught one student across multiple repeated sessions.
6. A student can message their assigned tutor once payment/confirmation is complete, and cannot message any tutor they are not currently matched with.
7. Recordings older than 90 days are inaccessible unless marked "Keep permanently," and a student can never open another student's recording via a guessed or shared URL.

---

## 06 Tutor Experience — Functional Requirements

### 6.1 Registration & Verification

| ID | Requirement |
|---|---|
| FR-TU-001 | Allow tutor registration and login. |
| FR-TU-002 | Require acceptance of the Privacy Policy, Terms & Conditions, and Rules & Regulations before registration is finalized. |
| FR-TU-003 | Allow the tutor to provide a profile including experience and qualification information. |
| FR-TU-004 | Require Admin verification and approval before a tutor becomes visible to students. |
| FR-TU-005 | Award tutor badges based on experience, performance, and platform achievements (not on ratings — see FC-01). |

### 6.2 Subject Assignment

> ✅ **RESOLVED — v2.0 — Shortage-Triggered Second Subject**
> Grade is always a hard-match requirement. Flow: the system searches for a primary-subject match within the student's grade and filters; if none is found, it automatically attempts a secondary-subject match (same grade + other filters). If that also finds no match, the student may manually trigger the "No Exact Match" flow (1-to-1 only; Section 8, Path B) for Admin assignment; group formats route the double-fail straight to Admin (Section 8, Path C).

| ID | Requirement |
|---|---|
| **FR-TU-006** *(amended)* | Allow the tutor to select and rank subjects by preference/strength, most to least preferred, up to a maximum of two subjects per tutor. |
| FR-TU-007 | Assign a tutor to one subject under normal conditions (primary subject). |
| FR-TU-008 | Automatically consider a tutor's secondary subject only when a primary-subject search finds no match for the same grade and filters (system-triggered, not Admin-initiated). |

### 6.3 Availability, Matching & Scheduling

| ID | Requirement |
|---|---|
| FR-TU-009 | Allow the tutor to set and manage their availability and schedule. |
| FR-TU-010 | Notify the tutor of student matching requests and booking confirmations, for all formats. |
| FR-TU-011 | Display assigned students and upcoming classes to the tutor. |

### 6.4 Conducting Classes

| ID | Requirement |
|---|---|
| FR-TU-012 | Send the tutor a reminder and countdown 1 hour before each class. |
| FR-TU-013 | Require the tutor to send the session link at least 30 minutes before class (Jitsi, see Section 9). |
| FR-TU-014 | Require the tutor to record every session per platform rules, obtaining participant consent per FR-SC-008, and upload the recording immediately after class. |
| FR-TU-015 | Route each uploaded recording automatically to the correct student's Library. |
| FR-TU-016 | Allow the tutor to upload learning materials (PDFs, notes, books). |
| FR-TU-017 | Allow the tutor to submit student feedback and weekly assessments, and view student progress. |

### 6.5 Earnings

| ID | Requirement |
|---|---|
| FR-TU-018 | Allow the tutor to view earnings and payment history. |
| **FR-TU-019** *(amended)* | Automatically pay out tutor earnings on a fixed monthly cycle (Section 13); no manual payout request is required. The tutor may view the upcoming payout date and amount at any time. |

*Retired: display of tutor ratings/reviews (formerly FR-TU-020) — removed, see FC-01. This ID is not reused.*

### 6.6 Support & Messaging

| ID | Requirement |
|---|---|
| FR-TU-021 | Deliver relevant platform notifications to the tutor. |
| FR-TU-022 | Allow the tutor to report problems or submit claims. |
| FR-TU-023 | Provide the tutor a channel to contact AKEWTutor support. |
| **FR-TU-024** *(new)* | Allow the tutor to exchange in-platform text messages with each currently assigned student/parent, per Section 5.11 (FR-MS-001–004). |

**Definition of Done — Section 06 — Tutor Experience**
1. A tutor cannot rank a third subject — the profile form hard-caps subject selection at two.
2. A tutor's secondary subject is never shown to students or used in matching unless the primary-subject search has already failed for that grade.
3. At month-end, a verified tutor's earnings are paid out automatically with no "request payout" action required anywhere in the tutor UI.
4. A tutor sees no rating/review widget anywhere in their dashboard.
5. A tutor can message only students they are currently assigned to, and loses that thread once the assignment ends (subject to the 90-day archive window).

---

## 07 Tutoring Formats & Pricing Model

AKEWTutor supports three tutoring group formats at launch, confirmed as final. No additional group sizes are in scope.

| ID | Format / Group Size | Price/Student/Hr | Total/Hr | AKEWTutor Share | Tutor Share |
|---|---|---|---|---|---|
| FR-PR-001 | 1-to-1 (1 tutor + 1 student) | 350 ETB | 350 ETB | 100 ETB | 250 ETB |
| FR-PR-002 | 1-to-3 (1 tutor + up to 3 students) | 150 ETB | 450 ETB | 150 ETB | 300 ETB |
| FR-PR-003 | 1-to-5 (1 tutor + up to 5 students) | 100 ETB | 500 ETB | 175 ETB | 325 ETB |

| ID | Requirement |
|---|---|
| FR-PR-004 | Allow the Admin to configure pricing and revenue-split figures for each of the three formats without a code change. |

> ✨ **NEW IN v3.0 — Partial Group Formation (Gap 1)**
> For 1-to-3/1-to-5 formats, the system does not wait indefinitely for a full group. A group-formation window opens once the first student is auto-matched to a candidate class; the window length defaults to **48 hours** and is Admin-configurable. If the window closes with fewer than the target group size but at least one student matched, the class proceeds at whatever size was reached (floor of one student). **Per-student pricing is fixed at the FR-PR-002/003 rate regardless of final group size** — a 1-to-3 class that starts with two students still charges each student 150 ETB/hr; the platform and tutor absorb the shortfall against the Total/Hr and shares shown above rather than passing it to students. See Section 8, Path C for the matching mechanics.

**Definition of Done — Section 07 — Formats & Pricing**
1. Admin can change the 1-to-1 price and revenue split from the console and see it reflected on the next new booking with no deployment.
2. A 1-to-5 class that forms with only 3 students after the 48-hour window still bills each of those 3 students at 100 ETB/hr, not a recalculated rate.
3. No format other than 1-to-1 / 1-to-3 / 1-to-5 can be selected anywhere in the student-facing UI.

---

## 08 Tutor–Student Matching Workflow

Matching considers the student's subject, grade (always a hard match), academic level, preferences (Section 5.2, applied per the rules in 5.2's callout), schedule, and tutor availability. The workflow branches by tutoring format.

### Path A — 1-to-1 (Student-Selected Match)

| ID | Requirement |
|---|---|
| FR-MA-001 | System recommends suitable 1-to-1 tutors with a calculated match percentage. |
| FR-MA-002 | Student reviews recommended tutors and selects one. |
| FR-MA-003 | Admin is notified of the student's tutor selection and reviews the booking. |
| FR-MA-004 | Admin approves the booking. |
| FR-MA-005 | System requires payment completion. |
| FR-MA-006 | System confirms the tutoring schedule. |

### Path B — 1-to-1, No Exact Match

| ID | Requirement |
|---|---|
| FR-MA-007 | Allow the student to manually select "No Exact Match," sending a request to Admin. This is also the automatic hand-off point when the secondary-subject search (FR-TU-008) also fails. |
| FR-MA-008 | Allow Admin to review available tutors and manually assign a suitable one. |
| FR-MA-009 | Notify both student and tutor once a manual assignment is made. |
| FR-MA-010 | Require payment completion after manual assignment. |
| FR-MA-011 | Confirm the schedule after payment. |

### Path C — 1-to-3 / 1-to-5 (System Auto-Match)

> ✅ **RESOLVED — v2.0 — Format-Based Matching Split**
> For 1-to-3 and 1-to-5, students do not search or choose a tutor; given the platform's current user volume, the system automatically matches students into a class and no match information (match %, tutor profile) is shown to the student pre- or post-assignment. Admin approval is still required before the schedule is confirmed.

| ID | Requirement |
|---|---|
| FR-MA-012 | Automatically group and match students requesting 1-to-3 or 1-to-5 formats to an available, subject/grade-eligible tutor, without student search or selection, subject to the group-formation window in Section 7. |
| FR-MA-013 | Notify Admin of each system-generated auto-match for review. |
| FR-MA-014 | Require Admin approval of the auto-match before the schedule is confirmed. |
| FR-MA-015 | Require payment completion after Admin approval. |
| FR-MA-016 | Confirm the schedule after payment; notify student and tutor. No match percentage or tutor-selection UI is shown to the student for these formats. |
| **FR-MA-017** *(new)* | If a primary- and secondary-subject auto-match search both fail for a 1-to-3/1-to-5 request, notify Admin immediately for manual group assembly — there is no student-facing "No Exact Match" trigger for group formats (that control is 1-to-1 only, FR-MA-007). |

### Cross-Path Rules

> ✨ **NEW IN v3.0 — Admin Rejection Handling (Gap 9)**
> If Admin rejects a booking rather than approving it: for **Path A**, the student is returned to the tutor-recommendation list (FR-MA-001) with the rejected tutor excluded from that student's next recommendation set; for **Path C**, the student's request re-enters the auto-match queue and a new candidate class is sought. In both cases the student/parent is notified of the rejection with a generic reason ("assignment could not be confirmed") and no payment is requested until a new match is approved.

> ✨ **NEW IN v3.0 — Group Continuity on Tutor Exit (Gap 6 / Round 4 Item 4)**
> If a tutor is removed from an active 1-to-3/1-to-5 class — whether by voluntary drop-out or Admin suspension — the existing student group is kept together as a unit and re-matched to a new eligible tutor via Path C rather than being dissolved and re-queued individually. If no single tutor can take the full group, the group falls back to the same double-fail handling as FR-MA-017 (Admin manual assembly, splitting the group only if unavoidable). Affected students are notified and are entitled to the refund policy in Section 13 for any un-tutored paid days during the gap.

**Definition of Done — Section 08 — Matching Workflow**
1. A 1-to-1 student who rejects/is rejected sees a fresh recommendation list that excludes the previously rejected tutor.
2. A 1-to-3/1-to-5 student is never shown a tutor name, photo, or match percentage before Admin approval.
3. When both primary and secondary subject searches fail for a group-format request, Admin receives a notification without the student ever seeing a "No Exact Match" button.
4. If a tutor is suspended mid-cohort, the affected students reappear together as one group in Admin's re-match queue, not as individual re-matching requests.

---

## 09 Class Delivery, Recording & Make-up Management

### 9.1 Session Delivery & Recording

> ✅ **RESOLVED — v3.0 — Video Conferencing Method**
> **Jitsi** (public instance at launch, with a self-hosted instance evaluated for a later phase as usage grows) is confirmed as the video-conferencing tool, closing the item left open in v2.0 Section 18. The workflow is unaffected: the tutor manually generates and shares a session link at least 30 minutes before class (FR-CD-003).

| ID | Requirement |
|---|---|
| FR-CD-001 | Attach a fixed scheduled time to every confirmed session, for all formats. |
| FR-CD-002 | Send both tutor and student a reminder 1 hour before class, with a countdown. |
| FR-CD-003 | Require the tutor to provide the Jitsi session link at least 30 minutes before class. |
| FR-CD-004 | Require the tutor to record the session per platform rules and upload it immediately after class. |
| FR-CD-005 | Store each recording in the correct student's Library automatically, per the storage architecture in Section 5.8. |
| FR-CD-006 | Restrict students to accessing only their own class recordings. |
| FR-CD-007 | Allow tutors to access recordings/classes associated with their own students, as permitted by platform rules. |
| FR-CD-008 | Grant Admin full management access to recordings and the Library. |
| FR-CD-009 | Allow storage of educational PDFs, books, and notes in the Library alongside recordings. |

> ✨ **NEW IN v3.0 — Failed Recording Upload (Gap 8)**
> If a tutor's recording upload fails or is not received within 2 hours of a session's scheduled end, the system flags the session as "recording missing" on both the tutor's and Admin's dashboards and sends the tutor a retry prompt. The tutor has 24 hours to upload before the session is escalated to Admin as a compliance issue (ties FR-TU-014, FR-AD-003). A missing recording does not, by itself, trigger a refund or make-up — the class itself still occurred — unless the student separately reports the class as not delivered, in which case it is handled under Section 9.2.

### 9.2 Make-up, Reschedule & Cancellation (New in v3.0)

> ✨ **NEW IN v3.0 — Closing the Missed-Session Gap (Gaps 4 & 5)**
> v2.0 had no defined behavior for a missed or rescheduled class. v3.0 introduces a dedicated FR-MK requirement area, distinguishing who caused the miss (tutor vs. student) and separating a genuine reschedule from a late cancellation.

| ID | Requirement |
|---|---|
| **FR-MK-001** | Where a tutor causes a missed session (no-show, late cancellation inside 12 hours, technical failure on the tutor's side), provide the student a mandatory free make-up session within 7 days, at no additional charge and with no refund/credit issued for that session. |
| **FR-MK-002** | Where a student causes a missed session (no-show, late cancellation inside 12 hours), provide no make-up and no refund/credit — the session counts as delivered against that month's billed sessions. |
| **FR-MK-003** | Escalate a tutor to mandatory Admin review after two or more tutor-caused misses within a rolling 30-day period (ties FR-AD-003, FR-SC-007). |
| **FR-MK-004** | Allow either party to request a reschedule of a specific upcoming session with at least 12 hours' notice, at no charge and with no make-up/refund implication. |
| **FR-MK-005** | Restrict outright cancellation of a confirmed session to the tutor or Admin; a student-initiated cancellation is instead processed under the no-show/make-up rules (FR-MK-001/002) according to how much notice was given. |
| **FR-MK-006** | Treat a reschedule request made inside the 12-hour window as a same-day miss, subject to the tiered tutor-caused / student-caused rules above rather than the free-reschedule path. |
| **FR-MK-007** | Cap free reschedules at 2 per student per month before further reschedule requests require Admin discretion. |
| **FR-MK-008** | Limit a reschedule to shifting the fixed session time (FR-CD-001) within the tutor's existing availability; a reschedule carries no billing impact and does not consume a make-up session. |

**Definition of Done — Section 09 — Class Delivery & Make-up**
1. A tutor no-show automatically queues a free make-up session for the student within a 7-day window, with no charge and no separate refund request needed.
2. A student no-show is logged as a delivered session and does not generate a make-up or a support ticket by default.
3. A tutor with two no-shows in 30 days appears on Admin's escalation list without Admin needing to search for it.
4. A reschedule requested 6 hours before class is routed through FR-MK-006's same-day-miss logic, not treated as a free reschedule.
5. A recording that never arrives is visible as "missing" to Admin within 2 hours of the session's scheduled end time.

---

## 10 Learning Engagement & Gamification

| ID | Requirement |
|---|---|
| FR-GA-001 | Provide weekly student assessments and progress tracking, with tutor feedback. |
| **FR-GA-002** *(amended)* | Award XP / learning points and maintain a student leaderboard, scoped per grade level and displaying students by first name and last-initial only (never full name), with weekly and monthly rankings. |
| FR-GA-003 | Award achievements, badges, and track streaks for consistent learning. |
| FR-GA-004 | Present weekly and monthly learning challenges, and milestones. |
| FR-GA-005 | Award tutor badges based on experience, performance, and achievements (not ratings). |
| FR-GA-006 | Recognise improvement, consistency, and academic effort. |
| FR-GA-007 | Generate personalized tutor recommendations for each student (1-to-1 only) — see FR-SP-020 for the matching criteria. |

> ✨ **NEW IN v3.0 — Leaderboard Scope & Privacy (Round 4 Item 6)**
> The original leaderboard requirement did not specify scope or student-identity handling. v3.0 confirms leaderboards are always **per-grade** (a Grade 4 student never appears against Grade 10 students) and display **first name + last initial only**, consistent with the minor-protection stance in Section 14.

**Definition of Done — Section 10 — Gamification**
1. The leaderboard for a Grade 3 student contains only other Grade 3 students.
2. No leaderboard entry anywhere in the product shows a student's full last name.
3. Tutor badges are computed from experience/performance/achievement fields only — no rating field exists to award them from.

---

## 11 Administrator Dashboard — Functional Requirements

### 11.1 People & Verification

| ID | Requirement |
|---|---|
| FR-AD-001 | Allow Admin to manage students, parents, and tutors, including parent–student relationship records (Section 4.2). |
| FR-AD-002 | Allow Admin to approve or reject tutor registrations and verification. |
| FR-AD-003 | Allow Admin to suspend or restrict accounts when necessary, including tutors escalated under FR-MK-003. |
| FR-AD-004 | Allow Admin to manage tutor badges (performance/achievement-based, not rating-based). |

### 11.2 Bookings & Scheduling

| ID | Requirement |
|---|---|
| FR-AD-005 | Notify Admin of every tutor selection (1-to-1) or system auto-match (1-to-3/1-to-5) requiring review. |
| FR-AD-006 | Allow Admin to manually assign tutors for the 1-to-1 "No Exact Match" flow, and for group-format double-fail assembly (FR-MA-017). |
| FR-AD-007 | Allow Admin to approve bookings/auto-matches and manage schedules, for all formats, including approving or rejecting per the handling rules in Section 8. |
| FR-AD-008 | Allow Admin to monitor active and upcoming tutoring sessions. |

### 11.3 Money

| ID | Requirement |
|---|---|
| FR-AD-009 | Allow Admin to manage tutoring formats and prices. |
| FR-AD-010 | Allow Admin to manage payments and platform revenue. |
| FR-AD-011 | Allow Admin to manage tutor earnings and monthly payouts. |
| FR-AD-012 | Allow Admin to review and process prorated refunds per the policy in Section 13. |

> ✨ **NEW IN v3.0 — Stale Booking Approvals (Gap 10)**
> A booking or auto-match awaiting Admin approval is flagged **overdue** if not actioned within 48 hours, and surfaces at the top of Admin's review queue. If it remains unactioned past 5 days, the affected student/parent is proactively notified of the delay and the case is escalated within the support workflow (FR-AD-020). No automatic refund is triggered by a stale approval on its own — refunds still require the Section 13 policy conditions to be met.

> ✨ **NEW IN v3.0 — Suspension of an Active Tutor (Gap 6)**
> Admin-initiated suspension of a tutor who has active students is handled the same way as a voluntary tutor drop-out: affected 1-to-1 students re-enter Path B ("No Exact Match" → manual Admin assignment); affected group cohorts are kept together and re-matched as a unit per Section 8. The suspension reason is not disclosed to students. Admin retains discretion to expedite re-matching for large or exam-critical cohorts.

### 11.4 Content & Platform

| ID | Requirement |
|---|---|
| FR-AD-013 | Allow Admin to manage subjects and grade levels. |
| FR-AD-014 | Allow Admin to manage the educational Library and control recording access and retention, including reviewing sessions flagged "recording missing" (Section 9.1). |
| FR-AD-015 | Allow Admin to manage the Privacy Policy, Terms, and Rules & Regulations. |
| FR-AD-016 | Allow Admin to manage promotional offers and discounts. |

### 11.5 Oversight & Reporting

| ID | Requirement |
|---|---|
| FR-AD-017 | Allow Admin to manage complaints, claims, and disputes, including reviewing flagged in-platform message threads (FR-MS-004). |
| FR-AD-018 | Allow Admin to manage leaderboards and student achievements, and monitor assessments and progress. |
| FR-AD-019 | Allow Admin to manage notifications and platform announcements. |
| FR-AD-020 | Provide Admin with customer support management tools, including the stale-approval escalation workflow (Gap 10). |
| FR-AD-021 | Provide Admin with platform statistics, reports, and activity history. |
| **FR-AD-022** *(renumbered)* | Provide Admin oversight of tutor performance and badge history for quality-monitoring purposes (renumbered from a duplicate v1.0 ID formerly shared with the retired tutor-review management item; no functional change). |

*Retired: management of tutor reviews (formerly FR-AD-020 in v1.0) — removed, see FC-01. The FR-AD-020 ID was reassigned to Customer Support Management Tools above to remove a duplicate-ID collision found during the v3.0 audit; no requirement content changed.*

**Definition of Done — Section 11 — Administrator Dashboard**
1. Any approval item sitting for 48+ hours is visibly flagged and sorted to the top of Admin's queue without a manual filter.
2. A tutor suspended while actively teaching a 1-to-3 cohort produces one group re-match case, not three individual ones.
3. No two functional requirements in the final document share the same FR-AD ID.
4. A rejected booking's affected student never sees an internal suspension/rejection reason string, only a generic status message.

---

## 12 Notification Framework

| ID | Requirement |
|---|---|
| FR-NO-001 | Notify on registration completion and tutor approval. |
| FR-NO-002 | Notify on tutor matching/auto-match and booking confirmation. |
| FR-NO-003 | Notify on Admin approval and payment confirmation. |
| FR-NO-004 | Notify of upcoming class, 1-hour reminder/countdown, and the session link. |
| FR-NO-005 | Notify of class cancellation, reschedule, or a make-up session being scheduled (Section 9.2). |
| FR-NO-006 | Notify of assessment availability and weekly performance summary. |
| FR-NO-007 | Notify when new recordings or materials are uploaded. |
| FR-NO-008 | Notify of achievement/badge earned and leaderboard changes. |
| FR-NO-009 | Notify of payment confirmation and tutor earnings updates. |
| FR-NO-010 | Notify of claims/support updates and important platform announcements. |
| **FR-NO-011** *(new)* | Notify a student/tutor when they receive a new in-platform message (FR-MS-001), via the same notification channel used for other in-app alerts. |

**Definition of Done — Section 12 — Notifications**
1. Every FR-MK make-up/reschedule event produces a notification, not just original bookings.
2. A new message thread reply triggers a notification within the same delivery pipeline (push/SMS/email per user preference) as a class reminder.

---

## 13 Payment & Billing Policy

| ID | Requirement |
|---|---|
| FR-PB-001 | Require payment completion after tutor matching/assignment and Admin approval, before the schedule is confirmed. |
| FR-PB-002 | Allow students and parents to view payment history at any time. |
| **FR-PB-003** *(amended)* | Send a payment reminder 3 days before the next monthly payment is due, with a visible countdown; the due date is anchored to each student's individual billing-cycle start date (the date their first paid session was confirmed), not a shared platform-wide billing date. |
| **FR-PB-004** *(amended)* | Require payment to be completed on the due day, or the next day at the latest, measured from that student's individual billing-cycle anchor date. |
| FR-PB-005 | Pause the tutoring schedule if payment is not received, showing only a "Pay for this month" prompt until payment is completed. |
| FR-PB-006 | Provide a direct emergency contact channel to Admin via phone number or Telegram. |
| FR-PB-007 | Process prorated refunds per the policy below, on Admin approval. |
| **FR-PB-008** *(new)* | Allow students in Grades 6–12 to complete payment independently via Chapa (Section 16) with no guardian account or guardian approval required. |

> ✨ **NEW IN v3.0 — Refund Proration Formula (Gap 7)**
> Prorated refunds (Section 3, FR-AD-012, FR-PB-007) are calculated by **sessions actually delivered**, not calendar days: refund = (sessions remaining in the paid month ÷ total sessions billed for that month) × amount paid. A session already delivered — including a free make-up session under FR-MK-001, which does not itself consume an extra billed slot — is never counted as "undelivered" for refund purposes.

> ℹ️ **Billing Cycle**
> Billing cycle confirmed as monthly: the hourly rates in Section 7 are used to calculate each student's monthly total, anchored to that student's individual billing start date per FR-PB-003.

**Definition of Done — Section 13 — Payment & Billing**
1. Two students who joined on different calendar dates receive their payment reminders 3 days before their own respective due dates, not a shared platform date.
2. A refund for 2 undelivered sessions out of 8 billed sessions returns exactly 2/8 of the amount paid, not a day-based fraction.
3. A free make-up session never appears as a separately billed or separately refundable line item.
4. A Grade 10 student with no linked guardian can complete a payment end-to-end via Chapa.

---

## 14 Safety, Trust & Compliance

| ID | Requirement |
|---|---|
| FR-SC-001 | Require acceptance of the Privacy Policy, Terms & Conditions, and Rules & Regulations before registration completes. |
| FR-SC-002 | Require tutor verification and ongoing Admin monitoring. |
| FR-SC-003 | Provide a student/tutor reporting and complaint/claim system. |
| FR-SC-004 | Restrict tutor–student voice/video communication to scheduled class sessions only; permit other in-platform interactions (quizzes, assessments, text messaging per Section 5.11, etc.) outside class time. No communication is permitted outside the platform. |
| FR-SC-005 | Protect student information; apply reasonable data-handling care for younger students as good practice (non-blocking design guideline). |
| FR-SC-006 | Enforce recording and Library access controls per Section 5.8. |
| FR-SC-007 | Allow account suspension or restriction for rule violations, including the tiered no-show escalation in FR-MK-003. |
| **FR-SC-008** *(new)* | Obtain recorded consent (a one-time acknowledgment at booking, plus an in-session indicator) from students/parents and tutors that class sessions are recorded, before the first recorded session for that pairing. |
| **FR-SC-009** *(new)* | Display a visible recording-in-progress indicator to all participants for the duration of a recorded session. |

> ℹ️ **Recording Consent (Round 4 Item 7)**
> v2.0 defined recording storage and retention but not participant consent. v3.0 closes this with a one-time consent acknowledgment (FR-SC-008) plus a persistent in-session visual indicator (FR-SC-009), satisfying baseline consent practice without requiring a re-consent flow on every individual class.

*Retired: the tutor review/rating system present in v1.0 has been removed platform-wide (Feature Change FC-01).*

**Definition of Done — Section 14 — Safety, Trust & Compliance**
1. A tutor cannot start a first-ever recorded session with a given student before that student/parent has acknowledged the recording-consent notice.
2. Every recorded session shows a persistent recording indicator visible to both participants throughout.
3. No tutor–student contact channel exists outside the platform's scheduled sessions and text messaging.

---

## 15 Non-Functional Requirements

Confirmed by client as final — no changes from v2.0.

### 15.1 Platform & Device Support

| ID | Requirement |
|---|---|
| NFR-001 | Support current versions of major desktop browsers (Chrome, Safari, Edge, Firefox). |
| NFR-002 | Support mobile web browsing on Android and iOS devices, with a responsive layout. |

### 15.2 Performance

| ID | Requirement |
|---|---|
| NFR-003 | Core pages (homepage, dashboard, tutor search) should load within 3 seconds on a standard broadband connection. |
| NFR-004 | The system should support concurrent use by all registered students and tutors during peak class hours without degraded performance. |

### 15.3 Availability

| ID | Requirement |
|---|---|
| NFR-005 | Target platform uptime of 99.5% or higher, excluding scheduled maintenance windows. |
| NFR-006 | Scheduled maintenance should avoid peak tutoring hours where possible. |

### 15.4 Security & Data Privacy

| ID | Requirement |
|---|---|
| NFR-007 | Encrypt user data in transit (HTTPS) and at rest for sensitive fields (payment details, contact information). |
| NFR-008 | Store passwords using a secure hashing algorithm; never store plain-text passwords. |
| NFR-009 | Restrict access to student data and recordings strictly according to the roles and rules defined in Sections 5, 9, and 14. |
| NFR-010 | Retain personal data only as long as necessary and in line with the platform's Privacy Policy. |

### 15.5 Scalability

| ID | Requirement |
|---|---|
| NFR-011 | Architecture should allow the addition of new subjects and grade levels without structural rework (group formats fixed at three per Section 7). |
| NFR-012 | Media storage (recordings, materials) should scale independently as usage grows, per the architecture in Section 5.8. |

---

## 16 Third-Party Dependencies & Integrations

> ✅ **RESOLVED — v3.0 — Third-Party Providers — Fully Closed**
> All provider decisions left open in v2.0 Section 18 are now confirmed, closing both Section 18 items.

| Dependency | Purpose | Status |
|---|---|---|
| Video Conferencing | External video conferencing for tutoring sessions | Confirmed — Jitsi (public instance at launch) |
| Payment Gateway | Processing student/parent payments in ETB | Confirmed — Chapa |
| SMS Provider | Phone number verification and SMS notifications | Confirmed — Geez SMS |
| Email Provider | Email verification and notification delivery | Confirmed — Brevo |
| Telegram | Direct emergency contact channel with Admin | Confirmed by client (unchanged from v2.0) |
| Cloud Object Storage | Storing class recordings and Library materials | Confirmed — Cloudflare R2 |

These are provider decisions, not yet technical integration specifications — API contracts, webhook design, and failover behavior for each provider are Technical Specification concerns and follow from this document.

---

## 17 End-to-End User Journey

The full platform experience, from a student's point of view, in a single flow:

1. Register
2. Select learning needs & format
3. Get matched with a tutor
4. Choose tutor (1-to-1) / auto-match (1-to-3/5)
5. Admin approval
6. Pay
7. Schedule confirmed
8. Receive reminder
9. Join class
10. Access recording & materials
11. Weekly assessment
12. Track progress
13. Continue learning — message tutor, reschedule, or trigger make-up as needed

---

## 18 Resolution Record — v2.0 to v3.0

Both items open at the end of v2.0 are now resolved, and this section consolidates every decision made since, drawn from the AKEWTutor SRS Resolution Log across five consistency-audit rounds. **No items remain open.**

### 18.1 Former Section 18 Items — Both Closed

| Topic | Resolution |
|---|---|
| Video conferencing tool | Jitsi confirmed (public instance at launch; self-hosted evaluated for later phase). See Section 9.1. |
| Third-party providers | Chapa (payment), Geez SMS (SMS), Brevo (email), Cloudflare R2 (storage) all confirmed. See Section 16. |

### 18.2 Numbered Gaps Resolved (Rounds 1–3)

| Gap | Topic | Resolution |
|---|---|---|
| Gap 1 | Partial group formation for 1-to-3/1-to-5 | 48-hour Admin-configurable formation window; class proceeds at reached size (floor of 1); per-student price fixed regardless of final size. → Section 7. |
| Gap 2 | Independent minor payment | Grades 6–12 students may pay via Chapa with no guardian required. → Section 13 (FR-PB-008). |
| Gap 3 | Preference handling in matching | Budget = hard filter (1-to-1 only); language = hard filter (all formats); teaching style = soft/scored (1-to-1 only). → Section 5.2. |
| Gap 4 | Tutor-caused missed session | Mandatory free make-up within 7 days, no refund. → Section 9.2 (FR-MK-001). |
| Gap 5 | Student-caused missed session / reschedule vs. cancellation | No make-up/refund for student-caused misses; 12-hour free-reschedule window; cancellation restricted to tutor/Admin. → Section 9.2 (FR-MK-002, 004–006). |
| Gap 6 | Tutor suspension / group continuity | Suspension treated like voluntary drop-out; groups kept together and re-matched as a unit. → Sections 8, 11.3. |
| Gap 7 | Refund proration basis | Proration by sessions delivered, not calendar days. → Section 13. |
| Gap 8 | Failed recording upload | Flagged "recording missing" after 2 hours; 24-hour tutor retry window before escalation. → Section 9.1. |
| Gap 9 | Admin rejection handling | 1-to-1 returns to recommendation list (tutor excluded); group format re-enters auto-match queue. → Section 8. |
| Gap 10 | Stale booking approvals | Flagged overdue at 48 hours, escalated with student notification at 5 days. → Section 11.3. |
| Gap 11 | Group-format double subject-match failure | Routes directly to Admin manual assembly — no student-facing trigger for group formats. → Section 8 (FR-MA-017). |

Three minor items were also resolved in this range: a wording clarification on FR-SP-025 (unique vs. total student count), a wording clarification on FR-SP-030 / Path C profile visibility, and confirmation that recordings are encoded at 720p compressed (not raw).

### 18.3 Round 4 — SRS-Level Audit (8 Items)

| # | Topic | Resolution |
|---|---|---|
| 1 | Messaging gap | Added FR-MS-001–004 (Section 5.11) — in-platform, text-only, matched-pair messaging. |
| 2 | Account-model grade capture | Amended FR-AC-002/003 to capture grade at invite time and add a 14-day expiring invite with a day-7 reminder. → Section 4.2. |
| 3 | Tutor subject cap | Amended FR-TU-006 to cap ranked subjects at two per tutor. → Section 6.1. |
| 4 | Tutor payout trigger | Amended FR-TU-019 to automatic monthly payout — removed the manual "request payout" step. → Section 6.5. |
| 5 | Sole-guardian removal | Added FR-AC-008: a "guardian required" hold state (access paused, no data lost) rather than an undefined account state. → Section 4.2. |
| 6 | Leaderboard scope & privacy | Amended FR-GA-002 to per-grade scope, first-name + last-initial display. → Section 10. |
| 7 | Recording consent | Added FR-SC-008/009 — one-time consent acknowledgment plus an in-session recording indicator. → Section 14. |
| 8 | FR-AD-020 ID collision | Reassigned FR-AD-020 to Customer Support Management Tools; the retired v1.0 tutor-review item is superseded, not merged. → Section 11.5. |

### 18.4 Round 5 — Final Consistency Check (5 Items)

| # | Topic | Resolution |
|---|---|---|
| 1 | Message notifications | Added FR-NO-011 so new messages notify through the standard notification pipeline. → Section 12. |
| 2 | Billing-cycle anchor | Amended FR-PB-003/004 so due dates anchor to each student's individual join date, not a shared platform date. → Section 13. |
| 3 | Guardian-removal cross-reference | Confirmed FR-AC-007/008 read consistently: revocation triggers the hold state, it does not delete the relationship record. → Section 4.2. |
| 4 | Group cohesion on suspension — cross-check | Confirmed Section 8's Path C group-continuity language and Section 11.3's Gap 6 box describe the same single behavior, not two conflicting ones. |
| 5 | FR-SP-025 unique-count wording | Confirmed "total students taught" means a distinct/unique student count, worded explicitly in Section 5.6. |

> ✅ **RESOLVED — v3.0 — Overall Status**
> Every item raised across the v2.0 client review and all five subsequent consistency-audit rounds is now resolved and reflected in the numbered sections above. **Zero items remain open.** This document is ready to serve as the basis for the Technical/System Specification.

---

## 19 Approval & Sign-off

By signing below, the client confirms that this v3.0 Software Requirements Specification — the final requirements baseline, with no open items — accurately reflects the scope of work to be delivered, and authorizes the design and development team to proceed to Technical/System Specification on this basis.

**Client / Project Owner**

Name: _______________________________ &nbsp;&nbsp; Signature: _______________________________ &nbsp;&nbsp; Date: _______________

**Agency / Development Team**

Name: _______________________________ &nbsp;&nbsp; Signature: _______________________________ &nbsp;&nbsp; Date: _______________
