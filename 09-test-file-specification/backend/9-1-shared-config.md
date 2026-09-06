## Project: AKEWTutor — Backend Test Documentation: Shared Config (Auth, Notifications, Policies, Announcements) + Cross-Cutting Foundations
**Links back to:** [05a. Backend Folder & File Structure §0, §1], [8-1. Function-Level Spec: Shared Config]
**Conventions:** see `00-api-conventions.md` §0.1–0.7 for envelope shapes, auth labels, and common error statuses referenced throughout.

Per the standing rule in `05a-backend-structure.md` ("every service file gets a mirrored test file under `tests/`"): test file mirrors `src/` exactly under `tests/`, never co-located. Test runner: Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())` to reset mock state between cases. Every test suite in this project is additionally required to cover: (1) every FR/NFR the source file implements, traced explicitly below; (2) the relevant OWASP Top 10 (2021) risk categories for that file, per `9.X` security subsections; (3) any edge case already flagged in Doc 08's own function-level spec.

**Scope note:** this is the one test doc that also covers Doc 05a §0's cross-cutting foundations (middleware, jwt/password/pagination utils) — they have no owning feature, and shared-config is the foundation feature they sit alongside architecturally.

---

### 9.0 FR/NFR Traceability Summary

| Source file | FRs covered | NFRs covered |
|---|---|---|
| auth.middleware.ts | (enforces role/ownership gates for every other feature) | NFR-007, NFR-009 |
| errorHandler.middleware.ts | — | NFR-007 (never leaks internals) |
| validate.middleware.ts | — | NFR-007 (input hygiene, first line of injection defense) |
| jwt.ts | FR-SP-003 | NFR-007, NFR-008 |
| password.ts | FR-SP-003 | NFR-008 |
| pagination.ts | — (shared list helper referenced by NFR-004 at scale) | — |
| auth.service.ts | FR-SP-001–005, FR-TU-001–002, FR-AC-002, FR-AC-005 | NFR-007, NFR-008 |
| notification.service.ts | FR-NO-001–011 | NFR-004 (must not block the triggering action) |
| adminAnnouncement.service.ts | FR-AD-019 | — |
| policy.service.ts | FR-SC-001, FR-AD-015 | NFR-010 (retains full version history, not a privacy risk here since content is public) |

NFR-001, NFR-002, NFR-003, NFR-005, NFR-006, and OWASP A06:2021 are intentionally not claimed by any file in this doc set — see §9.20 for why and what covers them instead.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| prisma/schema.prisma | — | Not required (declarative, see 9.6) | — |
| prisma/seed.ts | — | Not required (see 9.6) | — |
| src/middleware/auth.middleware.ts | tests/middleware/auth.middleware.test.ts | Unit (mocked `jwt.ts`) | ☐ |
| src/middleware/errorHandler.middleware.ts | tests/middleware/errorHandler.middleware.test.ts | Unit (mocked `req`/`res`/`next`) | ☐ |
| src/middleware/validate.middleware.ts | tests/middleware/validate.middleware.test.ts | Unit (real Zod schemas, mocked `req`/`res`/`next`) | ☐ |
| src/utils/jwt.ts | tests/utils/jwt.test.ts | Unit (real `jsonwebtoken`, test secret) | ☐ |
| src/utils/password.ts | tests/utils/password.test.ts | Unit (real `bcrypt`) | ☐ |
| src/utils/pagination.ts | tests/utils/pagination.test.ts | Unit (pure function) | ☐ |
| src/services/auth.service.ts | tests/services/auth.service.test.ts | Unit (mocked Prisma, bcrypt, jwt, sms/email clients) | ☐ |
| src/controllers/auth.controller.ts | tests/controllers/auth.controller.test.ts | Unit (mocked auth.service via `vi.mock`) | ☐ |
| src/routes/auth.routes.ts | tests/routes/auth.routes.test.ts | Integration (supertest, mocked service layer) | ☐ |
| src/utils/providers/sms.client.ts | tests/utils/providers/sms.client.test.ts | Unit (mocked Geez SMS HTTP call) | ☐ |
| src/utils/providers/email.client.ts | tests/utils/providers/email.client.test.ts | Unit (mocked Brevo HTTP call) | ☐ |
| src/services/notification.service.ts | tests/services/notification.service.test.ts | Unit (mocked Prisma, sms/email clients) | ☐ |
| src/controllers/notification.controller.ts | tests/controllers/notification.controller.test.ts | Unit (mocked notification.service) | ☐ |
| src/routes/notification.routes.ts | tests/routes/notification.routes.test.ts | Integration (supertest, mocked service layer) | ☐ |
| src/services/adminAnnouncement.service.ts | tests/services/adminAnnouncement.service.test.ts | Unit (mocked Prisma, mocked `notification.service.dispatchNotification`) | ☐ |
| src/controllers/adminAnnouncement.controller.ts | tests/controllers/adminAnnouncement.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/adminAnnouncement.routes.ts | tests/routes/adminAnnouncement.routes.test.ts | Integration (supertest) | ☐ |
| src/services/policy.service.ts | tests/services/policy.service.test.ts | Unit (mocked Prisma) | ☐ |
| src/controllers/policy.controller.ts | tests/controllers/policy.controller.test.ts | Unit (mocked service) | ☐ |
| src/routes/policy.routes.ts | tests/routes/policy.routes.test.ts | Integration (supertest) | ☐ |
| src/jobs/notificationRetry.job.ts | — | Not required — underlying logic covered by `notification.service.test.ts`'s `retryFailed` cases; the interval-registration wrapper itself is excluded per Doc 05a's standing convention (only `src/services/*` gets a mirrored test) | — |
| src/jobs/scheduler.ts | — | Not required — no business logic of its own (Doc 8-1) | — |

---

### 9.2 Test Case Detail — auth.middleware.test.ts

FRs: (infrastructure for every authenticated FR). NFRs: NFR-007, NFR-009. **OWASP: A07:2021 – Identification and Authentication Failures, A01:2021 – Broken Access Control.**

#### authMiddleware

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid token — attaches req.user | mock `verifyAccessToken` to return `{ id, role }` | call `authMiddleware(req, res, next)` with a well-formed `Authorization: Bearer <token>` header | `req.user = { id, role }` set; `next()` called with no error |
| Missing Authorization header | no header on `req` | call `authMiddleware` | throws/passes to `next` an `ApiError(401, "Missing, invalid, or expired token")` |
| Malformed header (no "Bearer " prefix) | `Authorization: sometoken` | call `authMiddleware` | same `ApiError(401, ...)` |
| Expired token | mock `verifyAccessToken` to throw a `TokenExpiredError` | call `authMiddleware` | `ApiError(401, "Missing, invalid, or expired token")` — expiry is not distinguished from any other invalid-token reason in the response, per Doc 8-1 |
| Signature mismatch / tampered token | mock `verifyAccessToken` to throw a `JsonWebTokenError` | call `authMiddleware` | same `ApiError(401, ...)` |
| Does not re-fetch the User row | mock `verifyAccessToken` to return `{ id, role }`; spy on `prisma.user.findUnique` | call `authMiddleware` | assert `prisma.user.findUnique` was **never** called — `req.user` is built solely from the token payload, per Doc 8-1's explicit edge case |
| Algorithm-confusion / "none" algorithm rejected | craft a token with `alg: "none"` or an unexpected algorithm | call `authMiddleware` | `ApiError(401, ...)` — verification must pin the expected signing algorithm, not trust the token's own `alg` header (OWASP A02/A07 — a classic JWT library misconfiguration) |

#### requireRole

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Role in allow-list | `req.user = { id, role: 'ADMIN' }` | call `requireRole('ADMIN')(req, res, next)` | `next()` called with no error |
| Role not in allow-list | `req.user = { id, role: 'STUDENT' }` | call `requireRole('ADMIN')(req, res, next)` | `ApiError(403, "Insufficient permissions")` |
| Multiple allowed roles | `req.user = { id, role: 'PARENT' }` | call `requireRole('STUDENT', 'PARENT')(req, res, next)` | `next()` called |
| Called without authMiddleware having run (no req.user) | `req.user` undefined | call `requireRole('ADMIN')(req, res, next)` | throws/handles gracefully as `ApiError(401, ...)` or `403`, not an unhandled `TypeError` on `req.user.role` — asserts the middleware defends against its own misuse in the chain |

---

### 9.3 Test Case Detail — errorHandler.middleware.test.ts

NFRs: NFR-007 (no internal detail leakage). **OWASP: A05:2021 – Security Misconfiguration, A09:2021 – Security Logging and Monitoring Failures.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| ApiError passed through | `err = new ApiError(404, "Not found", [])` | call `errorHandlerMiddleware(err, req, res, next)` | responds with the standard error envelope, `statusCode: 404`, `message: "Not found"`, `errors: []` |
| ApiError with field errors | `err = new ApiError(400, "Validation failed", [{ field: "email", message: "Invalid email" }])` | call handler | `errors` array passed through unchanged |
| Unrecognized/unexpected error | `err = new TypeError("Cannot read property 'x' of undefined")` | call handler | responds `500` with a generic message (e.g. `"Internal server error"`) — **the raw error message and stack are never included in the response body** |
| Unrecognized error is logged server-side | spy on the logger (`console.error` or the project's logger module) | call handler with a generic `Error` | assert the logger was called with the real error/stack — confirms the detail is captured for operators even though it's withheld from the client (OWASP A09 — don't silently swallow) |
| Zod validation error surfaced by validate.middleware | `err` shaped as thrown by `validate.middleware.ts` (`ApiError(400, ..., errors[])`, one entry per failed field) | call handler | `400` with `errors` populated field-by-field, matching §0.1/§0.2 exactly |
| No stack trace or SQL/Prisma error text ever reaches the client | `err` = a raw `PrismaClientKnownRequestError` (unwrapped, simulating a missed try/catch upstream) | call handler | responds generic `500`, and the response body is asserted to **not** contain the Prisma error's raw `meta`/`message` substrings — a direct regression test for accidental internals leakage (OWASP A05) |

---

### 9.4 Test Case Detail — validate.middleware.test.ts

NFRs: NFR-007. **OWASP: A03:2021 – Injection (schema validation is the first line of defense against malformed/oversized/type-confused input reaching a query).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid body passes through | a real minimal schema, e.g. `z.object({ body: z.object({ email: z.string().email() }) })` | call `validate(schema)(req, res, next)` with a matching body | `next()` called with no error; `req.body` is the parsed (and thus type-coerced/stripped) result, not the raw input |
| Invalid body throws 400 with field errors | same schema | call with `{ body: { email: "not-an-email" } }` | `ApiError(400, ..., errors: [{ field: "email", ... }])` — one entry per failed field |
| Multiple simultaneous field failures | a schema requiring 2+ fields | call with both fields invalid | `errors` array has 2 entries, not just the first (fail-fast-per-field is not the same as fail-fast-per-request — confirms Zod's `.parse` isn't called with an early-abort option that would hide the second error) |
| query/params validated independently of body | a schema validating only `params` | call with a valid body but an invalid `params.id` | rejected, confirms all three of `body`/`query`/`params` are actually checked, not just `body` |
| Unknown/extra fields are stripped, not silently accepted (mass-assignment guard) | a schema with `.strict()` or default Zod object stripping behavior, e.g. `z.object({ body: z.object({ email: z.string() }) })` | call with `{ body: { email: "a@b.com", role: "ADMIN" } }` | either rejected (if `.strict()`) or `req.body` after parsing contains no `role` key — confirms a client cannot smuggle an unexpected privileged field (e.g. `role`, `isVerified`, `userId`) past validation into a service call (OWASP A08:2021 – Software and Data Integrity Failures / classic mass-assignment) |
| Oversized payload does not crash the process | a schema with a `.max()` string length constraint (e.g. `content: z.string().min(1)` from `publishPolicySchema`) | call with a multi-megabyte string | rejected by schema length constraint (if one exists) or handled without an unhandled exception — flags to whoever implements body-size limits that this is enforced at the schema layer only where a max length is explicitly declared, not implicitly everywhere (see 9.7 open item) |

---

### 9.5 Test Case Detail — jwt.test.ts

FRs: FR-SP-003. NFRs: NFR-007, NFR-008. **OWASP: A02:2021 – Cryptographic Failures, A07:2021 – Identification and Authentication Failures.**

#### signAccessToken

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Signs a valid token | — | call `signAccessToken({ id: 'u1', role: 'STUDENT' })` | returns a JWT string with 3 dot-separated segments; decoding it (without verification) shows `id`/`role` in the payload |
| Token does not embed the password or any sensitive field | — | call `signAccessToken({ id, role })` | decoded payload contains only `id`/`role` (plus standard `iat`/`exp`) — never a password hash or full user object |
| Token has a bounded expiry | — | call `signAccessToken`, decode payload | `exp` is present and set to a fixed, short-to-medium duration from `iat` — not absent (a non-expiring token) |

#### verifyAccessToken

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid, unexpired token verifies | sign a token with `signAccessToken` | call `verifyAccessToken(token)` | resolves/returns the decoded `{ id, role }` payload |
| Expired token throws | sign a token with a forced past expiry (e.g. via a manual `jwt.sign` call with `expiresIn: '-1s'` in the test) | call `verifyAccessToken(token)` | throws — caller (`authMiddleware`) is responsible for catching, not this function |
| Tampered signature throws | sign a token, then mutate one character of the signature segment | call `verifyAccessToken(tamperedToken)` | throws |
| Token signed with a different secret is rejected | sign a token using a different secret than `JWT_SECRET` | call `verifyAccessToken(token)` | throws — confirms the secret is actually enforced, not just "any well-formed JWT accepted" |
| Token signing algorithm is pinned | sign a token using `HS256`; attempt to verify a token with `alg` header changed to `none` and no signature | call `verifyAccessToken` | throws — verification must explicitly restrict accepted algorithms rather than trusting the token's own header (OWASP A02 — the canonical "alg: none" JWT bypass) |

---

### 9.6 Test Case Detail — password.test.ts

NFRs: NFR-008. **OWASP: A02:2021 – Cryptographic Failures.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Hash then compare succeeds for the correct password | — | `hashPassword("mypassword123")` then `comparePassword("mypassword123", hash)` | resolves `true` |
| Compare fails for the wrong password | — | `hashPassword("mypassword123")` then `comparePassword("wrongpassword", hash)` | resolves `false`, does not throw |
| Hash is never plain text and is non-deterministic per call | — | `hashPassword("samepassword")` called twice | the two resulting hashes differ (bcrypt salts each call), and neither equals the plain input — a direct regression test against NFR-008's "never store plain-text passwords" |
| Hash uses a real bcrypt cost factor, not a stub | — | inspect `hashPassword`'s output | hash string is a well-formed bcrypt hash (`$2a$`/`$2b$` prefix with a two-digit cost factor read from config, not hardcoded to a trivially low value like `$2b$04$` in production config) — flags to implementer that the salt-rounds config value should itself be reviewed, not asserted to a specific number here since that's a config concern, not a function-behavior one |

---

### 9.7 Test Case Detail — pagination.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Defaults applied when page/limit omitted | — | `paginate(undefined, undefined)` | returns `{ skip: 0, take: 20, page: 1, limit: 20 }` per §0.6's `page=1&limit=20` default |
| Custom page/limit computed correctly | — | `paginate(3, 10)` | `{ skip: 20, take: 10, page: 3, limit: 10 }` |
| Limit clamped at the documented max (100) | — | `paginate(1, 500)` | `limit` clamped to 100, not passed through raw — a client-supplied unbounded `limit` must not be able to force a full-table scan (ties NFR-004/NFR-003 performance protection under load) |
| Negative or zero page/limit rejected or floored | — | `paginate(-1, 0)` | normalized to the minimum valid value (`page: 1`, `limit` floored to at least 1), never producing a negative `skip` |
| paginatedResponse wraps correctly | `items = [...]`, `total = 47`, `page = 2`, `limit = 20` | call `paginatedResponse(items, total, page, limit)` | returns `{ items, page: 2, limit: 20, total: 47 }` exactly, matching the shape used across every list endpoint in Docs 06-api 01–08 |

---

### 9.8 Test Case Detail — auth.service.test.ts

FRs: FR-SP-001–005, FR-TU-001–002, FR-AC-002, FR-AC-005. NFRs: NFR-007, NFR-008. **OWASP: A07:2021 – Identification and Authentication Failures, A01:2021 – Broken Access Control (grade-routing rule), A04:2021 – Insecure Design (account-enumeration resistance).**

#### registerUser

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful student registration (Grade 6–12) | mock `prisma.$transaction` to create `User` + `StudentProfile` | call `registerUser('STUDENT', { email, password, grade: 9, termsAccepted: true })` | resolves `{ userId, role: 'STUDENT', studentProfileId, grade: 9, accountStatus: 'ACTIVE', verificationRequired: true }` per API spec §1.2 |
| Grade 1–5 rejected on the student self-registration path | — | call `registerUser('STUDENT', { ...valid, grade: 3 })` | throws `ApiError(400, "Students in Grades 1–5 require a parent-initiated account — see /guardianship/students")` — FR-AC-002/005 routing rule enforced in the service, not the schema |
| Successful parent registration | mock `prisma.$transaction` | call `registerUser('PARENT', { email, password, termsAccepted: true })` | resolves `{ userId, role: 'PARENT', parentProfileId, onboardingStatus: 'PENDING', verificationRequired: true }` |
| Successful tutor registration | mock `prisma.$transaction` | call `registerUser('TUTOR', { email, password, termsAccepted: true })` | resolves with `verificationStatus: 'PENDING'` — tutor is not visible/matchable yet (this function's job ends at account creation per Doc 8-1) |
| Duplicate email/phone | mock the transaction to reject with a Prisma unique-constraint error | call `registerUser` with an already-used email | throws `ApiError(409, "An account with this email/phone already exists")` |
| User + profile created atomically | mock `prisma.$transaction`; spy on its call | call `registerUser` | assert both the `User` create and the role-specific profile create were passed as one transaction array — a partial failure must not leave an orphaned `User` row with no profile |
| Verification code dispatched after successful registration | mock a successful transaction; spy on `notification.service.dispatchNotification` (or the sms/email client, per whichever layer actually issues it) | call `registerUser` with only `email` supplied | dispatch called via the email channel, not SMS |
| Verification code dispatched via SMS when only phone supplied | same as above, phone-only input | call `registerUser` | dispatch called via the SMS channel |
| `termsAccepted !== true` never reaches the service (schema-level 400) | — | (schema test, not service test — cross-referenced here for completeness) | see `auth.schema.ts` coverage under 9.4-equivalent schema tests; the service itself trusts that `termsAccepted` is always `true` by the time it's called, per FR-SP-005/FR-SC-001 |
| Password is never persisted in plain text | mock `prisma.user.create`; spy on its call args | call `registerUser` with `password: "plaintext123"` | assert the `create` call's `passwordHash` field is a bcrypt hash, not the literal string `"plaintext123"` — a direct regression test against NFR-008 at the exact point of storage |

#### login

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid credentials (by email) | mock `prisma.user.findFirst` to find a user by email; `bcrypt.compare` → `true` | call `login(email, password)` | resolves `{ accessToken, user: AuthUserDTO }` |
| Valid credentials (by phone) | mock lookup by phone instead | call `login(phone, password)` | resolves the same shape — confirms the identifier is checked against both email and phone, not just one |
| Identifier not found | mock `findFirst` → `null` | call `login(unknownIdentifier, anyPassword)` | throws `ApiError(401, "Invalid email/phone or password")` |
| Wrong password | mock `findFirst` to find a user; `bcrypt.compare` → `false` | call `login(identifier, wrongPassword)` | throws the **identical** `ApiError(401, "Invalid email/phone or password")` — same status and message as the not-found case, asserted explicitly (this is the specific case Doc 8-1 calls out as needing coverage, and the easiest one to accidentally get subtly wrong) |
| Timing-hardening — bcrypt.compare still runs when identifier not found | mock `findFirst` → `null`; spy on `bcrypt.compare` | call `login(unknownIdentifier, anyPassword)` | assert `bcrypt.compare` was still invoked (against a dummy/constant hash) even though no real user row was found — a service that short-circuits before this call would pass the message-only test above while still leaking timing information (OWASP A07 — user-enumeration via response-time side channel) |
| Token issued with the correct user id and role | mock a valid login; spy on `signAccessToken`'s call args | call `login(identifier, password)` | assert `signAccessToken` was called with `{ id: user.id, role: user.role }` — not an email, not a placeholder |
| Password/hash never included in the resolved result | mock a valid login | call `login(identifier, password)` | assert `result.user` has no `passwordHash`/`password` field |
| Suspended/restricted account still returns the identical generic error | mock `findFirst` to find a user with `accountStatus: 'SUSPENDED'` | call `login(identifier, correctPassword)` | **open item, flag for implementer confirmation:** Doc 08 does not explicitly state whether a suspended account gets the identical 401 or a distinct message. Given the account-enumeration-resistance pattern already established for the not-found/wrong-password cases, this doc recommends the identical `ApiError(401, ...)` (never confirming account existence or state to an unauthenticated caller) — but this should be confirmed against `adminPeople.service.ts`'s suspension behavior (Doc 8-2) before finalizing this test, since a mismatch here would be a genuine security decision, not a guess to silently bake into the suite |

#### logout

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Resolves as a no-op | — | call `logout()` | resolves `undefined`/void — confirms this is a deliberate stateless-JWT no-op (Doc 8-1), not an unimplemented stub that should have written something |

#### requestPasswordReset

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Matching identifier | mock `findFirst` to find a user | call `requestPasswordReset(identifier)` | resolves successfully; dispatch call made to send a reset code |
| Non-matching identifier | mock `findFirst` → `null` | call `requestPasswordReset(unknownIdentifier)` | resolves successfully with **the same outward shape** — no error, no distinguishing return value; this is the account-enumeration-resistance case Doc 8-1 explicitly calls out |
| Downstream SMS/email provider failure does not change the response shape | mock a matching user; mock the sms/email client to reject | call `requestPasswordReset(identifier)` | still resolves successfully (the failure is swallowed/logged, not propagated) — a thrown error here would create a response-shape difference an attacker could use to fingerprint provider failures against real vs. fake identifiers |
| Reset code is single-use and time-bounded | mock a matching user; spy on the code-generation call | call `requestPasswordReset(identifier)` twice in succession | the second call generates a new code; the first code should no longer validate in `resetPassword` (cross-referenced with the case below) |

#### resetPassword

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid, unexpired, unused code | mock a matching reset-code record | call `resetPassword(userId, code, newPassword)` | resolves; `User.passwordHash` updated to a hash of `newPassword`; the code is invalidated (single-use) |
| Expired code | mock a reset-code record past its expiry | call `resetPassword(userId, expiredCode, newPassword)` | throws `ApiError(400, "This reset link is no longer valid — request a new one")` |
| Already-used code | mock a reset-code record already marked used | call `resetPassword(userId, usedCode, newPassword)` | throws the same `ApiError(400, ...)` |
| Code belongs to a different userId | mock a valid code record for `userId: 'A'` | call `resetPassword('B', thatCode, newPassword)` | throws the same generic `ApiError(400, ...)` — never a distinct "code doesn't belong to you" message that would confirm code validity to the wrong caller |
| New password is hashed, never stored in plain text | mock a valid reset | call `resetPassword(userId, code, "newplain123")` | assert the persisted update uses a bcrypt hash of `"newplain123"`, not the literal string |

#### verifyContact / resendVerification

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid code verifies email | mock a matching, unexpired code issued for the email channel | call `verifyContact(userId, code)` | resolves `{ emailVerifiedAt: <timestamp> }`; `phoneVerifiedAt` untouched |
| Valid code verifies phone | mock a matching code issued for the phone channel | call `verifyContact(userId, code)` | resolves `{ phoneVerifiedAt: <timestamp> }` |
| Invalid or expired code | mock no matching code / an expired one | call `verifyContact(userId, badCode)` | throws `ApiError(400, "Invalid or expired code — request a new one")` |
| resendVerification regenerates without penalizing the original attempt | mock the existing code lookup | call `resendVerification(userId)` | a new code is generated/dispatched; no lockout or failure state is introduced by a prior failed/expired attempt |
| No server-side rate limit on resend (documented gap, not silently assumed) | — | call `resendVerification(userId)` many times in a row within the unit test | **flagged, not implemented:** per Doc 8-1, no rate limit is enforced server-side on this endpoint — do not write a test asserting throttling behavior that doesn't exist in the spec; instead, see 9.11's open-item note recommending this be revisited as an OWASP A04 (Insecure Design) / resource-exhaustion concern before launch |

---

### 9.9 Test Case Detail — auth.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| register (student route) delegates with the fixed role | `vi.mock` auth.service; `registerUser` resolves a DTO | call controller bound to the `/register/student` route with `req.body` | `registerUser` called with `'STUDENT'` (not read from `req.body`) and `req.body`; responds `201` |
| register (parent/tutor routes) fix their own role the same way | same pattern | call the parent and tutor controller variants | `registerUser` called with `'PARENT'` / `'TUTOR'` respectively — confirms role is never client-suppliable (OWASP A01 — a client sending `role: 'ADMIN'` in the body must have no effect, since the route itself pins the role argument) |
| login delegates and responds 200 | mock `login` resolves `{ accessToken, user }` | call controller with `req.body.identifier`, `req.body.password` | `login` called with both; responds `200` with `{ accessToken, user }` |
| login propagates 401 unchanged | mock `login` to throw `ApiError(401, "Invalid email/phone or password")` | call controller | error passed through via `asyncHandler`, not swallowed or rewritten into a different message |
| logout delegates | mock `logout` resolves void | call controller (requires `authMiddleware` to have run, so `req.user` exists in the route test, not this one) | responds `200`, `{}` |
| forgotPassword always responds 200 regardless of service internals | mock `requestPasswordReset` resolves void (it never throws, per Doc 8-1) | call controller with any identifier | responds `200`, `{}` — the controller does no branching of its own on match/no-match |
| resetPassword propagates its 400 | mock `resetPassword` to throw `ApiError(400, "This reset link is no longer valid — request a new one")` | call controller | error passed through unchanged |
| verify (contact) delegates to the correct service function by route | mock `verifyContact` | call the `/verify-contact` controller variant | `verifyContact` called with `req.body.userId`, `req.body.code` |
| verify (resend) delegates to the correct service function by route | mock `resendVerification` | call the `/resend-verification` controller variant | `resendVerification` called with the appropriate identifier — confirms the two routes sharing one handler name genuinely branch to different service calls, not accidentally both calling the same one |

---

### 9.10 Test Case Detail — auth.routes.test.ts

**OWASP: A01:2021 – Broken Access Control (public/authenticated boundary), A05:2021 – Security Misconfiguration.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Registration/login/verify/reset routes require no auth | mock controller layer, no Authorization header | request each of `/register/student`, `/register/parent`, `/register/tutor`, `/login`, `/verify-contact`, `/resend-verification`, `/forgot-password`, `/reset-password` | all reach their controller mock (200/201), none return 401 |
| logout requires auth | no Authorization header | request `POST /auth/logout` | `401`, controller mock never invoked |
| Each register route validates against its own schema | mock controller layer | request `/register/student` with `grade` missing | rejected by `validate(registerStudentSchema)`, controller never called |
| register/student rejects an out-of-range grade at the schema layer if outside 1–12 entirely | mock controller layer | request `/register/student` with `grade: 15` | rejected by schema (`z.number().max(12)`) — distinct from the *service-layer* 400 for grade 1–5, which is syntactically valid but semantically routed elsewhere; confirms both layers of validation are exercised, not just one |
| login validates required fields | mock controller layer | request `/login` with `{ identifier: "" }` (missing password) | rejected by `validate(loginSchema)` |
| No route in this router accepts a body field it doesn't declare (mass-assignment smoke test at the HTTP layer) | mock controller layer | request `/register/student` with a body including an extra `role: "ADMIN"` or `accountStatus: "ACTIVE"` field alongside valid fields | request either rejected (if schema is `.strict()`) or reaches the controller with the extra field stripped from `req.body` — the controller mock's received `req.body` is asserted to not contain the injected field |

---

### 9.11 Test Case Detail — sms.client.test.ts / email.client.test.ts

**OWASP: A10:2021 – Server-Side Request Forgery (outbound HTTP calls to a third-party API — confirm the destination host is the fixed provider endpoint, not attacker-influenced), A02:2021 – Cryptographic Failures (API key handling).**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful send | mock the underlying HTTP client to resolve a success response | call `send(to, subjectOrTemplate, body)` | resolves `{ success: true, providerRef }` |
| Provider returns a failure/error status | mock the HTTP client to resolve a failure response (e.g. Geez SMS/Brevo error payload) | call `send(...)` | resolves `{ success: false }`, does not throw — callers (`notification.service.ts`) rely on this to write a `FAILED` `Notification` row rather than crash |
| Provider request times out or network fails | mock the HTTP client to reject | call `send(...)` | resolves `{ success: false }` (normalized), consistent with the failure case above — the wrapper does not leak a raw network exception to `notification.service.ts` |
| API key is never logged or included in a thrown error's message | spy on the module's logger, if any | call `send(...)` with a forced failure | assert no log line or error message contains the raw `GEEZ_SMS_API_KEY`/`BREVO_API_KEY` value |
| Destination endpoint is the fixed provider host, not derived from user input | inspect the HTTP client call's URL/config | call `send(to, subjectOrTemplate, body)` | assert the request target is the hardcoded Geez SMS/Brevo API base URL — none of `to`/`subjectOrTemplate`/`body` (all ultimately traceable to user-supplied registration data) can influence *where* the request is sent, only its payload (OWASP A10 — outbound SSRF-adjacent hygiene, since `to` originates from user input) |

---

### 9.12 Test Case Detail — notification.service.test.ts

FRs: FR-NO-001–011. **OWASP: A01:2021 – Broken Access Control (ownership check on markRead).**

#### dispatchNotification

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful delivery via SMS | `User.preferredNotificationChannel = 'SMS'`; mock `sms.client.send` → success | call `dispatchNotification(userId, type, payload)` | creates a `Notification` row (`status: 'QUEUED'` then `'SENT'`, `sentAt` set); `sms.client.send` called, `email.client.send` not called |
| Successful delivery via Email | `preferredNotificationChannel = 'EMAIL'` | call `dispatchNotification` | `email.client.send` called, `sms.client.send` not called |
| Delivery failure writes a FAILED row, never throws | mock the relevant client to resolve `{ success: false }` | call `dispatchNotification` | `Notification.status` set to `'FAILED'`; the function itself resolves normally — a caller (e.g. `auth.service.registerUser`) must never see this function throw |
| Correct default status on write | mock a pending send | call `dispatchNotification` | the row is initially written with `status: 'QUEUED'` — **not** `'PENDING'`, matching Doc 8-1's explicit correction of an earlier draft that used a non-existent enum value; a test asserting the wrong status here would silently reintroduce that bug |
| Never propagates an exception to the caller under any failure mode | mock the underlying client to throw (not just resolve failure) | call `dispatchNotification` | still resolves normally, `Notification.status: 'FAILED'` — confirms the swallow is unconditional, not narrowed to only the `{ success: false }` resolve path |

#### listForUser

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns paginated notifications for the caller only | mock `prisma.notification.findMany` scoped to `userId` | call `listForUser(userId, false, 1, 20)` | assert the Prisma query filter includes `userId` — a query without this filter would leak other users' notifications (OWASP A01) |
| unreadOnly filter applied | — | call `listForUser(userId, true, 1, 20)` | assert the query filter also includes `readAt: null` (or equivalent) |
| No notifications yet | mock `findMany` → `[]` | call `listForUser(userId, false, 1, 20)` | resolves `{ notifications: [], page, limit, total: 0 }`, not an error, per §0.3 |

#### markRead

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Owner marks their own notification read | mock `findUnique` to return a row with `userId` matching the caller | call `markRead(notificationId, userId)` | resolves `{ id, readAt }`; update call includes the same `userId` filter as a defense-in-depth ownership check, not just a pre-check-then-update race |
| Notification not found | mock `findUnique` → `null` | call `markRead(badId, userId)` | throws `ApiError(404, "Notification not found")` |
| Notification belongs to a different user (IDOR attempt) | mock `findUnique` to return a row with a different `userId` | call `markRead(notificationId, callerUserId)` | throws `ApiError(403, "Not authorized to modify this notification")` — this is the canonical IDOR test (OWASP A01): a caller must not be able to mark another user's notification read by guessing/incrementing an ID |

#### retryFailed

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Retries all FAILED rows | mock `prisma.notification.findMany({ where: { status: 'FAILED' } })` to return 3 rows | call `retryFailed()` | resolves `{ retried: 3 }`; each row re-dispatched via the same channel logic as `dispatchNotification` |
| No FAILED rows | mock `findMany` → `[]` | call `retryFailed()` | resolves `{ retried: 0 }`, no error |
| Retry cap respected (does not loop forever) | mock a row that has already failed at the configured max-attempt count | call `retryFailed()` | that row is **not** retried again and remains permanently `FAILED` — flagged per Doc 8-1 as a config constant to set at implementation time; this test should be written once that constant exists, and is listed here as a required case rather than silently omitted |

---

### 9.13 Test Case Detail — notification.controller.test.ts / notification.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listMyNotifications delegates with the caller's own id | mock service; `req.user.id` set by (mocked) `authMiddleware` | call controller | `listForUser` called with `req.user.id`, never a client-suppliable user id from query/body — confirms there is no `?userId=` override path that would let a caller read someone else's notifications |
| markAsRead delegates | mock service resolves `{ id, readAt }` | call controller with `req.params.id` | `markRead` called with `(req.params.id, req.user.id)`; responds `200` |
| markAsRead propagates 403/404 | mock service to throw each | call controller | both propagate unchanged, not rewritten |
| GET / requires auth | no Authorization header | request `GET /notifications` | `401` |
| PATCH /:id/read requires auth | no Authorization header | request `PATCH /notifications/123/read` | `401` |
| GET / validates query params | mock controller | request `GET /notifications?limit=not-a-number` | rejected by `validate(listNotificationsQuerySchema)` |

---

### 9.14 Test Case Detail — adminAnnouncement.service.test.ts

FRs: FR-AD-019.

#### composePlatformAnnouncement

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Sends to every user in the target roles | mock `prisma.user.findMany` to return N users matching `audienceRoles` | call `composePlatformAnnouncement(title, body, ['STUDENT', 'PARENT'], adminId)` | `dispatchNotification` called once per matched user; the `findMany` filter is asserted to include `role: { in: ['STUDENT', 'PARENT'] }` |
| Empty audience | mock `findMany` → `[]` | call `composePlatformAnnouncement` | resolves successfully with zero dispatch calls, not an error |
| Batching for large audiences (flagged, not a hard requirement) | mock `findMany` to return e.g. 5,000 users | call `composePlatformAnnouncement` | **open item, not implemented as a strict assertion:** Doc 8-1 notes sends "should be batched... a note for implementation, not a hard requirement from the source docs." This test should assert dispatch calls are not fired as a single unbounded `Promise.all` once a batching strategy is chosen at implementation time — until then, only assert that all N users are eventually notified, not the concurrency shape |
| Announcement record returned matches input | mock a successful send | call `composePlatformAnnouncement(title, body, roles, adminId)` | resolves an `AnnouncementDTO` reflecting `title`, `body`, `audienceRoles`, `createdById: adminId` |

#### adjustNotificationRules (list read, per Doc 8-1's naming note)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Lists announcements paginated | mock `prisma` query | call `listAnnouncements(1, 20)` | resolves `PaginatedAnnouncementsDTO` shape matching §0.6 |
| No announcements yet | mock query → `[]` | call `listAnnouncements(1, 20)` | resolves `{ items: [], page, limit, total: 0 }`, not an error |

---

### 9.15 Test Case Detail — adminAnnouncement.controller.test.ts / adminAnnouncement.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| createAnnouncement delegates with `req.user.id` as creator | mock service | call controller | `composePlatformAnnouncement` called with `req.user.id`, not a client-suppliable creator id |
| listAnnouncements delegates | mock service | call controller | `adjustNotificationRules`/list called with query pagination params |
| Both routes require Admin role | no Authorization header, then a valid non-Admin token | request `POST /admin/announcements` and `GET /admin/announcements` with (a) no token, (b) a Student token | (a) `401`; (b) `403` — a Student or Tutor JWT must never reach the controller here |

---

### 9.16 Test Case Detail — policy.service.test.ts

FRs: FR-SC-001, FR-AD-015.

#### getCurrentPolicy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns the highest version for the type | mock `prisma.policyDocument.findFirst` ordered by `version desc`, filtered by `type` | call `getCurrentPolicy('PRIVACY')` | resolves the latest version's `PolicyDocumentDTO` |
| No versions published yet | mock the query → `null` | call `getCurrentPolicy('REFUND')` | throws `ApiError(404, "Policy not yet published")` |
| Invalid policy type defensively rejected | call with a value outside the enum (simulating a raw path param bypassing route-level validation) | call `getCurrentPolicy('NOT_A_TYPE' as any)` | throws `ApiError(400, "Invalid policy type")` — defensive re-validation at the service layer even though the route schema should already catch this, per Doc 8-1's explicit note that this arrives as a raw path param |

#### publishNewVersion

| Case | Setup | Action | Expected result |
|---|---|---|---|
| First version for a type | mock max-version lookup → none exists | call `publishNewVersion('SAFETY', content, adminId)` | creates version `1` |
| Subsequent version increments correctly | mock max-version lookup → `3` exists | call `publishNewVersion('SAFETY', content, adminId)` | creates version `4` — not overwriting version 3 |
| Never updates or deletes a prior version | spy on Prisma calls | call `publishNewVersion` | assert only a `create` call was made, no `update`/`delete` against any existing `PolicyDocument` row — full history is preserved (ties NFR-010's data-retention posture, in the opposite direction: this data is deliberately *not* purged) |
| Repeated publishes across all 5 types version independently | mock separate max-version lookups per type | call `publishNewVersion` for `PRIVACY`, then `TERMS`, then `PRIVACY` again | the second `PRIVACY` call becomes version 2, unaffected by `TERMS` having its own version 1 — confirms versioning is scoped per `type`, not global |

---

### 9.17 Test Case Detail — policy.controller.test.ts / policy.routes.test.ts

**OWASP: A01:2021 – Broken Access Control.**

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getPolicy delegates and is public | mock controller layer, no Authorization header | request `GET /policies/PRIVACY` | `200`, not `401` |
| getPolicy propagates 404 | mock service to throw `ApiError(404, "Policy not yet published")` | call controller | error passed through unchanged |
| publishPolicy requires Admin | no Authorization header, then a Tutor token | request `POST /admin/policies` with (a) no token, (b) a Tutor token | (a) `401`; (b) `403` |
| publishPolicy validates body against the 5-value enum | mock controller layer, valid Admin token | request `POST /admin/policies` with `{ type: "MADE_UP", content: "..." }` | rejected by `validate(publishPolicySchema)`, controller never called |
| publishPolicy passes `req.user.id` as the creator | mock service | call controller with a valid Admin token | `publishNewVersion` called with `req.user.id`, not any client-suppliable field |

---

### 9.18 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The "identifier not found" and "wrong password" cases in `login` assert the **identical** `ApiError` object shape (status + message), not just "both return roughly 401-ish" — this is the case Doc 8-1 flags as easiest to subtly break.
- [ ] The timing-hardening case (`bcrypt.compare` still runs on a not-found identifier) is asserted via a spy on the call itself, not inferred from the error message alone.
- [ ] `requestPasswordReset`'s matching-vs-non-matching cases assert **identical outward behavior**, including when the downstream provider fails — a test that only checks the happy path would miss a response-shape leak on provider failure.
- [ ] `notification.service.dispatchNotification`'s error-swallowing is tested by making the mocked provider client actually throw/fail, not by asserting the function's return type merely allows success.
- [ ] `markRead`'s 403 (wrong owner) and 404 (not found) cases are tested as genuinely distinct branches, not collapsed into one generic "can't mark read" test — this is the project's canonical IDOR check and deserves its own explicit case.
- [ ] `auth.routes.test.ts`'s mass-assignment case actually asserts on what `req.body` contains when it reaches the (mocked) controller, not just that the HTTP call returned some status code.
- [ ] `Notification.status` default is asserted as the literal string `'QUEUED'`, not `'PENDING'` — Doc 8-1 explicitly flags this as a previously-wrong value; a test written from muscle memory (copying the wrong pattern from another project) could silently reintroduce the bug this doc already corrected.
- [ ] No test in this file hardcodes a JWT string, hash, or dispatch payload that isn't actually derived from the code under test's real call args.

---

### 9.19 Out of Scope for Automated Testing (and why)

- **`prisma/schema.prisma`, `prisma/seed.ts`** — declarative/bootstrap artifacts excluded from the mirrored-test-file rule per Doc 8-1's own note; seed idempotency (`upsert`, not `create`) is verified manually against a real dev database.
- **`src/jobs/scheduler.ts` and the interval-registration wrapper in `notificationRetry.job.ts`** — no business logic of their own (Doc 8-1); the logic they call is already covered via `notification.service.test.ts`'s `retryFailed` cases directly.
- **Real Geez SMS / Brevo network behavior** — `sms.client.ts`/`email.client.ts` are unit-tested against a mocked HTTP layer only; actual provider auth, deliverability, and response-shape drift need a manual or separately-tracked integration pass.
- **Real bcrypt/JWT cryptographic strength** (cost factor tuning, key rotation) — unit tests confirm the functions are *called* correctly and produce internally-consistent results; production-grade parameter choices (bcrypt rounds, `JWT_SECRET` length/entropy, key rotation policy) are a security-review/config concern, not a unit-test assertion.
- **Server-side rate limiting on login, resend-verification, or forgot-password** — **explicitly flagged as an open item**: no rate limit is documented anywhere in Docs 02/06/08 for these endpoints. This is a real OWASP A04:2021 (Insecure Design) / brute-force gap worth raising with whoever owns Section 15 (NFRs) before launch — this test suite does not fabricate a rate-limit test for a control that was never specified, but this doc records the gap rather than silently ignoring it (see 9.8's `resendVerification` note and the login brute-force risk generally).
- **CSRF** — not applicable in the traditional sense; this is a stateless Bearer-JWT API with no cookie-based session, so CSRF tokens are out of scope by design, not by oversight.

---

### 9.20 Non-Functional Requirements & A06:2021 — Out of Scope for This Suite (and why)

The following are deliberately not exercised anywhere in the 09 test-file docs (backend or frontend). Naming them here — rather than leaving them absent from every §9.0 traceability table — closes the one place this doc set's otherwise-consistent "document why, don't just skip" habit (§9.19 above, and the equivalent sections in Docs 9-2 through 9-8) fell silent.

| Requirement | What it says | Why it's out of scope for Vitest/RTL | Covered instead by |
|---|---|---|---|
| NFR-001 | Support current major desktop browsers | Cross-browser rendering/behavior can't be meaningfully asserted by a jsdom-based unit/component suite | A cross-browser pass via Playwright or BrowserStack, run separately from this suite |
| NFR-002 | Mobile web responsive layout | Responsive/viewport behavior is a real-browser layout concern, not a unit-test assertion | The same Playwright/BrowserStack pass as NFR-001, at representative mobile viewport widths |
| NFR-003 | Core pages load within 3s | Load-time is a real-network/real-build performance measurement, not something a mocked unit test can produce a meaningful number for | Lighthouse CI, run against built pages |
| NFR-005 | 99.5%+ uptime | Uptime is an operational/infrastructure property of the deployed system, not a property any single test run can assert | Uptime monitoring (e.g. a status-check/alerting service) on the deployed environment |
| NFR-006 | Maintenance avoids peak hours | This is a scheduling/operations policy, not application code with a testable code path | A documented maintenance-window policy, enforced procedurally, not via test |
| A06:2021 (Vulnerable and Outdated Components) | Using known-vulnerable dependencies | Dependency vulnerabilities are a property of `package.json`/lockfile contents at a point in time, not of application logic a unit test exercises | CI-integrated dependency scanning (e.g. `npm audit`, Dependabot/Snyk), not a test-file-level case |

NFR-004 (notification dispatch must not block the triggering action) is the one NFR that *is* exercised in this suite — see `notification.service.test.ts` (§9.12) — and is excluded from the table above accordingly.

---

### 9.21 Open Boundary-Condition Decisions (cross-doc punch list)

The following behaviors are correctly left as **"flagged, not hard-asserted"** in their owning docs, because Doc 02/08 don't specify a single required outcome. They're collected here so they get decided once, deliberately, before implementation starts — rather than three different engineers guessing differently:

1. **Re-awarding a badge the student/tutor already has** (`badge.service.ts`, Doc 9-6 §9.4) — no-op, upsert, or `409`?
2. **Re-closing an already-`CLOSED_BY_ADMIN` message thread** (`messaging`/`adminDispute` flow, Doc 9-5) — no-op or `409`?
3. **Reschedule boundary at exactly 12.0 hours before class** (Doc 9-4) — inclusive (still free to reschedule) or exclusive (counts as a same-day miss)?
4. **Jitsi-link submission at exactly 30 minutes before class** (Doc 9-4) — counted as on-time or late?

Whoever starts implementation should confirm each of these with the Doc 02 owner and record the chosen behavior back in the relevant Doc 08 function-level spec, so the "flagged" note in Doc 09 can be resolved into a hard-asserted test case.

---

**Next:** proceed to → [9-2. Backend Test Documentation: Accounts & Guardianship]
