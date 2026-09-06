# AKEWTutor — Use Cases

**Project:** AKEWTutor — Online Tutoring Platform

**Links back to:** [01. Problem & Solution Statement], [02. Requirements]
**Links forward to:** [04. Database & Data Model]

**Version:** 1.0 — derived from SRS v3.1 (Sections 04–14)
**Status:** Draft for review — every use case below traces to at least one FR-xxx / NFR-xxx ID from Doc 02. Later documents (06-api, 07-frontend-specification, 08-function-level-specification, 09-test-file-specification) should trace back to a use case number (UC-xx), not directly to an FR ID, so that a single UC can bundle several related requirements into one coherent user-facing flow.

---

> ℹ️ **Numbering discipline (read before adding any future use case)**
> Use cases are numbered sequentially in the order a real user moves through the system, grouped into lettered sections for readability. New use cases — including any added for a future release — must be appended as new numbers (UC-76, UC-77, ...) at the end of this document, even if they would conceptually fit earlier alongside a related section. Never renumber existing use cases and never insert a new use case in the middle of the sequence: doing so breaks every FR-to-UC reference this document establishes, and every downstream API/frontend/function/test reference that will point back to a UC number. If topical grouping matters for readability, use section headers and cross-references, not renumbering. Retired use cases (should any ever be retired) follow the same rule as retired FR IDs in Doc 02: the number is never reused, and a one-line "retired" note replaces the entry.

