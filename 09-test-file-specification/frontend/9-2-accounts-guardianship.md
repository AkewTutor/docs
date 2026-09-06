## Project: AKEWTutor — Frontend Test Documentation: Accounts & Guardianship
**Links back to:** [05b §2], [07-02], [8-2. Frontend Function-Level Spec: Accounts & Guardianship]
**Conventions:** see `9-1-shared-config.md` for the shared Vitest/RTL/QueryClient setup, mock patterns, and OWASP category definitions reused throughout this doc.

**Depends on:** Shared Config (hard — every page/guard/hook here assumes `9-1`'s foundation is already tested and working).

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| hooks/useStudentProfile.ts (useMyStudentProfile, useUpdateProfile, useUpdateAcademicProfile) | FR-SP-006–010 | NFR-009 (parent/student scoping) |
| hooks/useGuardianship.ts | FR-AC-002–008, FR-SP-001 (invite path) | NFR-009 |
| hooks/useTutorProfile.ts | FR-TU-003, FR-TU-006 | — |
| hooks/useAvailability.ts | FR-TU-009 | — |
| hooks/useSubjects.ts | FR-AD-013, NFR-011 (extensible catalog) | — |
| hooks/useAdminTutorVerification.ts | FR-TU-004, FR-AD-002 | NFR-009 |
| hooks/useAdminPeople.ts | FR-AD-001, FR-AD-003, FR-AD-004 | NFR-009, NFR-010 |
| pages/InviteActivationPage.tsx | FR-AC-002 (invite path) | NFR-008 |
| components/SubjectRankingForm.tsx | FR-TU-006 (two-subject cap) | — |
| components/AvailabilityCalendar.tsx | FR-TU-009 | — |
| components/GuardianInviteStatusCard.tsx | FR-AC-003–004 | — |
| pages/admin/TutorVerificationPage.tsx, PeopleManagementPage.tsx | FR-TU-004, FR-AD-001–004 | NFR-009 |

---

### 9.1 Test File Map

| Source file | Test file | Notes |
|---|---|---|
| src/hooks/useStudentProfile.ts | tests/hooks/useStudentProfile.test.ts | mandatory (hook rule) |
| src/hooks/useGuardianship.ts | tests/hooks/useGuardianship.test.ts | mandatory — full blocks below |
| src/hooks/useTutorProfile.ts | tests/hooks/useTutorProfile.test.ts | mandatory |
| src/hooks/useAvailability.ts | tests/hooks/useAvailability.test.ts | mandatory |
| src/hooks/useSubjects.ts | tests/hooks/useSubjects.test.ts | mandatory |
| src/hooks/useAdminTutorVerification.ts | tests/hooks/useAdminTutorVerification.test.ts | mandatory |
| src/hooks/useAdminPeople.ts | tests/hooks/useAdminPeople.test.ts | mandatory |
| src/pages/InviteActivationPage.tsx | tests/pages/InviteActivationPage.test.tsx | non-trivial: password-match check, role-dependent redirect |
| src/pages/parent/AddStudentPage.tsx | tests/pages/AddStudentPage.test.tsx | non-trivial: grade-conditional confirmation copy |
| src/pages/parent/GuardianSettingsPage.tsx, student/GuardianSettingsPage.tsx | tests/pages/GuardianSettingsPage.test.tsx | non-trivial: dual-mode rendering, revoke confirmation gate |
| src/components/GuardianInviteStatusCard.tsx | tests/components/GuardianInviteStatusCard.test.tsx | non-trivial: conditional "Resend" visibility |
| src/components/SubjectRankingForm.tsx | tests/components/SubjectRankingForm.test.tsx | non-trivial: live 2-item cap enforcement (8-2 flags this explicitly) |
| src/components/AvailabilityCalendar.tsx | tests/components/AvailabilityCalendar.test.tsx | non-trivial: grid interaction, occupied-cell disable |
| src/pages/admin/TutorVerificationPage.tsx, components/TutorVerificationCard.tsx | tests/pages/TutorVerificationPage.test.tsx | non-trivial: required-reason gate on reject |
| src/pages/admin/PeopleManagementPage.tsx | tests/pages/PeopleManagementPage.test.tsx | non-trivial: required-reason gate on suspend |
| src/pages/admin/SubjectManagementPage.tsx | tests/pages/SubjectManagementPage.test.tsx | non-trivial: shows both active/inactive, flagged hook addition |
| src/pages/student/AcademicProfilePage.tsx | tests/pages/AcademicProfilePage.test.tsx | non-trivial: two independently-submitted forms |
| src/pages/tutor/TutorProfilePage.tsx | tests/pages/TutorProfilePage.test.tsx | non-trivial: `verificationStatus` is read-only, never editable |
| src/pages/tutor/AvailabilityPage.tsx, SubjectRankingPage.tsx | — | thin page wrappers around the components above; covered by the component tests + one smoke test confirming wiring, not separately detailed |
| src/components/StudentSidebar.tsx, TutorSidebar.tsx, ParentSidebar.tsx, AdminSidebar.tsx | tests/components/Sidebars.test.tsx | non-trivial: active-item highlighting per role + logout wiring (grouped into one file, see §9.6) |

---

### 9.2 Test Case Detail — useGuardianship.test.ts (full blocks)

**OWASP: A01:2021 – Broken Access Control.**

#### useAddStudent

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success invalidates `[RELATIONSHIPS]` | mock `POST /guardianship/students` success | `mutate({ grade, studentName })` | `[RELATIONSHIPS]` query key invalidated |
| Full grade range 1–12 accepted client-side | mock success | `mutate({ grade: 1, ... })` and `mutate({ grade: 12, ... })` | both succeed — confirms this hook is **not** artificially restricted to 6–12 the way `RegisterPage`'s student mode is (8-2's explicit distinction between the two grade ranges across two different entry points); a copy-paste error importing `RegisterPage`'s validation here would silently break Grade 1–5 enrollment, so this case exists specifically to catch that class of regression |
| Grade outside 1–12 rejected client-side | attempt `mutate({ grade: 13, ... })` / `mutate({ grade: 0, ... })` | — | rejected before any request fires |

