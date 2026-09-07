## Project: AKEWTutor — Frontend Function-Level Spec: Accounts & Guardianship
**Conventions:** see `0-frontend-conventions.md`. **API reference:** `02-accounts-guardianship-api.md`. **Frontend spec reference:** `02-accounts-guardianship-frontend.md`.

**Depends on:** Shared Config (hard — every page here requires `auth.store.ts`, `8-1`'s routing/layout shell).

**Links back to:** [07-frontend-specification/02-accounts-guardianship-frontend.md], [05b. Frontend Folder & File Structure §2]
**Links forward to:** [9-2. Frontend Test Spec: Accounts & Guardianship]

---

### Shared Pattern: Simple Query Hook (definition — reused by every feature file below)

A "simple query hook" wraps exactly one `GET` endpoint with no client-side branching beyond the wire response: `useQuery({ queryKey, queryFn: () => api.get(...).then(r => r.data), ...options })`. No further breakdown is given per such hook beyond a table row (endpoint, query key, params, any non-default option like `enabled`/`refetchInterval`) — the full request/response shape is already fixed in the corresponding Doc 07 file and is not repeated here. A hook only earns a full `Field | Detail` block below when it has logic beyond that (computed/derived values, conditional client-side validation, cross-hook coordination, or a non-obvious cache-invalidation target).

| Hook | Endpoint | Query key | Params / options |
|---|---|---|---|
| useMyStudentProfile | GET /students/me/profile | [STUDENT_PROFILE, studentId] | `studentId` optional (Parent only) |
| useMyTutorProfile | GET /tutors/me/profile | [TUTOR_PROFILE] | — |
| useMyAvailability | GET /tutors/me/availability | [AVAILABILITY] | — |
| useSubjects | GET /subjects | [SUBJECTS] | — |
| useMyRelationships | GET /guardianship/relationships | [RELATIONSHIPS] | — |
| usePendingTutors | GET /admin/tutors/pending | [PENDING_TUTORS, page] | — |
| useUsers | GET /admin/people | [ADMIN_PEOPLE, page, role] | `role` optional filter |

### Shared Pattern: Simple Mutation + Invalidation

Wraps one write endpoint, `onSuccess` invalidates exactly one query key family, no other branching:

| Hook | Endpoint | Invalidates |
|---|---|---|
| useUpdateProfile | PATCH /students/me/profile | [STUDENT_PROFILE, studentId] |
| useUpdateAcademicProfile | PATCH /students/me/academic-profile | [STUDENT_PROFILE, studentId] |
| useUpdateTutorProfile | PATCH /tutors/me/profile | [TUTOR_PROFILE] |
| useRankSubjects | PUT /tutors/me/subjects | [TUTOR_PROFILE] |
| useAddSlot | POST /tutors/me/availability | [AVAILABILITY] |
| useRemoveSlot | DELETE /tutors/me/availability/:id | [AVAILABILITY] |
| useRevokeRelationship | PATCH /guardianship/relationships/:id/revoke | [RELATIONSHIPS] |
| useApproveTutor | POST /admin/tutors/:id/approve | [PENDING_TUTORS] |
| useSuspendAccount | POST /admin/people/:id/suspend | [ADMIN_PEOPLE] |
| useEditRelationship | PATCH /admin/people/relationships/:id | [ADMIN_PEOPLE] |
| useAddStudent | POST /guardianship/students | (none — see full block, has redirect logic) |
| useResendInvite | POST /guardianship/invites/:id/resend | (none — fire-and-toast) |
| useInviteGuardian | POST /guardianship/guardian-invites | (none — fire-and-toast) |
| useRejectTutor | POST /admin/tutors/:id/reject | [PENDING_TUTORS] |

---

### src/hooks/useGuardianship.ts — full blocks

#### useAddStudent

| Field | Detail |
|---|---|
| Signature | `useAddStudent(): UseMutationResult<{ relationshipId: string; studentName: string }, AxiosError, { grade: number; studentName: string }>` |
| Purpose | Wraps `POST /guardianship/students` — the sole valid path for enrolling a Grade 1–5 child (§2.8). |
| Side effects | `onSuccess` invalidates `[RELATIONSHIPS]` so `AddStudentPage`'s "children" list picks up the new entry without a manual refetch. |
| Edge cases | No client-side grade restriction below 1 or above 12 needs enforcing here beyond basic range, since this is the *correct* endpoint for the full 1–12 range (unlike `/auth/register/student`, which is 6–12 only) — the form's copy, not its validation, carries the distinction (§2.8). |
| Test file | `tests/hooks/useGuardianship.test.ts` |

#### useActivateInvite

| Field | Detail |
|---|---|
| Signature | `useActivateInvite(): UseMutationResult<LoginResponse, AxiosError, { token: string; password: string }>` |
| Purpose | Wraps `POST /guardianship/invites/:token/activate` — the one guardianship flow reachable with no existing session (§2.4). |
| Side effects | On success, calls `authStore.setAuth(data.accessToken, data.user)` directly (same shape as `useLogin`'s success handler in 8-1) — activation logs the invitee straight in, it does not merely confirm the invite and then require a separate login step. |
| Edge cases | An expired/already-used token renders a form-level error distinct from a wrong-password-style error, since the person has no password to have gotten wrong yet at that point. |
| Test file | `tests/hooks/useGuardianship.test.ts` |

---

### src/pages/InviteActivationPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Grade 1–5 student or invited optional guardian sets a password and activates. |
| Route guard + layout | `PublicRoute` + `AuthLayout` |
| Local state | `token` read from the URL path param; React Hook Form — `password`, `confirmPassword`. |
| Behavior | 1. Client-side check `password === confirmPassword` before submit. 2. `useActivateInvite().mutate({ token, password })`. 3. On success, `navigate(roleDefaultRoute(data.user.role))` (per 8-1's shared util) — the activated account could be a Student or a guardian Parent, so the redirect target genuinely depends on the response, not a hardcoded `/student`. |

**States:** idle · submitting · error (expired/used token, password mismatch) · success (redirect)

### src/pages/student/AcademicProfilePage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/student/profile` — `ProtectedRoute(['STUDENT'])` + `DashboardLayout(StudentSidebar)` |
| Behavior | 1. `useMyStudentProfile()` (no `studentId` — self view). 2. Two forms: basic profile (`useUpdateProfile`, just `profilePictureUrl`) and academic profile (`useUpdateAcademicProfile`, the larger field set — `school`, `subjectsOfInterest`, `learningGoals`, etc.), submitted independently since they hit different endpoints. |

**States:** loading · error · success

### src/pages/parent/AddStudentPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/parent/add-student` — `ProtectedRoute(['PARENT'])` + `DashboardLayout(ParentSidebar)` |
| Local state | React Hook Form — `grade` (1–12), `studentName`. |
| Behavior | 1. On submit, `useAddStudent().mutate({ grade, studentName })`. 2. On success, shows a confirmation with the new `relationshipId` and, if `grade` is 6–12 (so the child will separately activate their own login later rather than being purely Parent-managed), copy explaining an activation invite has been sent. |

**States:** idle · submitting · error · success (confirmation, form resets for adding another child)

### src/pages/parent/GuardianSettingsPage.tsx, src/pages/student/GuardianSettingsPage.tsx (new)

| Field | Detail |
|---|---|
| Route (Parent) | `/parent/guardian-settings` — `ProtectedRoute(['PARENT'])` + `DashboardLayout(ParentSidebar)` |
| Route (Student) | `/student/guardian-invite` — `ProtectedRoute(['STUDENT'])` + `DashboardLayout(StudentSidebar)`, rendered in "invite-optional-guardian" mode (UC-07) |
| Behavior (Parent mode) | 1. `useMyRelationships()` lists all children. 2. Each row renders `GuardianInviteStatusCard` (below). 3. "Revoke" action → `useRevokeRelationship().mutate(id)`, behind a confirmation dialog since revocation is not reversible from this screen. |
| Behavior (Student mode) | 1. Simple form: email or phone, `useInviteGuardian().mutate({ email, phone })`. 2. Below the form, `useMyRelationships()` filtered to relationships where the caller is the student side, so an already-invited guardian shows status rather than allowing a duplicate invite silently. |

**States:** loading · error · success (list ± invite form depending on mode)

### src/components/GuardianInviteStatusCard.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ relationship: ParentStudentRelationship; onResend: () => void }` |
| Behavior | Renders a `StatusBadge` (foundation component) for `INVITED` / `ACTIVE` / `REVOKED`; shows a "Resend" button only when `status === 'INVITED'`, wired to `useResendInvite` at the call site (`onResend` prop, not called internally, so the parent page controls the mutation and its toast). |

---

### src/pages/tutor/TutorProfilePage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/tutor/profile` — `ProtectedRoute(['TUTOR'])` + `DashboardLayout(TutorSidebar)` |
| Behavior | 1. `useMyTutorProfile()`. 2. Read-only `verificationStatus` badge (`PENDING`/`APPROVED`/`REJECTED`) — this field is never part of the edit form, since only Admin can change it (`useApproveTutor`/`useRejectTutor`, admin-only hooks below). 3. Editable fields (`qualifications`, `experienceYears`, `education`) submit via `useUpdateTutorProfile`. |

**States:** loading · error · success

### src/pages/tutor/SubjectRankingPage.tsx, src/components/SubjectRankingForm.tsx (new)

| Field | Detail |
|---|---|
| Route | `/tutor/subjects` — `ProtectedRoute(['TUTOR'])` + `DashboardLayout(TutorSidebar)` |
| `SubjectRankingForm` props | `{ subjects: Subject[]; currentRankings: TutorSubjectRanking[]; onSave: (ranked: { subjectId: string; rank: 1 \| 2 }[]) => void }` |
| Local state (form) | `selections: { subjectId: string; rank: 1 \| 2 }[]`, max length 2. |
| Behavior | 1. Renders `useSubjects()`'s full catalog as a searchable/selectable list. 2. Selecting a subject when `selections.length === 2` is **disabled at the UI level** (§2.8) — the third checkbox/button is rendered `disabled`, not merely validated on submit, mirroring the backend's 400 rule ("Each subject may be ranked once, and ranks must be unique"). 3. Drag-to-reorder between the two selected subjects toggles which is rank 1 vs rank 2. 4. "Save" calls `onSave(selections)`, which the page wires to `useRankSubjects().mutate(selections)`. |
| Edge cases | Removing a selection (to swap for a different subject) re-enables selection immediately — the 2-item cap is evaluated live against current `selections.length`, not a one-time lock. |
| Test file | `tests/components/SubjectRankingForm.test.tsx` |

**States (page):** loading (subjects + current rankings) · error · success

### src/pages/tutor/AvailabilityPage.tsx, src/components/AvailabilityCalendar.tsx (new)

| Field | Detail |
|---|---|
| Route | `/tutor/availability` — `ProtectedRoute(['TUTOR'])` + `DashboardLayout(TutorSidebar)` |
| `AvailabilityCalendar` props | `{ slots: AvailabilitySlot[]; onAddSlot: (slot) => void; onRemoveSlot: (slotId) => void }` |
| Behavior | 1. Renders a 7-day-by-hour weekly grid from `slots`. 2. Click on an empty cell opens an inline slot-creation form (day/start/end/recurring), submitting via `onAddSlot`. 3. Click on an existing slot offers a "Remove" confirmation, submitting via `onRemoveSlot`. |
| Edge cases | Overlapping-slot prevention is **not** duplicated client-side beyond a same-cell-already-occupied visual disable — the backend's overlap-detection business rule (if any beyond simple day/time uniqueness) is authoritative and any resulting 400/409 surfaces as a toast rather than being pre-validated here, since the exact overlap algorithm belongs to the backend spec, not this file. |
| Test file | `tests/components/AvailabilityCalendar.test.tsx` |

**States (page):** loading · error · success

---

### src/pages/admin/TutorVerificationPage.tsx, src/components/TutorVerificationCard.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/tutor-verification` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| `TutorVerificationCard` props | `{ tutor: TutorProfile & { userId: string }; onApprove: () => void; onReject: (reason: string) => void }` |
| Behavior (card) | Renders the pending tutor's profile summary; "Approve" calls `onApprove` directly (no confirmation — reversible via suspend later); "Reject" opens an inline reason field (required, non-empty) before calling `onReject(reason)`, mirroring `useRejectTutor`'s required field. |
| Behavior (page) | `usePendingTutors(page)` lists cards; wires each card's callbacks to `useApproveTutor().mutate(tutorId)` / `useRejectTutor().mutate({ tutorId, reason })`. |

**States:** loading · empty (`tutors: []`, steady state not an error) · success

### src/pages/admin/PeopleManagementPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/people` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | 1. `useUsers(page, roleFilter)` — a role filter dropdown drives `roleFilter`. 2. Each row's "Suspend" action opens a required-reason dialog before calling `useSuspendAccount().mutate({ userId, reason })` — same required-reason pattern as tutor rejection above, kept consistent across the feature rather than one screen requiring a reason and another not. 3. A relationship-editing affordance (for correcting a miskeyed Parent-Student link) wires to `useEditRelationship`. |

**States:** loading · success (paginated table)

### src/pages/admin/SubjectManagementPage.tsx (new)

| Field | Detail |
|---|---|
| Route | `/admin/subjects` — `ProtectedRoute(['ADMIN'])` + `DashboardLayout(AdminSidebar)` |
| Behavior | 1. `useSubjects()` lists all subjects (active and inactive — this admin view, unlike the tutor/student-facing catalog, shows both). 2. Create/deactivate actions call endpoints not otherwise surfaced elsewhere in this feature's frontend spec — `useCreateSubject`/`useDeactivateSubject` (named in Doc 05b §2 but not given a hook-level entry in `02-accounts-guardianship-frontend.md`'s §2.5/2.6; this file adds them here as simple mutations invalidating `[SUBJECTS]`, since the page cannot function without them and Doc 07 evidently omitted them by oversight rather than by design — **flagging this addition rather than silently inventing it**, worth a quick confirmation with whoever owns Doc 07 that these two hooks belong in `02-accounts-guardianship-frontend.md` §2.5). |

**States:** loading · success

---

### src/components/StudentSidebar.tsx, TutorSidebar.tsx, AdminSidebar.tsx, ParentSidebar.tsx (new/modify)

| File | Nav entries |
|---|---|
| `StudentSidebar.tsx` | Profile, Find a Tutor (→ 8-3), Upcoming Classes (→ 8-4), Progress (→ 8-4), Messaging, Payments, Achievements/Leaderboard (→ 8-6), Notifications, Support/Complaints (→ 8-8) |
| `TutorSidebar.tsx` | Profile, Subjects, Availability, Upcoming Classes (→ 8-4), Messaging, Earnings (→ 8-7), Notifications, Support/Complaints (→ 8-8) |
| `ParentSidebar.tsx` (new, per frontend spec §2.1's flag — carried forward, not re-litigated here) | Add Student, Guardian Settings, (child's) Upcoming Classes (→ 8-4), (child's) Payments (→ 8-7), Notifications |
| `AdminSidebar.tsx` | Tutor Verification, People, Subjects — plus further entries added incrementally by 8-3 (Matching Queue), 8-6 (Badges/Challenges), 8-7 (Pricing/Refunds/Payouts/Promotions), 8-8 (Disputes/Reports); this file specifies only the entries this feature owns |

Each sidebar highlights the active nav item from the current route (`useLocation().pathname` prefix match against its own `ROUTES` entries) and renders a "Logout" action wired to `useLogout().mutate()` (8-1).

---

**Next:** proceed to → [8-3. Frontend: Matching & Cohorts]
