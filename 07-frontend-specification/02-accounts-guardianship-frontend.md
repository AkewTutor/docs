## Project: AKEWTutor — Frontend Specification
**Feature:** Accounts & Guardianship
**Conventions:** see `0-frontend-conventions.md`.
**API reference:** `02-accounts-guardianship-api.md`

**Depends on:** Shared Config (hard — every page here requires `auth.store.ts` to already exist).

**Links back to:** [0. Frontend Conventions], [06-api/02-accounts-guardianship-api.md], [05b. Frontend Folder & File Structure §2]
**Links forward to:** [8-2. Frontend Function-Level Spec: Accounts & Guardianship]

---

### 2.1 Routes

```
/invite/:token/activate        → PublicRoute → AuthLayout → InviteActivationPage
/student/profile                → ProtectedRoute(['STUDENT']) → DashboardLayout(StudentSidebar) → AcademicProfilePage
/parent/add-student              → ProtectedRoute(['PARENT'])  → DashboardLayout(ParentSidebar)  → AddStudentPage
/parent/guardian-settings        → ProtectedRoute(['PARENT'])  → DashboardLayout(ParentSidebar)  → GuardianSettingsPage
/student/guardian-invite         → ProtectedRoute(['STUDENT']) → DashboardLayout(StudentSidebar) → GuardianSettingsPage (invite-optional-guardian mode, UC-07)
/tutor/profile                   → ProtectedRoute(['TUTOR'])   → DashboardLayout(TutorSidebar)   → TutorProfilePage
/tutor/subjects                  → ProtectedRoute(['TUTOR'])   → DashboardLayout(TutorSidebar)   → SubjectRankingPage
/tutor/availability               → ProtectedRoute(['TUTOR'])   → DashboardLayout(TutorSidebar)   → AvailabilityPage
/admin/tutor-verification          → ProtectedRoute(['ADMIN'])   → DashboardLayout(AdminSidebar)   → TutorVerificationPage
/admin/people                     → ProtectedRoute(['ADMIN'])   → DashboardLayout(AdminSidebar)   → PeopleManagementPage
/admin/subjects                   → ProtectedRoute(['ADMIN'])   → DashboardLayout(AdminSidebar)   → SubjectManagementPage
```