#### useActivateInvite

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Success calls `authStore.setAuth` directly | mock `POST /guardianship/invites/:token/activate` → `{ accessToken, user }` | `mutate({ token, password })` | `authStore.setAuth` called with the response — confirms activation logs the invitee straight in (no intermediate login step), matching 8-2's explicit divergence from a plain "invite confirmed, now log in" flow |
| Expired/used token error distinct from a wrong-password shape | mock a token-specific error status | `mutate({ token, password })` | error is recognizable by the page as "token" class, not conflated with a credential error — since the invitee has no pre-existing password to have gotten wrong |

---

### 9.3 Test Case Detail — InviteActivationPage.test.tsx, AddStudentPage.test.tsx, GuardianSettingsPage.test.tsx, GuardianInviteStatusCard.test.tsx

**OWASP: A07:2021 – Identification and Authentication Failures (account activation flow).**

#### InviteActivationPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Password mismatch blocked client-side | enter `password` ≠ `confirmPassword` | submit | `useActivateInvite().mutate` never called |
| Success redirect depends on the response role, not a hardcoded route | mock success with `user.role = 'PARENT'` in one run and `'STUDENT'` in another | submit each | `navigate(roleDefaultRoute('PARENT'))` and `navigate(roleDefaultRoute('STUDENT'))` respectively — this is the case 8-2 explicitly calls out (the activated account "could be a Student or a guardian Parent"), so a hardcoded `/student` redirect here would silently misroute half of all activations |
| Expired/used token renders distinctly from a password-mismatch error | mock the token-class error from §9.2 | submit | a token-specific message renders, not the generic mismatch copy |

#### AddStudentPage

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Grade 6–12 shows activation-invite copy; Grade 1–5 does not | submit with `grade: 8`, then separately `grade: 3`, mocking success both times | — | confirmation copy differs per 8-2's rule — a Grade 6–12 add mentions an activation invite having been sent (since that child will separately log in); a Grade 1–5 add does not (parent-managed, no separate login) |
| Form resets after success, allowing another child to be added | mock success | submit, observe post-submit state | fields cleared, form remains on-page (no navigation away) |

