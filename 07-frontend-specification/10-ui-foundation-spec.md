## AKEWTutor — UI Foundation Spec

**Project:** AKEWTutor — Online Tutoring Platform

**Links back to:** [05b. Frontend Folder & File Structure], [Feature Decomposition §3]
**Links forward to:** [07. Frontend Specification (features 1–8)]

**Status: visual design DEFERRED — behavioral contract only.**

The design tokens, color palette, spacing scale, typography, and final visual treatment of every primitive and common component listed below are an intentional open decision — not an oversight — held until the product owner finalizes the app's look and feel. Nothing in this document should be read as a visual spec.

What this document *does* provide: the **props/behavior contract** for every foundation-layer component already referenced by name across Docs 05b, 07 (features 1–8), and 08. This exists so that hook logic, data wiring, state management, and page composition in the numbered feature docs can be built and tested now, against a stable interface, without waiting on the visual design. When the real UI foundation spec is authored, it should implement these same props/behaviors — the visual layer changes, the contract underneath should not need to.

**What this document does NOT cover (pending):** color tokens, the `space-*` spacing scale's actual values, typography scale, dark/light theme (if any), icon set, animation/motion conventions, responsive breakpoints beyond what Tailwind ships by default, and final visual treatment of every state (hover/focus/disabled/error) for each primitive.

> ⚠️ **Resolution path (audit note, revised):** the tokens below are pending a real design pass, not a documentation fix — they'll be resolved via: (1) design/prototype the UI, (2) generate a design file from that prototype, (3) derive the `ui-foundation` component contracts (colors, spacing scale, typography) from the design file, (4) write `src/styles/globals.css`'s `@theme` block from those tokens. This document's props/behavior contracts (§0.1–0.3) are already stable and buildable against now; only the **values** — not the contracts — are waiting on that design pass. When it's done, this section is replaced in place with the resolved token values; no other document changes as a result (per the note at the end of this file).

---

### 0.1 Base primitives (`src/components/ui/`)

These wrap shadcn/ui equivalents (per Doc 07 §0's stated template) with no AKEWTutor-specific behavior beyond standard form/interaction semantics. Listed so downstream docs have a name and prop surface to reference; visual styling TBD.

**Fix (2026-09-07 audit):** filenames below are lowercase, matching the template's shadcn CLI output exactly (`npx shadcn@latest add button` writes `button.tsx`, imported as `@/components/ui/button`) — previously listed as PascalCase (`Button.tsx`), which didn't match what the CLI actually generates. Component/export names inside each file are unaffected (shadcn exports `Button`, `Card`, etc. regardless of filename casing) — only the filename and import path change.

| Component | Contract |
|---|---|
| `button.tsx` | `variant` ('primary' \| 'secondary' \| 'destructive' \| 'ghost'), `size` ('sm' \| 'md' \| 'lg'), `disabled`, `loading` (shows an inline spinner, keeps label visible), standard `onClick` |
| `card.tsx` | Plain container; `className` passthrough only — no forced padding/shadow contract yet |
| `input.tsx` | Standard controlled text input; `error` (boolean, for RHF+Zod integration per Doc 07's form conventions), `disabled` |
| `label.tsx` | Wraps `<label>`; `required` (renders an asterisk indicator) |
| `checkbox.tsx` | Standard controlled checkbox; `disabled`, `indeterminate` |
| `badge.tsx` | Generic pill; `variant` for semantic color grouping (specific colors TBD) — `StatusBadge` (below) is the specialized, semantically-aware wrapper most feature docs actually reference |

### 0.2 Common components (`src/components/common/`)

| Component | Props / behavior contract |
|---|---|
| `StatusBadge.tsx` | `status: string` (the raw enum value from the API — e.g. `PENDING`, `APPROVED`, `OVERDUE`, `MISSING`, `ESCALATED`, `INVITED`, `ACTIVE`, `REVOKED`, `COMPLETED`, `PAID`) + `severity?: 'default' \| 'warning' \| 'danger' \| 'success'` optional override for cases where the same status name needs different visual weight in different tables (per the severity-tiering convention referenced in 08-4's `ApprovalQueueTable`/`PlatformStatsGrid`/compliance-queue usage). Maps a closed set of known status strings to a pill; unmapped strings render with a neutral default rather than throwing. |
| `EmptyState.tsx` | `message: string`, `icon?: ReactNode`, `action?: { label: string; onClick: () => void }`. Renders a centered message with optional icon and single optional call-to-action button. Used both for genuine "nothing here yet" states and for expected 403/404 outcomes that should not render as error toasts (per 07 §4/§5's explicit usage). |
| `CountdownTimer.tsx` | `targetIso: string` (ISO datetime to count down to), `onExpire?: () => void`. Renders remaining time; calls `onExpire` once when the target passes. Used for `groupFormationWindowExpiresAt`, `dueDate` (payment reminders), invite expiry, and class-start countdowns — all pass a plain ISO string, no feature-specific variant needed. |
| `TopNavBar.tsx` | Renders current-user identity + role, `NotificationBell`, and a logout action. Content is role-aware (per Doc 05b's per-role layout), visual treatment TBD. |
| `Footer.tsx` | Static; no dynamic props expected. |
| `NotificationBell.tsx` | Wraps `useNotifications()` (shared-config, Doc 07 §1); renders unread count badge, opens a dropdown/panel listing recent `Notification` rows. No new data-fetching contract beyond that hook. |

### 0.3 Layout shell components

Per Doc 05b's per-role sidebar files (`StudentSidebar.tsx`, `TutorSidebar.tsx`, `AdminSidebar.tsx`, `ParentSidebar.tsx` — see M2 resolution in Doc 07 §0.7) and route-guard wrapper: contract is a static nav-item list per role plus an `active` route highlight; no props beyond the current route. Visual treatment TBD.

---

### 0.4 Spacing/token namespace (placeholder only)

Doc 05b's spacing namespace rule (`space-*` prefix, e.g. `mt-space-sm`) is a **naming convention**, fixed now to avoid collisions with Tailwind's built-in scale. The actual pixel/rem values behind `space-sm` / `space-md` / `space-lg` / `space-gutter` / `space-margin-mobile` are **not yet assigned** — they're derived from the design/prototype pass described above (§0's resolution-path note), then written into `globals.css`'s `@theme` block. Treat every current usage of these classes in Docs 07/08 as referencing a token name, not a resolved value, until that pass is complete.

---

**When this document is completed for real** (once visual design is finalized), replace this file's content in place — every prop/behavior contract above should still hold; only the "TBD" sections need resolving. No other document should need to change as a result, since nothing in Docs 05b/07/08 depends on the *visual* values, only on the props/behavior contracts fixed here.