> **M2 fix — resolved:** `ParentSidebar.tsx` is specified here (added to Doc 05b's file inventory) because Parent's page set — guardianship management, a scoped view of their child's profile/payments/achievements — is materially distinct from Student's. It composes the same `DashboardLayout` shell as `StudentSidebar`/`TutorSidebar`/`AdminSidebar`, differing only in nav items: Add Student, Guardian Settings, (child's) Upcoming Classes, (child's) Payments, (child's) Achievements, Notifications.

### 2.2 Types (added to src/types/index.ts)

```typescript
export interface StudentProfile {
  id: string;
  userId: string;
  grade: number;
  school: string | null;
  profilePictureUrl: string | null;
  subjectsOfInterest: string[]; // subject UUIDs
  academicLevel: string | null;
  learningGoals: string | null;
  preferredLanguage: string | null;
  learningSchedulePreference: { days: string[]; timeOfDay: string } | null;
  teachingStylePreference: string | null;
  budgetPreference: string | null; // Decimal-as-string
  formatPreference: 'ONE_TO_ONE' | 'ONE_TO_THREE' | 'ONE_TO_FIVE' | null;
  accountStatus: string;
}

export interface ParentStudentRelationship {
  id: string;
  studentId: string;
  parentId: string;
  status: 'INVITED' | 'ACTIVE' | 'REVOKED';
  createdAt: string;
}

export interface TutorProfile {
  id: string;
  userId: string;
  qualifications: string | null;
  experienceYears: number | null;
  education: string | null;
  verificationStatus: 'PENDING' | 'APPROVED' | 'REJECTED';
}

export interface TutorSubjectRanking {
  subjectId: string;
  subjectName: string;
  rank: 1 | 2;
}

export interface AvailabilitySlot {
  id: string;
  dayOfWeek: number; // 0-6
  startTime: string;
  endTime: string;
  isRecurring: boolean;
}

export interface Subject {
  id: string;
  name: string;
  isActive: boolean;
}
```

### 2.3 Hooks (src/hooks/useStudentProfile.ts)

```typescript
export function useMyStudentProfile(studentId?: string) {
  // studentId required when caller is Parent (per API §2.2's scoping rule); omitted for Student self-view
  return useQuery({
    queryKey: [QUERY_KEYS.STUDENT_PROFILE, studentId],
    queryFn: () => api.get<StudentProfile>('/students/me/profile', { params: { studentId } }).then((r) => r.data),
  });
}

export function useUpdateProfile(studentId?: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { studentId?: string; profilePictureUrl?: string }) =>
      api.patch<StudentProfile>('/students/me/profile', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.STUDENT_PROFILE, studentId] }),
  });
}

export function useUpdateAcademicProfile(studentId?: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: Partial<StudentProfile> & { studentId?: string }) =>
      api.patch<StudentProfile>('/students/me/academic-profile', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.STUDENT_PROFILE, studentId] }),
  });
}
```

### 2.4 Hooks (src/hooks/useGuardianship.ts)

```typescript
export function useAddStudent() {
  return useMutation({
    mutationFn: (body: { grade: number; studentName: string }) =>
      api.post('/guardianship/students', body),
  });
}

export function useResendInvite() {
  return useMutation({
    mutationFn: (relationshipId: string) => api.post(`/guardianship/invites/${relationshipId}/resend`),
  });
}

export function useActivateInvite() {
  return useMutation({
    mutationFn: ({ token, password }: { token: string; password: string }) =>
      api.post(`/guardianship/invites/${token}/activate`, { password }),
  });
}

export function useInviteGuardian() {
  // UC-07: a Grade 6-12 student invites an optional guardian
  return useMutation({
    mutationFn: (body: { email?: string; phone?: string }) => api.post('/guardianship/guardian-invites', body),
  });
}

export function useMyRelationships() {
  return useQuery({
    queryKey: [QUERY_KEYS.RELATIONSHIPS],
    queryFn: () => api.get<{ relationships: ParentStudentRelationship[] }>('/guardianship/relationships').then((r) => r.data),
  });
}

export function useRevokeRelationship() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => api.patch(`/guardianship/relationships/${id}/revoke`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.RELATIONSHIPS] }),
  });
}
```
`InviteActivationPage.tsx` reads `:token` from the URL and calls `useActivateInvite` on submit — this is the one guardianship flow reachable by someone with no existing session at all (a not-yet-registered Grade 1–5 student, or an invited optional guardian), which is why it lives under `AuthLayout`/`PublicRoute` in §2.1 rather than a protected route.

### 2.5 Hooks (src/hooks/useTutorProfile.ts, useAvailability.ts, useSubjects.ts)

```typescript
export function useMyTutorProfile() {
  return useQuery({
    queryKey: [QUERY_KEYS.TUTOR_PROFILE],
    queryFn: () => api.get<TutorProfile>('/tutors/me/profile').then((r) => r.data),
  });
}

export function useUpdateTutorProfile() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: Partial<TutorProfile>) => api.patch<TutorProfile>('/tutors/me/profile', body).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.TUTOR_PROFILE] }),
  });
}

export function useRankSubjects() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (subjects: { subjectId: string; rank: 1 | 2 }[]) =>
      api.put<{ subjects: TutorSubjectRanking[] }>('/tutors/me/subjects', { subjects }).then((r) => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.TUTOR_PROFILE] }),
  });
}

export function useMyAvailability() {
  return useQuery({
    queryKey: [QUERY_KEYS.AVAILABILITY],
    queryFn: () => api.get<{ slots: AvailabilitySlot[] }>('/tutors/me/availability').then((r) => r.data),
  });
}

export function useAddSlot() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: { dayOfWeek: number; startTime: string; endTime: string; isRecurring: boolean }) =>
      api.post('/tutors/me/availability', body),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.AVAILABILITY] }),
  });
}

export function useRemoveSlot() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (slotId: string) => api.delete(`/tutors/me/availability/${slotId}`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.AVAILABILITY] }),
  });
}

export function useSubjects() {
  return useQuery({
    queryKey: [QUERY_KEYS.SUBJECTS],
    queryFn: () => api.get<{ subjects: Subject[] }>('/subjects').then((r) => r.data),
  });
}
```

### 2.6 Hooks (src/hooks/useAdminTutorVerification.ts, useAdminPeople.ts)

```typescript
export function usePendingTutors(page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.PENDING_TUTORS, page],
    queryFn: () => api.get('/admin/tutors/pending', { params: { page, limit: 20 } }).then((r) => r.data),
  });
}

export function useApproveTutor() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (tutorId: string) => api.post(`/admin/tutors/${tutorId}/approve`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.PENDING_TUTORS] }),
  });
}

export function useRejectTutor() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ tutorId, reason }: { tutorId: string; reason: string }) =>
      api.post(`/admin/tutors/${tutorId}/reject`, { reason }),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.PENDING_TUTORS] }),
  });
}

export function useUsers(page = 1, role?: Role) {
  return useQuery({
    queryKey: [QUERY_KEYS.ADMIN_PEOPLE, page, role],
    queryFn: () => api.get('/admin/people', { params: { page, limit: 20, role } }).then((r) => r.data),
  });
}

export function useSuspendAccount() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ userId, reason }: { userId: string; reason: string }) =>
      api.post(`/admin/people/${userId}/suspend`, { reason }),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_PEOPLE] }),
  });
}

export function useEditRelationship() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, ...body }: { id: string; [k: string]: unknown }) =>
      api.patch(`/admin/people/relationships/${id}`, body),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_PEOPLE] }),
  });
}
```

### 2.7 Components

| File | Responsibility |
|---|---|
| `GuardianInviteStatusCard.tsx` | Shows one relationship's state (`INVITED`/expiring/`ACTIVE`) with a resend action when `INVITED` |
| `SubjectRankingForm.tsx` | Drag-to-rank UI over `useSubjects()`'s catalog; **hard-blocks selecting a 3rd subject client-side** (disables further selection once 2 are picked) mirroring the backend's 400 rule, so the error is prevented rather than only caught after submit |
| `AvailabilityCalendar.tsx` | Weekly grid; click-to-add opens a slot form, click-existing offers remove (`useRemoveSlot`) |
| `TutorVerificationCard.tsx` | One pending tutor's profile summary + Approve/Reject actions (Reject requires a `reason`, matching `useRejectTutor`'s required field) |
| `StudentSidebar.tsx`, `TutorSidebar.tsx`, `AdminSidebar.tsx` | Existing per Doc 05b |
| `ParentSidebar.tsx` (new, per §2.1's flag) | Parent-specific nav: Add Student, Guardian Settings, child's Upcoming Classes/Payments, Notifications |

### 2.8 Form Validation Notes

- `SubjectRankingForm`'s client-side rank uniqueness check mirrors the backend's 400 ("Each subject may be ranked once, and ranks must be unique") but the backend remains authoritative — the form disables the "Save" action rather than only relying on a post-submit error.
- `AddStudentPage`'s grade field accepts 1–12; a value routed here (via Parent, `POST /guardianship/students`) is the *only* valid path for Grades 1–5 (per `01-shared-config-api.md`'s explicit 400 on `/auth/register/student` for that range) — the form's copy should make clear this is the correct place for a younger child, not a workaround.

---

**Next:** proceed to → [03. Matching & Cohorts Frontend]