#### GuardianSettingsPage (dual mode)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Parent mode lists all children via `useMyRelationships` | render at `/parent/guardian-settings` | — | one `GuardianInviteStatusCard` per relationship |
| Student mode renders the invite form, not the list-only view | render at `/student/guardian-invite` | — | email/phone form visible |
| Revoke requires confirmation before firing | Parent mode, click "Revoke" on a card | — | `useRevokeRelationship().mutate` **not** called until the confirmation dialog is explicitly accepted — a click-through here is irreversible per 8-2, making the confirmation gate itself the security-relevant control worth testing, not just a UX nicety |
| Student mode's existing invite is shown as status, not a duplicate form | Student mode, `useMyRelationships` already contains an `INVITED` relationship | — | that relationship renders as a status card; the invite form does not silently allow submitting a second invite to the same target without at least surfacing the existing one |

#### GuardianInviteStatusCard

| Case | Setup | Action | Expected result |
|---|---|---|---|
| "Resend" only visible when `status === 'INVITED'` | render with `status: 'ACTIVE'`, then `status: 'REVOKED'`, then `status: 'INVITED'` | — | button present only in the last case |
| `onResend` is called, not an internal mutation | render with an `onResend` spy | click "Resend" | spy called — confirms the component does not call `useResendInvite` itself (8-2: the parent page owns the mutation and its toast) |

---

### 9.4 Test Case Detail — SubjectRankingForm.test.tsx, AvailabilityCalendar.test.tsx

**OWASP: A04:2021 – Insecure Design (client-side guard vs. authoritative backend rule).**

#### SubjectRankingForm

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Third selection is disabled at the UI level, not just rejected on submit | `selections.length === 2` already | attempt to select a third subject | the control for an unselected subject is rendered `disabled` — asserted via the DOM attribute, not merely "onClick did nothing" — mirrors the backend's 400 rule as a pre-emptive UX guard (8-2), explicitly **not** a substitute for the backend's own enforcement |
| Removing a selection re-enables selection immediately (live re-evaluation) | `selections.length === 2`, then remove one | attempt to select a new subject | now enabled — the cap is evaluated against current `selections.length` on every render, not a one-time lock that could get stuck disabled or, worse, stuck enabled after a removal |
| Drag-to-reorder toggles rank 1 vs rank 2 | two subjects selected | reorder | `onSave` receives ranks reflecting the new order |
| `onSave` payload shape matches `useRankSubjects`' expected input | select two subjects, save | — | `[{ subjectId, rank: 1 }, { subjectId, rank: 2 }]`, unique ranks, no duplicate `rank` values |

