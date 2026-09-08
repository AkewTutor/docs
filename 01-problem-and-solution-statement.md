## 1. Problem & Solution Statement

**Project:** AKEWTutor — Online Tutoring Platform

**Owners:**

| No. | Name | Id |
|---|---|---|
| 1 | _______________________________ | _______________ |
| 2 | _______________________________ | _______________ |
| 3 | _______________________________ | _______________ |

**Status:** v1.0 — Reconstructed from the v3.1 Requirements Baseline

**Links back to:** — (this is the first document in the sequence)
**Links forward to:** [02. Requirements]

---

> ℹ️ **How this document came to exist**
> AKEWTutor's original SRS (v3.0/v3.1) combined problem/solution narrative and functional requirements into a single document. This file extracts and separates that narrative layer, matching the project's standard two-document structure. Sections **1.5** (Scope) and **1.7** (Assumptions) are carried over directly from the original SRS with no changes in substance. Sections **1.1, 1.2, 1.4, 1.6, and 1.8** did not previously exist as separate written content — they were reconstructed based on what the SRS's decisions and scope already imply, and were confirmed by the project owner as matching original intent (in-conversation confirmation, September 2026 — see `00-pending-client-actions.md`, Action 2, closed). No further review of these five sections is needed.

---

### 1.1 Problem Statement

Access to quality academic support outside the classroom is inconsistent and difficult to arrange reliably for students in Grades 1–12. Families seeking a tutor typically rely on informal referrals, with no reliable way to verify a tutor's qualifications, compare tutors by subject and budget, or confirm that sessions consistently happen as agreed. There is no dependable system for scheduling, payment, or oversight — a family has no visibility into whether a class actually took place, no structured way to request a make-up session when it doesn't, and no safe, moderated channel to communicate with a tutor without exchanging personal contact details.

For younger learners in particular (Grades 1–5), parents need direct oversight and control over who their child interacts with, how sessions are recorded, and how payment works — none of which an informal, referral-based arrangement provides. For older students (Grades 6–12), the friction runs the other way: many are capable of managing their own learning and payments independently, but informal tutoring still offers no structured record of progress, materials, or session history.

On the tutor side, qualified educators have no structured platform to reach students matched to their subject expertise, manage their availability, build a track record, or receive dependable payment on a predictable schedule — their income and reach depend on ad hoc referrals rather than a system that actively connects them with demand.

The core gap: there is no trustworthy, structured platform connecting students and tutors in Ethiopia that combines vetted matching, transparent scheduling, safe communication, recorded accountability, and dependable payment — in ETB, and adapted to local constraints (e.g., no assumption of a native video-conferencing build, and a payment/SMS/email stack suited to the local market).

### 1.2 Target Users

| User type | Who they are | What they need from this system |
|---|---|---|
| Student (Grades 6–12) | A learner old enough to manage their own account, subjects, schedule, and payment independently, with an optional guardian link. | Find a suitably matched, verified tutor; book and pay for sessions; attend classes reliably; track progress and recordings; message their tutor safely. |
| Student (Grades 1–5) | A younger learner whose account exists under a parent/guardian's oversight. | The same learning experience and safety protections as older students, without needing to manage registration, matching approval, or payment themselves. |
| Parent / Guardian | The adult responsible for a younger student's (or, optionally, an older student's) account, payments, and oversight. | Confidence that tutors are verified, sessions are recorded and safe, communication is moderated, payment is predictable, and they retain control over their child's account and data. |
| Tutor | A subject-qualified educator offering paid tutoring across one or two ranked subjects. | A steady, structured way to reach matched students, manage availability, deliver and record classes, communicate safely with assigned students, and receive dependable, automatic monthly pay. |
| Administrator | The AKEWTutor project/operations team. | Full oversight and control: verify tutors, approve every match before it's billed, manage pricing and payouts, resolve disputes and complaints, and monitor platform health and safety. |

### 1.3 Proposed Solution