> ℹ️ **A note on actors**
> Five actor types recur throughout: **Anonymous Visitor** (not logged in), **Parent/Guardian**, **Student**, **Tutor**, and **Admin**. Several student/parent use cases apply identically regardless of which of the two is the one physically acting (e.g. a Grade 1–5 parent may search for a tutor on the student's behalf, per the account model in FR-AC-002–005). Where the account model in Section 4.2 makes this distinction load-bearing, it is called out explicitly in the use case; otherwise "Student/Parent" is used to mean "whichever of the two currently has access to act."

---

## A. Public Website & Marketing

#### UC-01: Browse the public marketing site

| Field | Detail |
|---|---|
| Actor | Anonymous Visitor |
| Precondition | None |
| Trigger | Visitor navigates to the homepage or any standalone public page |
| Linked FR | Section 03 (Public Website Structure) — informational content, no FR-xxx ID |

**Main flow:**
1. Visitor lands on the homepage and sees the platform introduction, "Find a Tutor" / "Become a Tutor" CTAs, how-it-works, benefits, featured tutors, popular subjects/grades, success stories, promotions, FAQ, and contact/login/registration entry points.
2. Visitor navigates to any standalone public page (About, Find a Tutor, Subjects, Grades 1–12, Exam Preparation, Become a Tutor, For Parents, For Schools, Pricing, Resources/Blog, FAQ & Contact, or a policy page).
3. System renders the requested page.

**Alternate / error flows:**
- None — these are static/semi-static informational pages with no auth-gated content.

**Postcondition (success):** Visitor understands the platform's value proposition and can proceed to registration or tutor discovery.

---

#### UC-02: Read the refund, privacy, safety, or terms policy

| Field | Detail |
|---|---|
| Actor | Anonymous Visitor / any authenticated role |
| Precondition | None |
| Trigger | User navigates to a policy page (Refund Policy, Privacy Policy, Safety Policy, Terms & Conditions) |
| Linked FR | Section 03 (Refund Policy — RESOLVED v2.0), FR-SC-001 |

**Main flow:**
1. User navigates to a policy page from the footer or a contextual link (e.g., a refund-eligibility question during a support flow).
2. System displays the current policy text.

**Alternate / error flows:**
- None.

**Postcondition (success):** User can make an informed decision (e.g., whether to expect a refund) before or during registration or a support interaction.

---

## B. Account Registration & Guardian Model

#### UC-03: Parent/Guardian registers (Grades 1–5 path)

| Field | Detail |
|---|---|
| Actor | Parent/Guardian |
| Precondition | None |
| Trigger | Parent submits the registration form |
| Linked FR | FR-SP-002, FR-SP-004, FR-SP-005, FR-AC-001, FR-AC-002 |

**Main flow:**
1. Parent registers via email or phone number.
2. Parent accepts the Privacy Policy, Terms & Conditions, and Rules & Regulations.
3. System verifies the email/phone via the SMS/email providers (Section 16).
4. System creates the parent account in a **pending onboarding** state — no full functionality yet.

**Alternate / error flows:**
- If verification fails or times out: system blocks account activation and allows the parent to request a new verification code.

**Postcondition (success):** Parent account exists in pending onboarding state, awaiting UC-04.

---

#### UC-04: Parent adds a student and sends an activation invite

| Field | Detail |
|---|---|
| Actor | Parent/Guardian |
| Precondition | Parent account exists in pending onboarding state (UC-03) |
| Trigger | Parent adds a student record |
| Linked FR | FR-AC-002, FR-AC-003 |

**Main flow:**
1. Parent enters the student's grade — this single field determines whether the account routes into the Grades 1–5 or Grades 6–12 model.
2. System creates a student record with status **invited** and sends the student an activation invite valid for 14 days.
3. System schedules an automatic reminder to the invited student at day 7.
4. Until the student accepts, the parent's access is restricted to invite management only (view/resend/regenerate).

**Alternate / error flows:**
- If the parent resends or regenerates the invite: the 14-day window restarts from that moment.
- If the invite expires unused: student record remains **invited**; parent can regenerate at any time (no hard expiry on the parent's ability to retry).

**Postcondition (success):** An invited student record exists, pending activation (UC-05).

---

#### UC-05: Student activates a parent-issued invite (Grades 1–5)

| Field | Detail |
|---|---|
| Actor | Student |
| Precondition | A valid, unexpired activation invite exists (UC-04) |
| Trigger | Student opens the invite link and completes account creation |
| Linked FR | FR-AC-003, FR-AC-004 |

**Main flow:**
1. Student opens the invite and completes registration (credentials, profile basics).
2. System sets the student account and the parent–student relationship to **active**.
3. System unlocks full parent functionality (booking, payment, oversight) on the parent's side.

**Alternate / error flows:**
- If the invite has expired: system informs the student the invite is no longer valid and directs them to ask their parent to resend it (UC-04).

**Postcondition (success):** Both parent and student have full, linked access appropriate to their roles.

---

#### UC-06: Student registers independently (Grades 6–12 path)

| Field | Detail |
|---|---|
| Actor | Student |
| Precondition | None |
| Trigger | Student submits the registration form |
| Linked FR | FR-SP-001, FR-SP-004, FR-SP-005, FR-SP-007, FR-AC-005 |

**Main flow:**
1. Student registers via email or phone number and accepts the Privacy Policy, Terms & Conditions, and Rules & Regulations.
2. System verifies the email/phone.
3. Student sets their own grade (6–12) directly — no parent-entry step.
4. System activates the student account immediately with no guardian required.

**Alternate / error flows:**
- If a Grade 1–5 value is somehow entered here: system routes the registration into the parent-first model (UC-03) instead, since grade is the determining field.

**Postcondition (success):** Student has an independent, fully active account and may optionally invite a guardian (UC-07) or proceed straight to profile-building (UC-13) and matching.

---

#### UC-07: Grades 6–12 student invites an optional guardian

| Field | Detail |
|---|---|
| Actor | Student |
| Precondition | Student account is active (UC-06) |
| Trigger | Student sends a guardian invite from account settings |
| Linked FR | FR-AC-005, FR-AC-006 |

**Main flow:**
1. Student sends an invite to a parent/guardian's email or phone.
2. Guardian accepts, following the same invite → active flow as UC-05.
3. System links the two accounts via a parent–student relationship record, with permissions the student/guardian agree on.

**Alternate / error flows:**
- If the guardian never accepts: the student's account and access remain fully functional and independent regardless — this invite is optional, not a gate.

**Postcondition (success):** A student-initiated guardian relationship exists, without altering the student's independent access.

---

#### UC-08: Guardian relationship is revoked or modified

| Field | Detail |
|---|---|
| Actor | Parent/Guardian, Student (Grades 6–12 only, for a relationship they initiated), or Admin |
| Precondition | An active parent–student relationship exists |
| Trigger | An authorized party revokes or modifies the relationship |
| Linked FR | FR-AC-007 |

**Main flow:**
1. A linked guardian, or Admin (for disputes/abuse), revokes or modifies a relationship's permissions.
2. For Grades 6–12, the student may also revoke a relationship they themselves initiated.
3. System updates the relationship record's status/permissions accordingly.

**Alternate / error flows:**
- A Grade 1–5 student cannot unilaterally revoke a mandatory guardian's access — any such attempt is rejected by the system, since this would leave the account without its required guardian outside the sanctioned removal path (UC-09).

**Postcondition (success):** The relationship record accurately reflects current permissions, without leaving a Grades 1–5 student without any guardian outside the controlled path in UC-09.

---

#### UC-09: Sole guardian removed from a Grades 1–5 account

| Field | Detail |
|---|---|
| Actor | Parent/Guardian (self-initiated) or Admin (abuse/dispute-initiated) |
| Precondition | The student has exactly one guardian on the account |
| Trigger | That sole guardian's relationship is removed |
| Linked FR | FR-AC-008 |

**Main flow:**
1. The sole guardian relationship is removed (self-initiated, or by Admin for abuse/dispute reasons).
2. System moves the student account into a **"guardian required" hold state**: no new bookings, no class access — but no data, progress, XP, or recordings are deleted.
3. System prompts for a new guardian invite (mirroring UC-04's invite → active flow).
4. Once a new guardian accepts, the account reactivates with all history intact.

**Alternate / error flows:**
- If no new guardian ever accepts: the account remains indefinitely in the hold state — no automatic deletion is triggered by this flow.

**Postcondition (success):** The student's access is safely paused rather than left in an undefined state, and full history is preserved for whenever a new guardian is added.

---

#### UC-10: Login, logout, and password reset

| Field | Detail |
|---|---|
| Actor | Student, Parent/Guardian, Tutor, or Admin |
| Precondition | An account exists |
| Trigger | User submits credentials, logs out, or requests a password reset |
| Linked FR | FR-SP-003 |

**Main flow:**
1. User submits credentials; system authenticates and starts a session.
2. User logs out; system ends the session.
3. User requests a password reset; system sends a reset link/code via the verified email or phone.

**Alternate / error flows:**
- Invalid credentials: system returns a generic error without revealing which field was wrong.
- Expired/used reset link: system informs the user and allows a new request.

**Postcondition (success):** User has secure, working access to their account at all times.

---

#### UC-11: Verify email or phone number

| Field | Detail |
|---|---|
| Actor | Student, Parent/Guardian, or Tutor |
| Precondition | Registration has been submitted |
| Trigger | System sends a verification code/link during registration |
| Linked FR | FR-SP-004 |

**Main flow:**
1. System sends a verification code/link via the SMS provider (Geez SMS) or email provider (Brevo), per Section 16.
2. User submits the code or clicks the link.
3. System marks the contact method as verified and unblocks account activation.

**Alternate / error flows:**
- Code expired or incorrect: system allows a resend, without permanently locking the registration attempt.

**Postcondition (success):** The account's contact method is confirmed reachable before it is relied on for reminders, payment confirmations, or emergency contact.

---

## C. Student Profile & Dashboard

#### UC-12: Build or edit a user profile

| Field | Detail |
|---|---|
| Actor | Student or Parent/Guardian |
| Precondition | Account is active |
| Trigger | User opens profile settings |
| Linked FR | FR-SP-006 |

**Main flow:**
1. User edits name, profile picture, and other basic profile fields.
2. System saves changes and reflects them immediately across the platform (dashboard, tutor-facing views where applicable).

**Alternate / error flows:**
- Invalid image format/size: system rejects the upload with a clear message, retaining the existing picture.

**Postcondition (success):** Profile information is current and accurate.

---

#### UC-13: Build the student academic profile

| Field | Detail |
|---|---|
| Actor | Student or Parent/Guardian (on the student's behalf, Grades 1–5) |
| Precondition | Student account is active |
| Trigger | User completes or edits the academic profile |
| Linked FR | FR-SP-007, FR-SP-008, FR-SP-009, FR-SP-010 |

**Main flow:**
1. User selects the student's Grade (1–12) and school.
2. User selects subjects, academic level, and learning goals.
3. User sets preferred language, learning schedule, and teaching style.
4. User indicates a budget preference and a tutoring-format preference (1-to-1, 1-to-3, or 1-to-5) — this single field determines which matching path (Section 8, Path A/B vs. Path C) the student enters.

**Alternate / error flows:**
- Required fields left blank: system blocks progression to tutor discovery until grade, subject, and format preference are set, since these are hard requirements for matching.

**Postcondition (success):** The profile contains everything the matching engine needs: grade (hard match), subject, budget (hard filter, 1-to-1 only), language (hard filter, all formats), teaching style (soft factor, 1-to-1 only), and format choice.

---

#### UC-14: View the student dashboard

| Field | Detail |
|---|---|
| Actor | Student or Parent/Guardian |
| Precondition | Student account is active |
| Trigger | User logs in or navigates to the dashboard |
| Linked FR | FR-SP-011, FR-SP-012, FR-SP-013, FR-SP-014, FR-SP-015, FR-SP-016 |

**Main flow:**
1. System displays a welcome message, upcoming lessons, and today's learning activities.
2. System displays recommended lessons (1-to-1 only, per FR-SP-020), assignments, quizzes, and recent grades.
3. System displays learning progress, XP, streaks, achievements, and leaderboard position.
4. System displays daily/weekly goals and active challenges, with access to notifications, messages, and rewards.

**Alternate / error flows:**
- No upcoming lessons scheduled yet (e.g., still in matching): dashboard shows an appropriate empty/in-progress state rather than an error.

**Postcondition (success):** The student/parent has a single, current view of everything active on their account.

---

## D. Tutor Onboarding, Verification & Subjects

#### UC-15: Tutor registers

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | None |
| Trigger | Tutor submits the registration form |
| Linked FR | FR-TU-001, FR-TU-002 |

**Main flow:**
1. Tutor registers and accepts the Privacy Policy, Terms & Conditions, and Rules & Regulations.
2. System creates the tutor account, not yet visible to students (pending verification, UC-18).

**Alternate / error flows:**
- Terms not accepted: registration cannot be finalized.

**Postcondition (success):** Tutor account exists, awaiting profile completion and verification.

---

#### UC-16: Tutor builds their profile

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | Tutor account exists (UC-15) |
| Trigger | Tutor completes the profile form |
| Linked FR | FR-TU-003 |

**Main flow:**
1. Tutor provides experience and qualification information (education, university, degree — optional field).
2. System stores the profile, still pending Admin verification.

**Alternate / error flows:**
- None beyond standard field validation.

**Postcondition (success):** A complete profile is ready for Admin review.

---

#### UC-17: Tutor selects and ranks subjects

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | Tutor account exists |
| Trigger | Tutor selects subjects during or after profile setup |
| Linked FR | FR-TU-006, FR-TU-007, FR-TU-008 |

**Main flow:**
1. Tutor ranks up to **two** subjects by preference/strength, most to least preferred.
2. System hard-caps selection at two — a third cannot be added.
3. Each ranked subject applies across the full Grade 1–12 span; no separate grade-range field exists or is required.
4. System treats the tutor's first-ranked subject as primary (always considered first in matching) and the second as a system-triggered fallback, only engaged automatically when a primary-subject search finds no match for the same grade and filters (FR-TU-008) — never Admin-initiated, never shown to students as a separate choice.

**Alternate / error flows:**
- Tutor attempts to select a third subject: system blocks the action and explains the two-subject cap.

**Postcondition (success):** Tutor is matchable on 1–2 ranked subjects, across all grades, with correct primary/secondary precedence.

---

#### UC-18: Admin verifies and approves a tutor

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Tutor has completed registration and profile (UC-15, UC-16) |
| Trigger | Admin reviews the pending tutor application |
| Linked FR | FR-TU-004, FR-AD-002 |

**Main flow:**
1. Admin reviews the tutor's profile, qualifications, and any supporting documentation.
2. Admin approves the tutor, making them visible and matchable to students.

**Alternate / error flows:**
- Admin rejects the application: tutor is notified and remains invisible to students; may resubmit with corrected information.

**Postcondition (success):** Only Admin-verified tutors ever appear in student-facing matching or search.

---

#### UC-19: Tutor sets and manages availability

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | Tutor is verified (UC-18) |
| Trigger | Tutor opens the availability/schedule management screen |
| Linked FR | FR-TU-009 |

**Main flow:**
1. Tutor sets recurring or one-off available time slots.
2. System makes these slots eligible for matching (Section 8) and for reschedule requests (Section 9.2), and reflects them on the tutor's 1-to-1 profile (FR-SP-026).

**Alternate / error flows:**
- Tutor removes availability that overlaps an already-confirmed session: system prevents removal of that specific slot until the session is resolved (completed, rescheduled, or cancelled through the proper flow).

**Postcondition (success):** The tutor's true availability is always what the matching and scheduling engines see.

---

## E. Tutor Discovery & Matching

#### UC-20: Student searches for a 1-to-1 tutor

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Academic profile is complete (UC-13); format preference is 1-to-1 |
| Trigger | Student opens "Find a Tutor" |
| Linked FR | FR-SP-017, FR-SP-018, FR-SP-019 |

**Main flow:**
1. Student searches by subject and grade.
2. Student filters by schedule/availability, budget (hard filter — tutors priced above budget excluded), language (hard filter), and any additional design-time filters (e.g., price).
3. System returns matching, Admin-verified tutors only.

**Alternate / error flows:**
- No tutors match the filters: system shows an empty state and surfaces the "No Exact Match" option (UC-25) rather than a dead end.

**Postcondition (success):** Student sees a filtered, relevant tutor list for the 1-to-1 format only — this flow does not apply to 1-to-3/1-to-5 (UC-28).

---

#### UC-21: Student views 1-to-1 tutor recommendations with match percentage

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Format preference is 1-to-1 |
| Trigger | Student opens the recommendations view (dashboard or search results) |
| Linked FR | FR-SP-020, FR-SP-021, FR-MA-001 |

**Main flow:**
1. System recommends tutors based on grade, subject, academic level, schedule, teaching-style fit (soft/scored), and tutor availability.
2. System displays a calculated match percentage (e.g., 95%, 80%) per recommended tutor.

**Alternate / error flows:**
- Zero recommendations returned: begins the 48-hour countdown toward automatic Admin escalation (UC-26) if the student takes no action.

**Postcondition (success):** Student can compare tutors on a consistent, transparent match score — 1-to-1 only; never shown for group formats.

---

#### UC-22: Student views a full tutor profile (1-to-1)

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Viewing a 1-to-1 recommendation or search result |
| Trigger | Student opens a tutor's profile |
| Linked FR | FR-SP-022, FR-SP-023, FR-SP-024, FR-SP-025, FR-SP-026, FR-SP-027 |

**Main flow:**
1. System displays tutor name, photo, verification badge, education/university/degree (optional field), subjects and grades taught, the count of **unique** students taught to date, and available time slots.
2. Student reviews the full profile alongside the match percentage before deciding.

**Alternate / error flows:**
- None — this is a read-only view; missing optional fields (e.g., degree) are simply omitted, not treated as errors.

**Postcondition (success):** Student has everything needed to make an informed 1-to-1 choice. Note: this full view never applies to 1-to-3/1-to-5, where students see only name and photo post-assignment (UC-28).

---

#### UC-23: Student selects a preferred tutor (Path A)

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | At least one suitable 1-to-1 match exists |
| Trigger | Student chooses a tutor from the recommendation list |
| Linked FR | FR-MA-002, FR-SP-028 |

**Main flow:**
1. Student selects a tutor.
2. System creates a booking request and routes it to Admin (UC-24).

**Alternate / error flows:**
- None at this step — rejection handling is covered in UC-32.

**Postcondition (success):** A pending 1-to-1 booking awaits Admin review.

---

#### UC-24: Admin reviews and approves a 1-to-1 booking (Path A)

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A student has selected a tutor (UC-23) |
| Trigger | Admin opens the pending-approvals queue |
| Linked FR | FR-MA-003, FR-MA-004, FR-AD-005, FR-AD-007 |

**Main flow:**
1. System notifies Admin of the selection.
2. Admin reviews and approves the booking.
3. System requires payment (UC-36) before confirming the schedule.

**Alternate / error flows:**
- Admin rejects instead of approving: handled by UC-32.
- Booking sits unactioned: subject to the stale-approval escalation in UC-35.

**Postcondition (success):** An approved booking proceeds to payment and schedule confirmation.

---

#### UC-25: Student manually triggers "No Exact Match" (Path B)

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | 1-to-1 search/recommendations returned no suitable tutor |
| Trigger | Student clicks "No Exact Match" |
| Linked FR | FR-MA-007, FR-SP-029 |

**Main flow:**
1. Student selects "No Exact Match."
2. System sends the request to Admin's queue for manual assignment (UC-27).

**Alternate / error flows:**
- This is also the automatic hand-off point when a tutor's secondary-subject search (FR-TU-008) also fails — no separate flow is needed for that case.

**Postcondition (success):** Admin has a manual-assignment case to work.

---

#### UC-26: Automatic escalation on a stalled 1-to-1 search

| Field | Detail |
|---|---|
| Actor | System (automatic); Admin (recipient) |
| Precondition | A 1-to-1 student's tutor search has returned zero matches continuously |
| Trigger | 48 continuous hours elapse with zero matches and no student action |
| Linked FR | FR-MA-018, FR-AD-005 |

**Main flow:**
1. System detects the continuous 48-hour zero-match condition.
2. System automatically notifies Admin and routes the case into Path B, exactly as though the student had clicked "No Exact Match."

**Alternate / error flows:**
- A match appears before the 48-hour mark: the countdown resets and no escalation fires.

**Postcondition (success):** No 1-to-1 student can remain stuck indefinitely without Admin becoming aware, even if they never notice or act themselves.

---

#### UC-27: Admin manually assigns a tutor (Path B)

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A "No Exact Match" case exists (UC-25 or UC-26) |
| Trigger | Admin opens the manual-assignment queue |
| Linked FR | FR-MA-008, FR-MA-009, FR-MA-010, FR-MA-011, FR-AD-006 |

**Main flow:**
1. Admin reviews available tutors and manually assigns a suitable one.
2. System notifies both student and tutor.
3. System requires payment (UC-36); on completion, confirms the schedule.

**Alternate / error flows:**
- No suitable tutor currently exists: Admin holds the case open until one becomes available (e.g., new tutor onboarded, or an existing tutor's availability opens up).

**Postcondition (success):** Student is matched via manual Admin assignment with the same payment/confirmation gate as any other path.

---

#### UC-28: Student requests a group format and is auto-matched (Path C)

| Field | Detail |
|---|---|
| Actor | Student/Parent; System (automatic matching) |
| Precondition | Academic profile complete (UC-13); format preference is 1-to-3 or 1-to-5 |
| Trigger | Student sets a group-format preference |
| Linked FR | FR-MA-012, FR-SP-030 |

**Main flow:**
1. Student's preference (grade, subject, schedule, language — teaching style and budget not used for group formats) enters the auto-match engine — no search or selection UI is shown.
2. System groups the student with others requesting the same subject/grade/schedule fit, up to an available, eligible tutor, within the group-formation window (UC-29).
3. Once formed, system displays only the assigned tutor's **name and photo** — no education, no total-students count, no match percentage, no time-slot picker.
4. System routes the auto-match to Admin for review (UC-31).

**Alternate / error flows:**
- Primary- and secondary-subject searches both fail: routed to UC-30 instead.

**Postcondition (success):** Student is placed into a candidate class with no student-facing match information at any point.

---

#### UC-29: Group-formation window closes with a partial group

| Field | Detail |
|---|---|
| Actor | System (automatic) |
| Precondition | A candidate 1-to-3/1-to-5 class has at least one matched student but has not reached target size |
| Trigger | The group-formation window (default 48 hours, Admin-configurable) elapses |
| Linked FR | Section 7 (Partial Group Formation, v3.0) |

**Main flow:**
1. System checks the candidate class size against the window deadline.
2. If below target size but at least one student is matched, the class proceeds at whatever size was reached.
3. Per-student pricing remains fixed at the standard FR-PR-002/003 rate regardless of final group size — the platform/tutor absorb the shortfall, not the students.

**Alternate / error flows:**
- Zero students matched by the deadline: the window simply extends/re-opens rather than forming an empty class (there is no "class of zero" state).

**Postcondition (success):** No student waits indefinitely for a full group, and no student pays more because the group formed smaller than the target size.

---

#### UC-30: Double subject-match failure for a group-format request

| Field | Detail |
|---|---|
| Actor | System (automatic); Admin (recipient) |
| Precondition | A 1-to-3/1-to-5 request's primary-subject search failed and the secondary-subject search also failed |
| Trigger | Both searches complete with no match |
| Linked FR | FR-MA-017, FR-AD-006 |

**Main flow:**
1. System detects the double failure.
2. System notifies Admin immediately for manual group assembly — there is no student-facing "No Exact Match" button for group formats (that control is 1-to-1 only, UC-25).

**Alternate / error flows:**
- None additional — this is itself the escalation/fallback path.

**Postcondition (success):** Group-format students are never left without a path forward, without ever being shown a manual-override control themselves.

---

#### UC-31: Admin reviews and approves an auto-match (Path C)

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A candidate class has formed (UC-28/UC-29) |
| Trigger | Admin opens the auto-match review queue |
| Linked FR | FR-MA-013, FR-MA-014, FR-MA-015, FR-MA-016, FR-AD-005, FR-AD-007 |

**Main flow:**
1. System notifies Admin of the auto-match.
2. Admin approves it.
3. System requires payment from each student in the class (UC-36); on completion, confirms the schedule and notifies student(s) and tutor.

**Alternate / error flows:**
- Admin rejects instead: handled by UC-32.
- Case sits unactioned: subject to stale-approval escalation (UC-35).

**Postcondition (success):** A group class proceeds to payment and schedule confirmation with the same Admin trust gate as 1-to-1.

---

#### UC-32: Admin rejects a booking or auto-match

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A pending booking (Path A) or auto-match (Path C) awaits Admin decision |
| Trigger | Admin rejects rather than approves |
| Linked FR | Section 8 (Admin Rejection Handling, v3.0), FR-AD-007 |

**Main flow:**
1. Admin rejects the case.
2. For Path A: student returns to the tutor-recommendation list (UC-21) with the rejected tutor excluded from the next set.
3. For Path C: student's request re-enters the auto-match queue (UC-28) and a new candidate class is sought.
4. System notifies the student/parent with a generic reason ("assignment could not be confirmed") — no internal rejection reason is disclosed.
5. No payment is requested until a new match is approved.

**Alternate / error flows:**
- None — this is itself the alternate flow to approval.

**Postcondition (success):** A rejected student is smoothly re-routed without being charged or given misleading internal information.

---

#### UC-33: Tutor exits an active group cohort (drop-out or suspension)

| Field | Detail |
|---|---|
| Actor | Tutor (voluntary drop-out) or Admin (suspension, see UC-34) |
| Precondition | A 1-to-3/1-to-5 class is active with an assigned tutor |
| Trigger | The tutor is removed from the cohort |
| Linked FR | Section 8 (Group Continuity on Tutor Exit, v3.0), Section 13 (refund policy) |

**Main flow:**
1. Tutor is removed (drop-out or suspension).
2. System keeps the existing student group together as a single unit and re-matches it to a new eligible tutor via Path C (UC-28), rather than dissolving and re-queuing students individually.
3. Affected students are notified and are entitled to a refund for any un-tutored paid days during the gap, per the sessions-delivered proration formula (Section 13).

**Alternate / error flows:**
- No single tutor can take the full group: falls back to the same double-fail handling as UC-30 (Admin manual assembly), splitting the group only if unavoidable.

**Postcondition (success):** A group's continuity is preserved as a cohort wherever possible, and no student is left un-tutored without a refund entitlement for that gap.

---

#### UC-34: Admin suspends an active tutor

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A verified tutor has active students |
| Trigger | Admin suspends the tutor (e.g., following FR-MK-003 escalation, or a safety/dispute finding) |
| Linked FR | FR-AD-003, Section 11.3 (Suspension of an Active Tutor, v3.0) |

**Main flow:**
1. Admin suspends the tutor.
2. Affected 1-to-1 students re-enter Path B (UC-25/UC-27 — "No Exact Match" → manual Admin assignment).
3. Affected group cohorts are kept together and re-matched per UC-33.
4. The suspension reason is not disclosed to students; Admin retains discretion to expedite re-matching for large or exam-critical cohorts.

**Alternate / error flows:**
- None additional.

**Postcondition (success):** Students are protected from service disruption without being exposed to sensitive internal disciplinary information.

---

#### UC-35: Stale booking approval is escalated

| Field | Detail |
|---|---|
| Actor | System (automatic); Admin (recipient) |
| Precondition | A booking or auto-match awaits Admin approval |
| Trigger | 48 hours elapse unactioned (flag), or 5 days elapse unactioned (escalation) |
| Linked FR | Section 11.3 (Stale Booking Approvals, v3.0), FR-AD-020 |

**Main flow:**
1. At 48 hours unactioned, system flags the case **overdue** and surfaces it at the top of Admin's review queue.
2. If still unactioned at 5 days, system proactively notifies the affected student/parent of the delay and escalates the case within the support workflow.

**Alternate / error flows:**
- No refund is automatically triggered by staleness alone — refunds still require the Section 13 policy conditions to be met independently.

**Postcondition (success):** No approval silently sits forever; the student is proactively informed if it does.

---

## F. Payment, Billing & Schedule Confirmation

#### UC-36: Complete payment after match/assignment approval

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | A booking or auto-match has been Admin-approved (any path) |
| Trigger | System presents the payment step |
| Linked FR | FR-SP-031, FR-MA-005, FR-MA-010, FR-MA-015, FR-PB-001 |

**Main flow:**
1. System requires payment completion (via Chapa, Section 16) before the schedule is confirmed, for all formats.
2. On successful payment, system confirms the tutoring schedule and notifies student and tutor.

**Alternate / error flows:**
- Payment fails or is abandoned: schedule remains unconfirmed; student can retry payment at any time without re-triggering the matching/approval steps.

**Postcondition (success):** No class is ever scheduled without payment having actually completed.

---

#### UC-37: Grades 6–12 student pays independently

| Field | Detail |
|---|---|
| Actor | Student (Grades 6–12) |
| Precondition | Student account has no linked guardian, or a guardian exists but is not required to approve payment |
| Trigger | Student completes UC-36 without guardian involvement |
| Linked FR | FR-PB-008 |

**Main flow:**
1. Student completes payment via Chapa directly, with no guardian account or guardian approval step required.

**Alternate / error flows:**
- None beyond standard payment-failure handling (UC-36).

**Postcondition (success):** A Grade 10 student with no linked guardian can complete a payment end-to-end.

---

#### UC-38: View payment history

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | At least one payment has been made |
| Trigger | User opens payment history |
| Linked FR | FR-SP-032, FR-PB-002 |

**Main flow:**
1. System displays all past payments, dates, and amounts.

**Alternate / error flows:**
- No payments yet: system shows an empty state.

**Postcondition (success):** Full payment transparency at any time.

---

#### UC-39: Receive monthly payment reminder

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Student has an active billing cycle |
| Trigger | 3 days before the student's individual billing due date |
| Linked FR | FR-PB-003, FR-PB-004 |

**Main flow:**
1. System sends a reminder with a visible countdown, anchored to that student's own billing-cycle start date (the date their first paid session was confirmed) — not a shared platform-wide date.
2. Payment is required on the due day, or the next day at the latest.

**Alternate / error flows:**
- Payment not completed by the deadline: triggers UC-40.

**Postcondition (success):** Two students who joined on different dates each get their reminder relative to their own cycle, never a shared date.

---

#### UC-40: Tutoring schedule pauses for non-payment

| Field | Detail |
|---|---|
| Actor | System (automatic) |
| Precondition | Payment due date has passed without payment (UC-39) |
| Trigger | Payment window lapses |
| Linked FR | FR-PB-005 |

**Main flow:**
1. System pauses the student's tutoring schedule.
2. Student/parent sees only a "Pay for this month" prompt until payment is completed.

**Alternate / error flows:**
- Any session that would have fallen inside this pause window is handled per UC-41, not treated as a missed session.

**Postcondition (success):** Access is paused cleanly, with a single unambiguous unblock action for the student.

---

#### UC-41: A session falls during an active payment pause

| Field | Detail |
|---|---|
| Actor | System (automatic) |
| Precondition | A session was scheduled to occur during an active payment pause (UC-40) |
| Trigger | The scheduled session time arrives while the pause is still active |
| Linked FR | FR-PB-009 |

**Main flow:**
1. System does not hold the session as scheduled.
2. Once payment completes, system automatically reschedules the session to the tutor's next available slot.
3. No make-up entry, no refund, and no billing impact are generated — this is neither a tutor-caused nor a student-caused miss.

**Alternate / error flows:**
- None — this is itself a fully defined, non-error path.

**Postcondition (success):** A payment lapse never gets mischaracterized as either party's fault.

---

## G. Class Delivery, Recording & Library

#### UC-42: Receive class reminder and countdown

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | A session is confirmed and scheduled (FR-CD-001) |
| Trigger | 1 hour before the scheduled class time |
| Linked FR | FR-SP-033, FR-CD-002, FR-NO-004 |

**Main flow:**
1. System sends both tutor and student a reminder with a countdown, 1 hour before class.

**Alternate / error flows:**
- None.

**Postcondition (success):** Neither party is caught unaware of an imminent class.

---

#### UC-43: Tutor provides the session link

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | A session is scheduled within the next 30+ minutes |
| Trigger | Tutor generates and shares the Jitsi link |
| Linked FR | FR-TU-013, FR-CD-003 |

**Main flow:**
1. Tutor generates a Jitsi session link.
2. System delivers it to the student at least 30 minutes before class start.

**Alternate / error flows:**
- Tutor fails to provide the link in time: this is a tutor-caused issue, evaluated under the missed-session rules (Section H) if the class is disrupted as a result.

**Postcondition (success):** The student always has a working session link well ahead of class time.

---

#### UC-44: Attend a live class

| Field | Detail |
|---|---|
| Actor | Student, Tutor |
| Precondition | Session link delivered (UC-43) |
| Trigger | Scheduled class time arrives |
| Linked FR | FR-CD-001 |

**Main flow:**
1. Both parties join via the Jitsi link at the fixed scheduled time.
2. Class proceeds per the tutor's teaching plan.

**Alternate / error flows:**
- One party fails to join: evaluated under Section H (tutor-caused or student-caused miss).

**Postcondition (success):** The class is delivered as scheduled.

---

#### UC-45: Tutor records the session with consent

| Field | Detail |
|---|---|
| Actor | Tutor; Student/Parent (consent) |
| Precondition | This is the first recorded session for this specific tutor–student pairing, or consent is already on file |
| Trigger | Class begins |
| Linked FR | FR-TU-014, FR-SC-008, FR-SC-009, FR-CD-004 |

**Main flow:**
1. Before the first recorded session for a given pairing, system has already obtained a one-time consent acknowledgment (at booking) from student/parent and tutor.
2. During the session, system displays a persistent recording-in-progress indicator to all participants.
3. Tutor records the session per platform rules and uploads it immediately after class.

**Alternate / error flows:**
- Consent was never obtained (e.g., a data gap): system blocks the session from being marked as recorded until consent is captured, protecting against a compliance gap.

**Postcondition (success):** Every recorded session has verifiable, prior consent and a visible in-session indicator throughout.

---

#### UC-46: Recording is automatically routed to the Library

| Field | Detail |
|---|---|
| Actor | System (automatic) |
| Precondition | A recording has been uploaded (UC-45) |
| Trigger | Upload completes |
| Linked FR | FR-CD-005, FR-TU-015 |

**Main flow:**
1. System stores the recording in cloud object storage (Cloudflare R2), with metadata only (student, tutor, session, storage key, file size, created/expiration date) in PostgreSQL.
2. System routes it automatically to the correct student's personal Library entry — no manual filing step by the tutor.
3. Recording is encoded at 720p, compressed.

**Alternate / error flows:**
- Upload fails or is delayed: handled by UC-48.

**Postcondition (success):** The right student always finds their recording in their own Library with no manual intervention.

---

#### UC-47: Student accesses their own recordings and materials

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | At least one recording or material exists for this student |
| Trigger | Student opens the Library |
| Linked FR | FR-SP-035, FR-SP-036, FR-SP-037, FR-CD-006, FR-CD-007, FR-CD-009 |

**Main flow:**
1. System displays the student's own class recordings via signed, temporary URLs, plus tutor-provided PDFs, books, and notes.
2. Student cannot access any other student's recording, even via a guessed or shared URL.

**Alternate / error flows:**
- Recording past its 90-day default retention and not marked "Keep permanently": no longer accessible — treated as an expected expiry, not an error.

**Postcondition (success):** Access is always scoped strictly to the student's own materials.

---

#### UC-48: Failed or missing recording upload is escalated

| Field | Detail |
|---|---|
| Actor | System (automatic); Tutor; Admin |
| Precondition | A session has ended | 
| Trigger | Recording is not received within 2 hours of the session's scheduled end |
| Linked FR | Section 9.1 (Failed Recording Upload, v3.0), FR-TU-014, FR-AD-003 |

**Main flow:**
1. System flags the session "recording missing" on both the tutor's and Admin's dashboards, and sends the tutor a retry prompt.
2. Tutor has 24 hours to upload before the session is escalated to Admin as a compliance issue.

**Alternate / error flows:**
- A missing recording does not, by itself, trigger a refund or make-up — the class itself still occurred — unless the student separately reports the class as not delivered, in which case Section H applies.

**Postcondition (success):** Recording-compliance gaps surface quickly to Admin without penalizing the student unless the class truly didn't happen.

---

#### UC-49: Student marks a recording "Keep permanently"

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Recording exists in the Library, within its retention window |
| Trigger | Student selects "Keep permanently" on a recording |
| Linked FR | FR-SP-037 |

**Main flow:**
1. Student marks the recording to bypass the default 90-day auto-deletion.
2. System retains it until the student or platform deletes it explicitly.

**Alternate / error flows:**
- None.

**Postcondition (success):** A student can preserve recordings they value beyond the default window.

---

## H. Make-up, Reschedule & Cancellation

#### UC-50: Tutor-caused missed session triggers a free make-up

| Field | Detail |
|---|---|
| Actor | System (automatic); Tutor; Student |
| Precondition | A tutor causes a missed session (no-show, late cancellation inside 12 hours, technical failure on the tutor's side) |
| Trigger | The miss is recorded |
| Linked FR | FR-MK-001, FR-MK-009 |

**Main flow:**
1. System automatically queues a mandatory free make-up session for the student within 7 days.
2. No additional charge and no refund/credit is issued for that specific session.
3. Tutor is paid a flat 50% of their normal per-session share when they deliver the make-up.

**Alternate / error flows:**
- This is the second tutor-caused miss within a rolling 30 days for this tutor: also triggers UC-52.

**Postcondition (success):** The student is made whole via a scheduled make-up with no manual request needed, and tutor pay reflects the reduced rate correctly.

---

#### UC-51: Student-caused missed session

| Field | Detail |
|---|---|
| Actor | System (automatic); Student |
| Precondition | A student causes a missed session (no-show, late cancellation inside 12 hours) |
| Trigger | The miss is recorded |
| Linked FR | FR-MK-002 |

**Main flow:**
1. System logs the session as delivered against that month's billed sessions.
2. No make-up and no refund/credit is provided.
3. Tutor is paid their normal full share, since the tutor was available and ready to teach.

**Alternate / error flows:**
- None — this does not, by itself, generate a support ticket.

**Postcondition (success):** The distinction between who caused the miss is applied consistently and automatically.

---

#### UC-52: Tutor is escalated after repeated misses

| Field | Detail |
|---|---|
| Actor | System (automatic); Admin |
| Precondition | A tutor has accumulated two or more tutor-caused misses within a rolling 30-day period |
| Trigger | The second qualifying miss occurs |
| Linked FR | FR-MK-003, FR-AD-003 |

**Main flow:**
1. System escalates the tutor to Admin's mandatory review queue.
2. Admin reviews and may suspend or restrict the tutor's account (UC-34) as warranted.

**Alternate / error flows:**
- None.

**Postcondition (success):** Repeated tutor-caused disruption is surfaced to Admin without requiring Admin to search for it.

---

#### UC-53: Either party requests a reschedule

| Field | Detail |
|---|---|
| Actor | Student/Parent or Tutor |
| Precondition | The request is made ≥12 hours before the affected session |
| Trigger | Either party initiates a reschedule request |
| Linked FR | FR-MK-004, FR-MK-007, FR-MK-008 |

**Main flow:**
1. Requesting party proposes a new time within the tutor's existing availability.
2. System processes the reschedule at no charge, with no make-up/refund implication and no billing impact.
3. System counts this against the student's cap of 2 free reschedules per month.

**Alternate / error flows:**
- The student has already used 2 free reschedules this month: further requests require Admin discretion.
- Request is made inside the 12-hour window: routed instead to UC-54.

**Postcondition (success):** Legitimate schedule shifts happen smoothly without being misclassified as misses, up to the monthly cap.

---

#### UC-54: Reschedule requested inside the 12-hour window

| Field | Detail |
|---|---|
| Actor | Student/Parent or Tutor |
| Precondition | A reschedule request is made less than 12 hours before the session |
| Trigger | Request submitted |
| Linked FR | FR-MK-006 |

**Main flow:**
1. System treats this as a same-day miss rather than a free reschedule.
2. The tiered tutor-caused/student-caused rules (UC-50/UC-51) apply based on who requested it.

**Alternate / error flows:**
- None — this is itself the alternate path to UC-53.

**Postcondition (success):** Last-minute changes are handled with the correct financial and make-up consequences, not the free-reschedule path.

---

#### UC-55: Student attempts to cancel a confirmed session

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | A session is confirmed |
| Trigger | Student attempts to cancel outright |
| Linked FR | FR-MK-005 |

**Main flow:**
1. System does not permit a student-initiated outright cancellation (that control is restricted to tutor or Admin).
2. Instead, the request is processed under the no-show/make-up rules (UC-50/UC-51), based on how much notice was given.

**Alternate / error flows:**
- None — this redirection is itself the defined behavior.

**Postcondition (success):** Cancellation-shaped requests from students always resolve to a well-defined miss classification, never an undefined "cancelled" state.

---

#### UC-56: Tutor is paid the reduced rate for a self-caused make-up

| Field | Detail |
|---|---|
| Actor | System (automatic); Tutor |
| Precondition | A make-up session is delivered as a direct result of the tutor's own earlier miss (UC-50) |
| Trigger | The make-up session is completed |
| Linked FR | FR-MK-009, FR-TU-018, FR-TU-019 |

**Main flow:**
1. System calculates the tutor's earnings for that specific session at a flat 50% of their normal per-session share.
2. This reduced-rate session is itemized distinctly from full-rate sessions in the tutor's earnings view and the monthly automatic payout.

**Alternate / error flows:**
- The make-up session resulted from a student-caused miss instead: tutor is paid full share (UC-51), not the reduced rate.

**Postcondition (success):** Tutor pay always correctly reflects fault, and the tutor can see exactly which sessions were paid at the reduced rate.

---

## I. In-Platform Messaging

#### UC-57: Student/parent messages their 1-to-1 tutor

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | A 1-to-1 match is confirmed and paid |
| Trigger | Either party opens the messaging thread |
| Linked FR | FR-MS-001, FR-MS-002, FR-TU-024 |

**Main flow:**
1. Student/parent and tutor exchange text-only messages in a private pair thread.
2. Thread is accessible only to this specific matched pair.

**Alternate / error flows:**
- Match is not yet confirmed/paid, or has since ended: messaging is blocked for that pairing.

**Postcondition (success):** Coordination (e.g., "running 5 minutes late") happens safely in-platform, with no outside-platform contact needed.

---

#### UC-58: Cohort messaging for 1-to-3/1-to-5

| Field | Detail |
|---|---|
| Actor | Student(s), Tutor |
| Precondition | A group match is confirmed and paid |
| Trigger | Any cohort member or the tutor opens the thread |
| Linked FR | FR-MS-001 |

**Main flow:**
1. Tutor and all currently assigned students in the cohort share a single thread.
2. Any message posted is visible to every current member of that cohort — no private sub-threads exist within a group.

**Alternate / error flows:**
- A student leaves the cohort (e.g., via format switch, UC-61): they lose access to the thread going forward, subject to the archive rule in UC-59.

**Postcondition (success):** Group coordination happens in one shared, transparent space, consistent with the no-private-thread rule for groups.

---

#### UC-59: Message history is archived after assignment ends

| Field | Detail |
|---|---|
| Actor | System (automatic) |
| Precondition | A matched assignment (pair or cohort) has ended |
| Trigger | 90 days elapse after the assignment's end date |
| Linked FR | FR-MS-003 |

**Main flow:**
1. System retains message history for the duration of the active assignment plus 90 days.
2. After that window, history is archived out of the active thread view.

**Alternate / error flows:**
- None — archiving does not mean permanent deletion; access constraints follow the same access-control rules as when the thread was active.

**Postcondition (success):** History remains available for a reasonable dispute window without cluttering active views indefinitely.

---

#### UC-60: Admin reviews or closes a flagged message thread

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A thread has been reported, or Admin is investigating a dispute |
| Trigger | Admin opens the flagged-thread view |
| Linked FR | FR-MS-004, FR-AD-017 |

**Main flow:**
1. Admin views the relevant message thread for investigation.
2. Admin may close/report the thread if it violates platform rules.

**Alternate / error flows:**
- None.

**Postcondition (success):** Admin can resolve disputes with full context, without students/tutors needing to forward evidence manually.

---

## J. Format Switching

#### UC-61: Student requests a tutoring-format switch

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Student currently has an active, matched assignment (any format) |
| Trigger | Student requests a format change (e.g., 1-to-1 → 1-to-3) |
| Linked FR | FR-SP-045, FR-SP-046, FR-SP-047, FR-SP-048, FR-SP-049 |

**Main flow:**
1. Student requests the switch.
2. System immediately cancels the current match and any remaining scheduled sessions under the old format.
3. System routes the student into matching under the newly selected format via the applicable path (Path A/B for 1-to-1, Path C for group formats), starting from format-preference capture (UC-13).
4. System prorates and refunds any remaining paid sessions in the current billing cycle, using the sessions-delivered formula (Section 13).
5. Outgoing tutor is notified the assignment ended due to a student-initiated format switch, not a rating/performance issue; student is notified once the new match is confirmed.

**Alternate / error flows:**
- The student was part of a group cohort: the rest of that cohort is left intact and unaffected by this student's switch — this is distinct from the group-continuity rule that applies when a tutor exits (UC-33).

**Postcondition (success):** A student can move formats with no manual Admin step required to initiate it, with a correct refund and no disruption to former cohort-mates.

---

## K. Progress, Feedback & Gamification

#### UC-62: View weekly assessment results and tutor feedback

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | A tutor has submitted a weekly assessment |
| Trigger | Student opens progress/assessment view |
| Linked FR | FR-SP-038, FR-GA-001 |

**Main flow:**
1. System displays assessment results and any tutor feedback for the period.

**Alternate / error flows:**
- No assessment submitted yet for the current week: system shows a pending state.

**Postcondition (success):** Student/parent can track academic progress over time.

---

#### UC-63: Earn XP, badges, and streaks

| Field | Detail |
|---|---|
| Actor | Student; System (automatic) |
| Precondition | Student engages in qualifying learning activity |
| Trigger | A qualifying event occurs (class attended, assessment completed, consistent activity, etc.) |
| Linked FR | FR-SP-039, FR-GA-003, FR-GA-006 |

**Main flow:**
1. System awards XP/points, achievements, and badges, and updates streak counters.
2. System recognizes improvement, consistency, and academic effort specifically, not just raw activity volume.

**Alternate / error flows:**
- A streak is broken by an inactive period: system resets the streak counter per platform rules, without deleting previously earned badges/XP.

**Postcondition (success):** Engagement is meaningfully and fairly recognized over time.

---

#### UC-64: View the per-grade leaderboard

| Field | Detail |
|---|---|
| Actor | Student/Parent |
| Precondition | Student has XP or ranking activity |
| Trigger | Student opens the leaderboard |
| Linked FR | FR-GA-002 |

**Main flow:**
1. System displays weekly and monthly rankings scoped strictly to the student's own grade level.
2. Each entry shows first name and last-initial only — never a full last name.

**Alternate / error flows:**
- None — a Grade 3 student's leaderboard never includes Grade 10 students, by design, not as an edge case.

**Postcondition (success):** Leaderboards motivate without exposing full student identity.

---

#### UC-65: View and participate in learning challenges

| Field | Detail |
|---|---|
| Actor | Student |
| Precondition | Weekly/monthly challenges are active |
| Trigger | Student opens the challenges view |
| Linked FR | FR-SP-040, FR-GA-004 |

**Main flow:**
1. System presents active weekly/monthly challenges and milestones.
2. Student participates via normal platform activity; system tracks progress toward each challenge.

**Alternate / error flows:**
- None.

**Postcondition (success):** Ongoing engagement is incentivized through fresh, time-boxed goals.

---

## L. Support, Safety & Account Management

#### UC-66: Submit a complaint, claim, or report

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | Account is active |
| Trigger | User opens the complaint/report system |
| Linked FR | FR-SP-042, FR-TU-022, FR-SC-003 |

**Main flow:**
1. User submits details of the complaint/claim/report.
2. System routes it to Admin's dispute-management queue (UC-88 in Section Q).

**Alternate / error flows:**
- Required details missing: system prompts for the minimum needed (e.g., which session/thread it relates to).

**Postcondition (success):** Every complaint reaches Admin with enough context to act on.

---

#### UC-67: Contact customer support

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | Account is active |
| Trigger | User opens the support contact channel |
| Linked FR | FR-SP-043, FR-TU-023 |

**Main flow:**
1. User reaches AKEWTutor support through the provided channel.

**Alternate / error flows:**
- None.

**Postcondition (success):** Users always have a way to reach a human for issues outside the automated flows.

---

#### UC-68: Manage account and privacy settings

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | Account is active |
| Trigger | User opens account/privacy settings |
| Linked FR | FR-SP-044 |

**Main flow:**
1. User views/updates account and privacy preferences.

**Alternate / error flows:**
- None.

**Postcondition (success):** User retains control over their own account configuration.

---

#### UC-69: Use the emergency contact channel

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | An urgent, time-sensitive issue arises during or around a session |
| Trigger | User contacts Admin via phone or Telegram |
| Linked FR | FR-PB-006 |

**Main flow:**
1. User reaches Admin directly via the published phone number or Telegram channel.
2. Admin responds manually — this is not an automated flow.

**Alternate / error flows:**
- None — no automated confidentiality or response-time guarantee is implied by this channel; handling is manual by design.

**Postcondition (success):** A direct, low-latency human channel exists for genuine emergencies.

---

## M. Tutor Ongoing Experience & Earnings

#### UC-70: Tutor views earnings and payout

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | Tutor has delivered at least one billable session |
| Trigger | Tutor opens the earnings view |
| Linked FR | FR-TU-018, FR-TU-019 |

**Main flow:**
1. System displays earnings and payment history, including the upcoming automatic monthly payout date and amount.
2. Any sessions paid at the reduced make-up rate (UC-56) are itemized distinctly from full-rate sessions.

**Alternate / error flows:**
- None — payout requires no manual "request payout" action anywhere in the tutor UI.

**Postcondition (success):** Tutor always has full, itemized visibility into pay, with payout happening automatically.

---

#### UC-71: Tutor submits feedback and weekly assessments

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | A class has been delivered |
| Trigger | Tutor opens the feedback/assessment form for a student |
| Linked FR | FR-TU-017 |

**Main flow:**
1. Tutor submits feedback and a weekly assessment for the student.
2. System makes this visible to the student/parent (UC-62) and factors it into progress tracking.

**Alternate / error flows:**
- None.

**Postcondition (success):** Students receive regular, structured feedback on their progress.

---

#### UC-72: Tutor uploads learning materials

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | Tutor has active students |
| Trigger | Tutor uploads a PDF, note, or book |
| Linked FR | FR-TU-016, FR-CD-009 |

**Main flow:**
1. Tutor uploads the material.
2. System stores it in the Library, accessible to the relevant student(s).

**Alternate / error flows:**
- Unsupported file type: system rejects the upload with a clear message.

**Postcondition (success):** Students have supplementary material alongside their recordings.

---

#### UC-73: Tutor reports a problem or contacts support

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | Account is active |
| Trigger | Tutor opens the problem-report or support channel |
| Linked FR | FR-TU-022, FR-TU-023 |

**Main flow:**
1. Tutor submits a problem report or reaches support directly.
2. System routes it appropriately (Admin dispute queue, or general support).

**Alternate / error flows:**
- None.

**Postcondition (success):** Tutors have the same baseline support access as students.

---

#### UC-74: Tutor receives platform notifications

| Field | Detail |
|---|---|
| Actor | Tutor |
| Precondition | A relevant event occurs (matching request, booking confirmation, class reminder, message, etc.) |
| Trigger | The triggering event fires |
| Linked FR | FR-TU-010, FR-TU-021, FR-TU-024, FR-NO-002 through FR-NO-011 |

**Main flow:**
1. System delivers the relevant notification through the tutor's preferred channel (push/SMS/email).

**Alternate / error flows:**
- None — see Section R for the consolidated notification-event list.

**Postcondition (success):** Tutors stay current on everything requiring their attention without needing to poll the dashboard.

---

## N. Admin Authentication & People Management

#### UC-75: Admin logs into the admin console

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin has valid credentials |
| Trigger | Admin submits credentials on the admin login screen |
| Linked FR | Implied prerequisite for Section 11 — no explicit FR-xxx ID in the SRS; carried forward here as a foundational use case since every Admin flow in Sections N–Q depends on it |

**Main flow:**
1. Admin submits credentials.
2. System authenticates and grants access to the protected admin console.

**Alternate / error flows:**
- Invalid credentials: generic error, no field-specific detail revealed.
- Session inactive beyond timeout: re-authentication required.

**Postcondition (success):** Admin has secure access to the full operational console.

---

#### UC-76: Admin manages people and relationship records

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated (UC-75) |
| Trigger | Admin opens the people-management screen |
| Linked FR | FR-AD-001 |

**Main flow:**
1. Admin views/edits student, parent, and tutor accounts, including parent–student relationship records.

**Alternate / error flows:**
- None beyond standard validation.

**Postcondition (success):** Admin has a single place to manage every account and relationship on the platform.

---

#### UC-77: Admin manages tutor badges

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated; tutor has performance/achievement data |
| Trigger | Admin opens tutor badge management |
| Linked FR | FR-AD-004, FR-GA-005 |

**Main flow:**
1. Admin reviews or adjusts a tutor's badges, computed strictly from experience/performance/achievement fields — never from a rating field, since none exists.

**Alternate / error flows:**
- None.

**Postcondition (success):** Badge integrity is maintained without any rating-based mechanism ever being reintroduced.

---

#### UC-78: Admin suspends or restricts an account

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated; grounds for suspension exist (rule violation, escalation, dispute) |
| Trigger | Admin initiates a suspension/restriction |
| Linked FR | FR-AD-003, FR-SC-007 |

**Main flow:**
1. Admin suspends or restricts the account.
2. If the account is a tutor with active students, this also triggers UC-34.

**Alternate / error flows:**
- None.

**Postcondition (success):** Rule violations and safety issues are actionable without needing a separate mechanism for tutors vs. other roles.

---

#### UC-79: Admin manages subjects and grade levels

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens subject/grade configuration |
| Linked FR | FR-AD-013, NFR-011 |

**Main flow:**
1. Admin adds/edits subjects and grade levels available platform-wide.
2. System reflects the change immediately in matching and profile-setup screens.

**Alternate / error flows:**
- None.

**Postcondition (success):** New subjects/grades can be added without structural rework, per the scalability NFR.

---

## O. Admin Money Management

#### UC-80: Admin configures pricing and revenue splits

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens pricing configuration |
| Linked FR | FR-AD-009, FR-PR-004 |

**Main flow:**
1. Admin updates the price/hr, total/hr, and revenue-split figures for any of the three formats.
2. System applies the change to the next new booking, with no code deployment required.

**Alternate / error flows:**
- None.

**Postcondition (success):** Admin can change 1-to-1 pricing and see it reflected immediately on the next booking.

---

#### UC-81: Admin manages payments and platform revenue

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens payments/revenue reporting |
| Linked FR | FR-AD-010 |

**Main flow:**
1. Admin views and manages payment records and platform revenue.

**Alternate / error flows:**
- None.

**Postcondition (success):** Full financial visibility for platform operations.

---

#### UC-82: Admin manages tutor earnings and payouts

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens the payout-management screen |
| Linked FR | FR-AD-011 |

**Main flow:**
1. Admin reviews and manages tutor earnings, including sessions paid at the reduced make-up rate (UC-56).
2. Monthly payouts proceed automatically per FR-TU-019, with Admin oversight rather than manual triggering.

**Alternate / error flows:**
- A payout dispute arises: Admin can adjust and re-process for the affected tutor.

**Postcondition (success):** Admin retains full oversight of the automatic payout process.

---

#### UC-83: Admin reviews and processes a refund

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A refund-eligible event exists (tutor drop-out, platform outage, format switch) |
| Trigger | Admin opens the refund-review queue |
| Linked FR | FR-AD-012, FR-PB-007 |

**Main flow:**
1. Admin reviews the case against the refund policy (Section 03) and the sessions-delivered proration formula (Section 13).
2. Admin approves the refund; system processes it.

**Alternate / error flows:**
- Case does not meet the policy conditions (e.g., student-caused miss): refund is denied, with the reason recorded.

**Postcondition (success):** Refunds are always proportional to sessions actually undelivered, and always Admin-reviewed before processing.

---

## P. Admin Content & Platform Management

#### UC-84: Admin manages Library and recording access

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens Library/recording administration |
| Linked FR | FR-AD-014 |

**Main flow:**
1. Admin manages retention rules, access controls, and reviews sessions flagged "recording missing" (UC-48).

**Alternate / error flows:**
- None.

**Postcondition (success):** Admin has full operational control over the Library, beyond what students/tutors can self-manage.

---

#### UC-85: Admin manages platform policies

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin edits the Privacy Policy, Terms, or Rules & Regulations |
| Linked FR | FR-AD-015 |

**Main flow:**
1. Admin updates policy content.
2. System reflects the update on the relevant public pages (UC-02) immediately.

**Alternate / error flows:**
- None.

**Postcondition (success):** Policy content stays current without a code deployment.

---

#### UC-86: Admin manages promotional offers and discounts

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens promotions management |
| Linked FR | FR-AD-016 |

**Main flow:**
1. Admin creates/edits/removes promotional offers or discounts.
2. System surfaces active promotions on the public site (UC-01) and applies eligible discounts at payment (UC-36).

**Alternate / error flows:**
- None.

**Postcondition (success):** Marketing promotions can be run and adjusted without engineering involvement.

---

## Q. Admin Oversight & Reporting

#### UC-87: Admin manages complaints and disputes

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | A complaint/report exists (UC-66) |
| Trigger | Admin opens the dispute-management queue |
| Linked FR | FR-AD-017 |

**Main flow:**
1. Admin reviews the complaint, including any flagged message threads (UC-60).
2. Admin resolves it — which may include a refund (UC-83), suspension (UC-78), or re-matching action.

**Alternate / error flows:**
- None.

**Postcondition (success):** Every complaint reaches a documented resolution.

---

#### UC-88: Admin manages leaderboards and achievements

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens leaderboard/achievement administration |
| Linked FR | FR-AD-018 |

**Main flow:**
1. Admin monitors and, where necessary, corrects leaderboard/achievement data (e.g., resolving a scoring dispute).

**Alternate / error flows:**
- None.

**Postcondition (success):** Gamification data integrity is maintained centrally.

---

#### UC-89: Admin manages notifications and announcements

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens notification/announcement management |
| Linked FR | FR-AD-019 |

**Main flow:**
1. Admin composes and sends a platform-wide announcement, or adjusts notification rules.

**Alternate / error flows:**
- None.

**Postcondition (success):** Admin can communicate platform-wide changes directly.

---

#### UC-90: Admin uses the stale-approval and support-management tools

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens the support-management dashboard |
| Linked FR | FR-AD-020 |

**Main flow:**
1. Admin works the stale-approval escalation workflow (UC-35) alongside general support-ticket management.

**Alternate / error flows:**
- None.

**Postcondition (success):** Operational and support workflows live in one place for Admin.

---

#### UC-91: Admin views platform statistics and reports

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated |
| Trigger | Admin opens the reporting dashboard |
| Linked FR | FR-AD-021, FR-AD-022 |

**Main flow:**
1. Admin views platform statistics, activity history, and tutor performance/badge history for quality-monitoring purposes.

**Alternate / error flows:**
- None.

**Postcondition (success):** Admin has the operational visibility needed to run the platform day to day.

---

## R. Notifications (Cross-Cutting)

#### UC-92: User receives a lifecycle notification

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | A qualifying platform event occurs |
| Trigger | Registration completion, tutor approval, matching/auto-match, booking confirmation, Admin approval, payment confirmation, class reminder, cancellation/reschedule/make-up scheduling, assessment availability, new recording/material, achievement/badge/leaderboard change, payment/earnings update, or claim/support update |
| Linked FR | FR-NO-001 through FR-NO-010 |

**Main flow:**
1. The qualifying event fires.
2. System delivers the corresponding notification via the user's preferred channel (push/SMS/email).

**Alternate / error flows:**
- Delivery channel fails (e.g., SMS provider outage): system retries or falls back to an alternate verified channel where configured.

**Postcondition (success):** Every FR-MK make-up/reschedule event, not just original bookings, reliably produces a notification.

---

#### UC-93: User receives a new-message notification

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | A message thread they're part of receives a new message (UC-57/UC-58) |
| Trigger | Message is sent |
| Linked FR | FR-NO-011 |

**Main flow:**
1. System notifies the recipient(s) through the same notification pipeline used for other in-app alerts.

**Alternate / error flows:**
- None.

**Postcondition (success):** Message replies are never missed simply because the recipient wasn't actively viewing the thread.

---

## S. Cross-Cutting Non-Functional Behavior

#### UC-94: Use the platform on a mobile device

| Field | Detail |
|---|---|
| Actor | Student/Parent, Tutor |
| Precondition | User is on a mobile screen size/touch device |
| Trigger | User performs any student/parent/tutor-facing flow above on a mobile device |
| Linked FR | NFR-001, NFR-002 |

**Main flow:**
1. User loads any student/parent/tutor page on a mobile browser.
2. System renders a fully responsive layout — dashboard, messaging, Library, matching, and all other flows above remain usable via touch.

**Alternate / error flows:**
- None — this is a rendering constraint applied across all student/parent/tutor use cases, not a distinct error-prone flow of its own. Unlike the Admin console (desktop-oriented by nature of its data-dense workflows), every student/parent/tutor use case in Sections A–M is expected to work on mobile.

**Postcondition (success):** No student/parent/tutor-facing functionality is mobile-degraded relative to desktop.

---

#### UC-95: Platform remains responsive under peak load

| Field | Detail |
|---|---|
| Actor | All roles (system-level concern) |
| Precondition | Peak class hours are underway |
| Trigger | Concurrent usage from registered students and tutors during peak hours |
| Linked FR | NFR-003, NFR-004, NFR-005, NFR-006 |

**Main flow:**
1. Core pages (homepage, dashboard, tutor search) load within 3 seconds on standard broadband.
2. The system supports concurrent use by all registered students/tutors during peak hours without degraded performance.
3. Platform uptime targets 99.5% or higher, excluding scheduled maintenance windows, which avoid peak tutoring hours where possible.

**Alternate / error flows:**
- A scheduled maintenance window must occur during a period that overlaps peak hours: Admin communicates this in advance via UC-89.

**Postcondition (success):** Every use case above remains usable at the actual volumes the platform is built to support.

---

**Next:** proceed to → [04. Database & Data Model]