#### AvailabilityCalendar

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Click on empty cell opens slot-creation form | render with no slots at a given cell | click that cell | inline form appears |
| Click on occupied cell offers "Remove," not a duplicate-create form | render with a slot at a given cell | click that cell | "Remove" confirmation, not a second creation form over an already-occupied cell |
| `onAddSlot`/`onRemoveSlot` called with the correct slot data | interact with the grid | — | callbacks receive day/start/end/recurring (add) or slot id (remove) matching what was clicked, not a stale/mismatched cell reference |
| No client-side overlap-detection logic beyond the same-cell disable | attempt an action on a cell adjacent to (but not identical to) an existing slot | — | the click is allowed through to `onAddSlot` — confirms the calendar does **not** invent its own overlap algorithm client-side (8-2's explicit note that any backend-side overlap rule beyond same-cell uniqueness surfaces as a toast on a 400/409, not a pre-emptive block here); this test exists to prevent someone "helpfully" adding client logic that silently diverges from the backend's actual rule |

---

### 9.5 Test Case Detail — TutorVerificationPage.test.tsx, PeopleManagementPage.test.tsx, SubjectManagementPage.test.tsx

**OWASP: A01:2021 – Broken Access Control (Admin-only), A04:2021 – Insecure Design (required-reason UX guard).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Approve fires with no confirmation dialog | render `TutorVerificationCard` | click "Approve" | `onApprove` called immediately — 8-2 explicitly notes this is reversible via suspend later, so no confirmation gate is expected here (a test asserting a confirmation dialog exists would be testing for behavior the spec deliberately does not want) |
| Reject requires a non-empty reason before firing | click "Reject" | leave the reason field blank, attempt to confirm | `onReject` **not** called; only fires once a non-empty reason is entered — same required-reason pattern enforced consistently with `PeopleManagementPage`'s suspend action below |
| Empty pending-tutor queue renders `EmptyState`, not an error | mock `usePendingTutors` → `{ tutors: [] }` | render | `EmptyState`, per API convention §0.3 |
| Suspend requires a non-empty reason | `PeopleManagementPage`, click "Suspend" on a row | leave reason blank | `useSuspendAccount().mutate` not called until a reason is provided |
| Role filter re-queries with the selected filter | change the role dropdown | — | `useUsers(page, roleFilter)` re-invoked with the new filter value |
| Subject Management shows both active and inactive subjects (admin-only view) | mock `useSubjects()` returning a mix of `isActive: true/false` | render | both are visible — distinguishes this admin view from the tutor/student-facing catalog per 8-2, which is expected to filter to active-only (tested in that catalog's own consuming page, not here) |

---

### 9.6 Test Case Detail — Sidebars.test.tsx (StudentSidebar, TutorSidebar, ParentSidebar, AdminSidebar)

**OWASP: A01:2021 – Broken Access Control (nav surface should not leak role-inappropriate entries).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Each sidebar renders only its own role's nav entries | render each of the four sidebars independently | — | `StudentSidebar` never renders `ParentSidebar`-only entries (Add Student, Guardian Settings) and vice versa — confirms `ParentSidebar` is genuinely a separate file per the M2 fix (0-frontend-conventions.md §0.2), not `StudentSidebar` with conditionally-hidden Parent items that could leak through a missed condition |
| Active nav item highlights based on `useLocation()` path prefix | render `StudentSidebar` at `/student/profile` | — | the "Profile" entry has the active-highlight class; others do not |
| Logout action wired to `useLogout` | click "Logout" in any sidebar | — | `useLogout().mutate()` called |

---

### 9.7 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `useAddStudent`'s grade-range test explicitly includes grades 1 and 5 (not just 6–12 examples), since the entire point of this hook is enabling the range `useRegister('student')` deliberately excludes.
- [ ] `SubjectRankingForm`'s "third selection disabled" case asserts the `disabled` DOM attribute directly, not just that a click handler no-ops — a form that fires `onSelect` but ignores the result internally would pass a looser assertion while still being a real accessibility/UX bug.
- [ ] `InviteActivationPage`'s role-dependent redirect test covers **both** Student and Parent outcomes in the same file, not just one, since a single-case test would not catch a hardcoded redirect that happens to match whichever role was tested.
- [ ] The `GuardianSettingsPage` revoke-confirmation test asserts the mutation is not called merely from clicking "Revoke" — it must also assert the mutation **is** called once the dialog is confirmed, so the test can't pass by simply never wiring the mutation at all.
- [ ] `AvailabilityCalendar`'s "no client-side overlap logic" case is a real behavioral assertion (the click is allowed through), not a lack-of-test — silently having no test here would be indistinguishable from "someone added overlap logic and nobody noticed," which is exactly the drift this case guards against.

---

### 9.8 Out of Scope for Automated Testing (and why)

- **`src/pages/tutor/AvailabilityPage.tsx`, `SubjectRankingPage.tsx`** — thin composition wrappers with no logic beyond passing hook data into the already-tested `AvailabilityCalendar`/`SubjectRankingForm` components; a full duplicate test suite here would only re-assert the same behavior through an extra layer of indirection.
- **Real drag-and-drop library internals** (`SubjectRankingForm`'s reorder) — the test suite asserts the *result* of a reorder (rank swap), not the drag library's own gesture-recognition correctness, which is that library's responsibility.
- **The flagged `useCreateSubject`/`useDeactivateSubject` hooks** — per 8-2, these were added to the frontend spec as a probable oversight fix rather than a confirmed design; their test cases are written against the assumed simple-mutation shape but are marked for re-verification once `02-accounts-guardianship-frontend.md` §2.5 is actually updated to include them formally.
- **Backend-side overlap-detection algorithm correctness** — covered by the backend's own `9-3-matching-cohorts.md`/availability-adjacent suites, not here; this doc only verifies the frontend does not duplicate or contradict that logic.

---

**Next:** proceed to → [9-3. Frontend Test Documentation: Matching & Cohorts]