AKEWTutor is an online tutoring platform connecting students in Grades 1–12 with qualified, Admin-verified tutors, in one, three, or five-student class formats priced accordingly. Every match — whether student-selected (1-to-1) or system auto-matched (1-to-3/1-to-5) — passes through Admin review before payment and scheduling are confirmed, giving the platform a consistent trust-and-safety gate regardless of format.

Once matched, classes are delivered over an externally generated Jitsi video link, recorded (with participant consent) and automatically routed to the student's personal Library, and backed by a structured make-up/reschedule policy that fairly distinguishes a tutor-caused miss from a student-caused one. A lightweight, text-only, in-platform messaging channel lets a student/parent and their assigned tutor(s) coordinate directly without ever needing to exchange outside contact details. A gamification layer (XP, badges, streaks, per-grade leaderboards) is layered on top to support engagement, particularly for younger learners, without exposing any student's full identity publicly.

The account model is deliberately split by age band: Grades 1–5 require parent-first registration and an active guardian relationship before the student can access anything, while Grades 6–12 students can register, get matched, and pay independently, with a guardian link available but optional. This reflects a real difference in oversight needs rather than treating all ages identically.

An Admin console sits behind all of this, giving the operations team full control over tutor verification, pricing and revenue splits (configurable without a code deployment), payouts, refunds, dispute resolution, and platform-wide safety monitoring — including message-thread review for disputes and proactive escalation when a booking approval or tutor performance issue sits unresolved too long.

### 1.4 Why This Approach (vs. alternatives considered)

> ✅ This section is reconstructed from the decisions already recorded in the v3.1 SRS. Confirmed by the project owner as matching original intent, including the one previously-inferred rationale below (FC-01) — see the confirmation note on that item.

**Admin-mediated approval for every match, rather than fully automated instant booking.** Every 1-to-1 selection and every 1-to-3/1-to-5 auto-match still requires Admin sign-off before payment and scheduling are confirmed (Sections 8, 11.2 of the SRS). This was chosen over a fully automated instant-booking flow because the platform connects minors with adult tutors — a human review step before money and scheduling are finalized is a deliberate safety control, not just a workflow inefficiency to be automated away later.

**Independent Parent and Student accounts, split by grade band, rather than one shared account model.** Grades 1–5 require parent-first registration and a mandatory active guardian; Grades 6–12 can register and pay independently. This was chosen because a single account model would either over-restrict capable older students or under-protect younger ones — the age-based split lets each group's actual oversight needs drive the account design instead of applying one policy to both.

**An external Jitsi link rather than a native, built-in video-conferencing system.** This was chosen to avoid the cost and engineering complexity of building real-time video infrastructure for the initial release. The SRS itself notes a self-hosted Jitsi instance is being evaluated for a later phase as usage grows (Section 9.1), which suggests this was treated as a "prove the model first, invest in infrastructure later" decision rather than a permanent architectural choice.

**A hard two-subject cap per tutor, rather than unlimited subjects.** Tutors rank up to two subjects, with the second only engaged as a system-triggered fallback when a primary-subject search fails (Section 6.2). This was chosen to keep tutor profiles focused and matching quality high, rather than allowing a tutor to spread thin across many subjects at the cost of teaching depth in any one of them.

**Removal of the tutor rating/review system, replaced with achievement-based badges.** *(The SRS records that this was removed, "Feature Change FC-01," but did not originally record the reasoning; the rationale below was inferred and has since been confirmed by the project owner as correct — September 2026.)* Public, subjective star-ratings can unfairly and disproportionately affect a tutor's income based on a small number of reviews, while an objective, achievement/experience-based badge system rewards consistent performance without exposing tutors to that risk.

**Chapa, Geez SMS, Brevo, and Cloudflare R2 as the third-party stack, rather than building custom equivalents.** These were chosen for direct compatibility with the Ethiopian market — ETB-denominated payment processing, local SMS delivery — while relying on managed infrastructure for storage and email rather than building and maintaining that infrastructure in-house for an initial release.

### 1.5 Scope

*(Carried over from the original SRS, Section 1.2 — unchanged in substance.)*

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

