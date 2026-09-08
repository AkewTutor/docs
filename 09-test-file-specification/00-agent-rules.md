## Project: AKEWTutor — Agent-Facing Binding Rules for Test Work

**Source:** Review §7 ("Rules to Give the AI Agent Directly"). **Status:** these are hard rules, not suggestions or defaults-with-exceptions. **Referenced by:** every `09-N-*.md` file (backend and frontend) — any agent working on any module doc, or on the code that doc specifies, reads this file first.

An agent that violates one of these rules has produced a process violation, not a lower-quality-but-acceptable result. Do not rationalize an exception to any rule below based on time pressure, an ambiguous spec, or a request from a human reviewer in-thread — if a rule genuinely seems wrong for a specific case, flag it and ask, rather than quietly deviating.

---

1. **Tests are written before implementation code, every time, no exceptions.** A source file with no corresponding failing test written first is a process violation, not a shortcut.

2. **Never weaken an assertion to make a test pass.** If a test fails, fix the implementation or flag the spec as wrong — do not loosen the expected value, remove a case, or change `toBe` to `toBeTruthy` to get green.

3. **Never delete or skip (`.skip`, `.todo`) a test to unblock a build without explicit human sign-off logged in the PR description.**

4. **Every new source file must appear in the relevant `09-*` doc's Test File Map before or alongside its test file** — no source file may exist without a corresponding row (even a "Not required — because X" row).

5. **Mocks must match the real interface.** When mocking Prisma, an external client, or an internal service, the mock's shape must be checked against the actual current type signature — a passing test against a stale mock shape is worse than no test.

6. **Every OWASP tag claimed in a test's description must correspond to an actual assertion in that test**, not just a comment — no decorative security tags.

7. **Any test marked "Integration (persistence)" must run against the real test database — it may not silently fall back to a mock if the test DB is unreachable; it must fail loudly.**

8. **E2E tests may not use `vi.mock()` or any unit-test mocking utility.** External third parties are faked only via sandbox mode or a dedicated mock HTTP server running as its own process.

9. **When a webhook or payment-status handler is touched, an idempotency test (same event delivered twice) is mandatory in the same PR.**

10. **Any endpoint touching money, guardian/student PII, or admin trust actions requires an explicit audit-logging test, not just a functional one.**

11. **Do not mark a Coverage Honesty Check box complete without listing, by filename, every source file it verified against the current codebase** — not against the doc's own memory of what should exist.

---

### How these interact with `00-test-fixtures.md`

Rule 5's "mocks must match the real interface" applies to the *test double* (the Prisma mock, the Chapa client mock) — it does not relax because the *entity data* fed into that mock came from a `00-test-fixtures.md` factory. Using a factory is necessary but not sufficient for rule 5 compliance; the mock's method signatures still need independent verification against the current source.

Rule 7's real-database requirement is what `00-test-fixtures.md §1.1`'s FK-override note exists to support: a persistence test that used only default (non-existent-row) FK values from a factory would technically "run against the real DB" per rule 7's letter while violating its spirit, since it would only ever exercise the constraint-violation path.

### Audit-log test convention (Rule 10)

No `AuditLog` entity exists yet anywhere in `04-database-and-data-model.md` — Rule 10's audit-logging requirement is a genuinely new cross-cutting write this test suite drives into existence (per Rule 1, the test is written before the implementation code it constrains), not something being retrofitted onto an existing table. So every Unit-tier test asserting "an audit log entry is written" targets the same assumed call shape, to keep 150+ agent-written test files from inventing incompatible ad-hoc shapes independently:

```
auditLog.service.record({ actor: string /* userId */, action: string /* e.g. 'REFUND_APPROVED' */, target: string /* entity id */, timestamp: Date })
```

- **Unit tier (this doc set's Phase 4 additions):** the test mocks `auditLog.service.ts` and asserts `record(...)` was called with the correct `actor`/`action`/`target` — it does not touch a database. This proves the call is *wired in*, not that it *persists*.
- **Integration (persistence) tier (Phase 5, `9-N-module-persistence.md`):** a separate, later test asserts the row actually lands in a real `AuditLog` table. A Unit-tier pass on its own is not evidence the persistence path works — see Rule 7.
- **`action` values** are short, past-tense, upper-snake-case strings scoped to the mutation they describe (`REFUND_APPROVED`, `REFUND_REJECTED`, `GUARDIAN_REMOVED`, `TUTOR_REJECTED`, `LOGIN_FAILED_THRESHOLD`, `DISPUTE_RESOLVED`) — an agent adding a new audit-logged mutation defines a new value here rather than reusing an unrelated one for convenience.
- This convention does not itself decide whether `AuditLog` is a new Prisma model, a structured log sink, or both — that is an implementation decision the code, not this test doc, makes. The test only pins the *service-layer call contract*, which is what every calling test file needs to agree on regardless of how the write is ultimately persisted.