**Explicitly out of scope for this phase:**
- Native mobile applications (iOS / Android)
- Built-in video conferencing — sessions rely on an externally generated Jitsi link
- Multi-currency or international payment support beyond ETB
- Support for learners outside Grade 1–12 (possible future expansion)
- Technical integration with school information systems
- Voice, video, or file-attachment messaging (in-platform messaging is text-only)

### 1.6 Success Criteria

*(Newly drafted — synthesized from the Definition of Done blocks throughout the v3.1 SRS. Please confirm these represent the right bar for calling this release complete.)*

- [ ] A Grade 6–12 student can register, build a profile, get matched, pay, and attend a class end-to-end with no guardian account required at any step.
- [ ] A Grade 1–5 student's account is fully guardian-gated: the student cannot be accessed or unlocked without an active, accepted guardian invite.
- [ ] All three formats (1-to-1, 1-to-3, 1-to-5) each support their full, distinct matching path (student-selected vs. system auto-matched) through to a confirmed, paid schedule.
- [ ] Every delivered class produces a recording that is automatically routed to the correct student's Library and is inaccessible to any other student.
- [ ] A tutor-caused missed session and a student-caused missed session are handled distinctly and automatically, with the correct make-up, refund, and tutor-pay consequences applied in each case.
- [ ] The Admin console supports the full operational loop — tutor verification, match approval, pricing/payout management, refund processing, and dispute/message-thread review — without requiring direct database access for any of it.
- [ ] Core child-safety protections are enforced platform-wide: recording consent before a first recorded session, no full-name exposure on leaderboards, and no tutor–student contact channel outside scheduled sessions and in-platform messaging.
- [ ] Pricing and revenue splits can be changed by Admin without a code deployment, and are reflected on the next new booking.

### 1.7 Assumptions & Open Questions

*(Carried over from the original SRS, Section 1.3, reformatted to track resolution status. All items below were confirmed by the client as part of the v2.0–v3.1 review process; none remain open.)*

| # | Assumption / Question | Owner | Resolved? |
|---|---|---|---|
| 1 | The platform operates primarily in Ethiopia, with pricing and payments in ETB. | Client | Yes |
| 2 | Tutoring sessions are conducted via an externally generated video link (Jitsi), not a native video system. | Client | Yes |
| 3 | Every booking passes through Admin review — there is no fully automated instant-booking path. | Client | Yes |
| 4 | Emergency support contact is manual, via phone number or Telegram. | Client | Yes |
| 5 | A tutor is normally linked to one subject; a second subject is a system-triggered exception under a declared shortage, and no tutor ranks more than two subjects. | Client | Yes |
| 6 | A tutor's ranked subject(s) apply across the full Grade 1–12 span, with no separate grade-range configuration required. | Client | Yes (v3.1) |
| 7 | Pricing and revenue-split figures are final at launch and Admin-configurable post-launch. | Client | Yes |

### 1.8 Version Roadmap

> ✅ This section is newly drafted for planning purposes, inferred from the "explicitly out of scope for this phase" language already used throughout the SRS. Confirmed by the project owner as matching intent (September 2026) — treated as a real staging plan for future phases, not just a discussion starting point.

The SRS repeatedly frames certain exclusions as scoped to "this phase" rather than permanently out of scope, which implies a staged path is already anticipated even though it was never written down as a formal roadmap. A plausible staging, based entirely on what's already marked as deferred:

- **This Release — Core Platform:** Everything in Section 1.5's in-scope list — web-based matching, booking, payment, class delivery, recordings, messaging, gamification, and full Admin oversight for Grades 1–12 in Ethiopia (ETB only).
- **Next Phase — Reach & Compatibility (candidate):** Native mobile applications (iOS/Android); multi-currency/international payment support, if expansion beyond Ethiopia is desired.
- **Later Phase — Scale & Integration (candidate):** Self-hosted Jitsi instance (already flagged for evaluation as usage grows); technical integration with school information systems; support for learners outside Grade 1–12.

**Next:** proceed to → [02. Requirements]
