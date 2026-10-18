# Research: Submit Leave Request

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-10-18
(revised 2026-10-18 after Tech Lead review)

Each entry records a decision, why it was chosen, and the alternatives considered. Status tags:

- **Approved**: confirmed by the Tech Lead review.
- **Recommended**: technical recommendation; no further approval requested unless stated.
- **Pending business approval**: proposed; must not be treated as policy until approved.

Environment check (2026-10-18): .NET SDK 10.0.401 and Docker 29.8 are installed on the
development machine.

---

## R1. Concurrency control: employee-level lock protocol (T1 — Approved)

**Problem**: Two submissions by the same employee can be processed at the same time. The spec
requires the outcome to equal sequential processing (FR-013): a balance for a leave type and year
is never over-committed (FR-011, FR-012), and no two accepted requests share a calendar date,
**even across leave types** (FR-014). Cross-type overlap means two submissions may touch
*different* balance rows, so protecting only the balance row is not enough.

**Decision**: serialize all writes that affect one employee's requests or balances with an
**update lock on that employee's row**, held until the transaction ends, plus database backstops.

### R1.1 SQL Server locking mechanism

- Lock statement (Infrastructure only): a primary-key seek on the employee row with table hints
  `UPDLOCK, ROWLOCK`, e.g., selecting `Id` and `IsActive` for the employee id.
- `UPDLOCK` takes an update (U) lock on the row and **holds it until the transaction commits or
  rolls back**, even under READ COMMITTED.
- U locks are incompatible with other U and X locks → a second protocol-following transaction for
  the same employee waits. U locks are compatible with shared (S) locks → plain readers (e.g., the
  read endpoint) are not blocked.
- `ROWLOCK` keeps the lock at row granularity so other employees are unaffected (the primary-key
  seek makes this the natural granularity; the hint documents intent).
- `HOLDLOCK`/SERIALIZABLE is **not** used: the row is known to exist, so key-range locks add
  nothing and increase deadlock risk. (This corrects the earlier draft.)
- Any ordinary update of the employee row (e.g., HR deactivating the employee or changing the
  manager) takes an X lock on the same row and is therefore serialized with submissions
  automatically.
- The lock statement re-reads `IsActive`; if the employee was deactivated between authentication
  and locking, the submission ends with `403 EMPLOYEE_NOT_FOUND` and nothing is written.

### R1.2 Transaction boundary (submit)

```text
outside transaction : authenticate → validate DTO → resolve employee by identity → resolve
                      active leave type (reference data)
BEGIN TRANSACTION (READ COMMITTED), inside EF execution strategy
  1. SET LOCK_TIMEOUT 5000 ms
  2. lock employee row (UPDLOCK, ROWLOCK)          ← serialization point
  3. read reference data: holidays in range, calendar-complete marker (B1, if approved)
  4. read balance (employee, type, year) and overlapping Pending/Approved requests
  5. Domain decision (all rules)                    ← rejection: ROLLBACK, return error
  6. SaveChanges once: INSERT LeaveRequest + UPDATE LeaveBalance (rowversion-checked)
COMMIT                                              ← lock released
after commit        : log outcome; return 201
```

No external calls (identity provider, notifications, HTTP) happen while the lock is held.

### R1.3 Lock ordering protocol (applies to every relevant write path)

| Rule | Statement |
|---|---|
| L1 | A transaction that creates or changes a leave request, or changes a leave balance, MUST lock the affected employee's row **first**, before reading any data used for its decision. |
| L2 | Employee-facing operations handle **one employee per transaction**. Batch operations (imports, year rollover) SHOULD also process one employee per transaction; if one transaction must cover several employees, it MUST lock them in ascending `EmployeeId` order before touching any other row. |
| L3 | After the employee lock, the transaction MAY read reference data (leave types, holidays) without extra locks, then read and write that employee's requests and balances. |
| L4 | A transaction MUST NOT acquire an employee lock after it has written to `LeaveRequest` or `LeaveBalance`. |
| L5 | No external I/O while holding the lock; keep transactions short. |
| L6 | Reference-data maintenance (leave types, public holidays) does not take employee locks and never writes requests or balances. |

**Write paths that MUST follow the protocol**:

| Write path | Feature | Status |
|---|---|---|
| Submit leave request | this feature | designed here |
| Approve / reject request (convert or release reservation) | approval (future) | must adopt L1–L5 |
| Cancel request (release reservation) | cancellation (future) | must adopt L1–L5 |
| Balance entitlement load, adjustment, year rollover | balance administration / HR integration (B3) | must adopt L1–L5 |
| Employee activation, deactivation, manager change | HR integration (B3) | serialized automatically (X lock on the same row) |

**Enforcement**: the lock is not a separate optional call. Application exposes a single
unit-of-work operation "execute for employee *X* in a transaction"; its Infrastructure
implementation always starts the transaction, sets the lock timeout, and takes the employee lock
before running the use-case work. Write use cases cannot obtain a transaction any other way. Each
new write path adds a concurrency integration test (see R13), and code review checks L1–L6.

### R1.4 Deadlocks, timeouts, and backstops

- Protocol followers cannot deadlock with each other: each waits on a single row lock taken first.
- A deadlock with a non-follower (SQL error 1205) is treated as transient by the SQL Server EF
  execution strategy, so the whole unit is retried from the start.
- Lock wait longer than 5 s raises error 1222 → rolled back → `503 SUBMISSION_TIMEOUT`, not
  retried automatically. Nothing is saved.
- Backstops for non-followers and bugs:
  - check constraint `UsedDays + ReservedDays <= EntitlementDays` and non-negative values on
    `LeaveBalance` → over-reservation is impossible at database level;
  - `rowversion` on `LeaveBalance` and `LeaveRequest` → a lost update becomes
    `409 CONCURRENCY_CONFLICT`; nothing saved; safe to retry.

### R1.5 Worked example (spec US2 #4)

Employee has 5 available days; requests A and B (3 days each) arrive together. A takes the
employee lock; B waits at step 2. A reads available = 5, reserves 3, commits (available = 2). B
acquires the lock, reads available = 2, Domain rejects B with `LEAVE_INSUFFICIENT_BALANCE`
(requested 3, available 2). Total reserved = 3 ≤ 5. If A and B share a date, B is rejected with
`LEAVE_OVERLAP` whatever their leave types.

### Alternatives considered

| Alternative | Why not chosen |
|---|---|
| Optimistic concurrency on `LeaveBalance` only (constitution default) | Misses cross-type overlap: two overlapping requests of different types update different rows and both commit. |
| Optimistic token on the Employee row bumped by every submission, plus bounded retry | Works, but needs a retry loop to turn conflicts into correct business errors and writes to Employee on every submission. |
| Unique index on a per-day table `(EmployeeId, Date)` for active requests | Strong DB-level overlap guarantee, but a second table every status change must keep in sync. Future option if overlap defects appear. |
| SERIALIZABLE isolation | Range locks on several tables; high deadlock risk. |
| `sp_getapplock` keyed by employee | Equivalent but invisible in the data model and easy to bypass by ordinary updates. |
| Conditional `UPDATE ... WHERE available >= @days` | Moves the balance rule into SQL (Principle VIII) and does not cover overlap. |

**Constitution note**: Principle V names optimistic concurrency as the default and allows another
mechanism if documented; recorded in plan.md → Complexity Tracking.

## R2. Transactions, EF Core execution strategy, and rollback

**Decision**: the unit of work from R1.3 wraps
`Database.CreateExecutionStrategy().ExecuteAsync(...)` around begin → lock timeout → lock → work →
single `SaveChangesAsync` → commit.

- SQL Server transient-fault retry (`EnableRetryOnFailure`) is enabled. EF Core rejects
  user-initiated transactions under a retrying strategy unless they run inside the strategy's
  `ExecuteAsync`, so each retry replays the whole unit including the lock. The change tracker is
  cleared at the start of each attempt.
- One `SaveChangesAsync` writes the new request and the balance change → they commit or roll back
  together (FR-018).
- Business rejections return before `SaveChangesAsync`; exceptions and cancellation dispose the
  uncommitted transaction → rollback. No request or reservation remains (FR-015, FR-018a).

**Alternatives**: `TransactionScope` (ambient, async pitfalls); implicit `SaveChanges`
transaction only (cannot hold the lock across reads and writes).

## R3. Working-day calculation and public holidays

**Decision**: a pure Domain service counts dates in `[start, end]` that are Monday–Friday and not
public holidays. The range is limited to one calendar year (FR-020), so day-by-day iteration
(≤ 366) is simple and fast.

- Weekend = Saturday and Sunday (A12, **unconfirmed**), a named Domain constant.
- Holidays on weekends are not counted twice; no substitute day (spec edge case).
- Counted days are stored on the request at submission (FR-006); recounting after later calendar
  edits is out of scope (spec: deferred).
- Calendar completeness: see R16 (B1, pending business approval).

## R4. Overlap detection

**Decision**: Domain rule over calendar dates (FR-014): ranges overlap when
`existing.Start <= new.End AND new.Start <= existing.End`. Candidates: the employee's Pending or
Approved requests of any leave type, loaded by an indexed query restricted to the new range. The
Domain policy decides and returns the overlapping request's id. Concurrency safety comes from R1.

## R5. Business-rule failure signalling

**Decision**: Domain and Application return an explicit result (success, or a typed error with a
stable code and data such as requested/available days). Exceptions are for unexpected faults
only. Small hand-written result type; no library.

## R6. HTTP status codes and error format (T2 — Approved)

**Policy**:

- `422 Unprocessable Content` — the request is well-formed but violates a **documented business
  rule** (each has a stable `errorCode`).
- `409 Conflict` — a **genuine state or concurrency conflict**: the data changed while the request
  was processed (rowversion conflict), or an operation targets a resource whose current state no
  longer allows it (used by future approval/cancellation, e.g., approving a non-Pending request).
- `400` — input shape/format problems, all field errors together.

| Situation | Status | errorCode |
|---|---|---|
| Created | `201` + `Location` | — |
| Missing fields, invalid formats, end < start, reason too long, malformed JSON | `400` | — (field errors) |
| Not authenticated | `401` (Problem Details body + `WWW-Authenticate`) | — |
| No active employee for the authenticated identity | `403` | `EMPLOYEE_NOT_FOUND` |
| Unknown/inactive leave type, start in past, spans leave years, zero working days, overlap, balance not available, insufficient balance; holiday calendar unavailable (B1, if approved) | `422` | see [contracts/errors.md](./contracts/errors.md) |
| Rowversion conflict | `409` | `CONCURRENCY_CONFLICT` |
| Lock wait / database timeout | `503` | `SUBMISSION_TIMEOUT` |
| Unhandled fault | `500` | — |

All errors are RFC 9457 Problem Details with `traceId`; `422`/`409`/`403`/`503` add `errorCode`
and rule-specific extensions. Codes are part of the contract (Constitution VI).

Overlap is a documented business rule (FR-014), so it is `422`, not `409`.

An unknown or inactive leave type is a business rule (`422 LEAVE_TYPE_INVALID`), not a field
error: validity depends on current data, and the lookup happens after input validation, so it
could not be reported in the same `400` (analysis finding I2). Empty-body error responses
(JWT 401 challenge, policy 403, unknown route 404) receive a Problem Details body through
`UseStatusCodePages()` (analysis finding C1).

## R7. Request validation in Minimal APIs

**Decision**: .NET 10 built-in Minimal API validation (`AddValidation()`; DataAnnotations and
`IValidatableObject` on the request DTO) → `400` with all field errors.

- Dates are received as strings and strictly parsed as `yyyy-MM-dd` during validation so a bad
  date does not abort deserialization and hide other field errors (FR-016, US3 #4–5).
- `reason`: required, trimmed, not whitespace-only, max 500 characters (A5, **unconfirmed**).
- Messages never echo the reason text.
- Unknown JSON properties (`employeeId`, `status`, `balance`, …) are ignored; the DTO has no such
  fields (FR-009).

**Alternatives**: FluentValidation (extra dependency); rejecting unknown properties (stricter but
brittle for clients; can be enabled later).

## R8. Authentication and subject-to-employee mapping (T4)

**Dependency**: an OpenID Connect / OAuth 2.0 identity provider that issues JWT access tokens for
this API (e.g., Microsoft Entra ID). **Not yet selected** — required before production.

**Token validation** (`Microsoft.AspNetCore.Authentication.JwtBearer`): issuer, audience,
lifetime, and signature validated against the provider's published signing keys (authority
metadata). Inbound claim-type mapping is disabled so claim names are used exactly as issued.

**Mapping rule** (authenticated subject → employee):

1. Read the **issuer** (`iss`) and the configured **subject claim** from the validated token.
   The claim must be stable, unique per person within the issuer, and never reassigned.
   - Default: `sub`.
   - Microsoft Entra ID: use `oid` (the user's immutable object id within the tenant); `sub` is
     pairwise per application and would change if the API's app registration changed.
2. Look up the employee by the pair `(IdentityIssuer, IdentitySubject)` — a unique key on
   Employee. Including the issuer prevents collisions if a second provider or tenant is ever
   trusted.
3. Exactly one **active** employee must match → that employee is the actor. No match or inactive →
   `403 EMPLOYEE_NOT_FOUND` (US4 #4). Multiple matches are impossible (unique key).
4. Mutable attributes (email, UPN, display name) are **never** used for matching.
5. No employee identifier from the body, route, query, or headers is ever used (FR-008).
6. The mapping values on Employee are provisioned by the employee data source (B3); this feature
   only reads them.

**Development and tests**: `dotnet user-jwts` issues local tokens (issuer and `sub` configurable)
for manual runs; integration tests replace the JWT handler with a test authentication handler
that sets issuer and subject per test.

**Configuration**: `Authentication:Authority`, `Authentication:Audience`,
`Authentication:SubjectClaim` (default `sub`), validated at startup.

## R9. Time and time zone

**Decision**: inject `TimeProvider`. "Today" for the past-date rule (A4, **unconfirmed**) uses a
configured organisation time zone (A10), `LeaveOptions.OrganisationTimeZone` (IANA id),
validated at startup — the application refuses to start without it. Submission timestamps are
stored as UTC `datetimeoffset`.

**Dependency**: the time-zone value (B4) — required before production.

## R10. Persistence mapping and balance precision (T7 — reassessed)

**Decision**: EF Core 10 with SQL Server; `DateOnly` → `date`; enums stored as strings;
`rowversion` on `LeaveBalance` and `LeaveRequest`; check constraints and indexes as in
[data-model.md](./data-model.md). Migrations live in Infrastructure and are applied by a deliberate
step (CLI or pipeline), never automatically at startup in production.

**Balance precision — reassessed**: balance figures are stored as **whole-day integers** (`int`),
replacing the earlier `decimal(5,2)` proposal.

- Requests are whole days only (A2), so reserved and used days are always integers.
- The only source of fractions would be entitlements (pro-rating for joiners, part-time). Those
  come from outside this feature (A3, B3); no requirement says they can be fractional.
- With `decimal`, the system would also need a rule for requesting against fractional availability
  (e.g., 2.5 days available, 3 requested), which is a business policy that has not been decided.
- Integers make the rules, constraints, tests, and API (`availableDays: integer`) simpler
  (Principle II: no speculative generality).

**Business dependency**: if A2 changes (half-day leave) or the entitlement source (B3) supplies
fractional entitlements, switch to `decimal(5,1)` or similar. That change touches the schema
(widening, non-destructive), Domain arithmetic, and the API contract (`integer` → `number`,
a breaking change requiring a new API version or a coordinated client update). Until then, any
data load with a fractional entitlement must be rejected by the loader, not silently rounded.

## R11. Data access abstraction

**Decision**: narrow Application ports implemented with EF Core in Infrastructure:

- current-employee lookup by `(issuer, subject)`,
- unit of work "execute for employee in a transaction" (owns the lock, R1.3),
- leave submission store (leave type, balance for type/year, overlapping active requests, add
  request),
- own-request query (by id **and** employee id, R14),
- public holiday calendar (holidays in range; completeness check if B1 is approved).

No generic repository or MediatR (Principle II). The constitution forbids EF Core in Application,
so these ports are the minimum abstraction required.

## R12. Logging and sensitive data

- Source-generated `LoggerMessage` events with fixed event IDs: request submitted, submission
  rejected (with `errorCode`), concurrency conflict, lock timeout, authorization denied, holiday
  calendar unavailable (warning, B1).
- Logged properties: `LeaveRequestId`, `EmployeeId` (internal GUID), `LeaveTypeCode`,
  `CountedDays`, `LeaveYear`, `ErrorCode`, `TraceId`. Never: reason text, names, emails, tokens,
  identity subjects, request bodies.
- Success is logged **after commit** so a rolled-back submission is never logged as accepted.
- `EnableSensitiveDataLogging` off everywhere except explicit local opt-in; HTTP body logging not
  enabled.
- Trace id in log scope and every Problem Details response.
- Health checks: `GET /health/live`, `GET /health/ready` (includes SQL Server), implemented as
  Minimal API endpoints so they appear in the OpenAPI document; anonymous by design and
  documented as such in contracts/openapi.yaml (Constitution IV, VI).
- Automated test: a distinctive reason string never appears in captured logs.

## R13. Testing infrastructure and prerequisites (T5)

**Decision**:

- xUnit for all tests. Two projects: `UnitTests` (no external dependencies) and
  `IntegrationTests` (requires Docker).
- Integration tests use Testcontainers for .NET (`MsSql` module) with one SQL Server container per
  test run, migrations applied once, and data reset between test classes. The EF in-memory
  provider is not used (Constitution III).
- API tests: `WebApplicationFactory<Program>` with a test authentication handler.
- Concurrency tests: N submissions released together by a barrier, each on its own DbContext and
  connection, repeated several times per run; assert the sequential-equivalent outcome. Every
  future write path from R1.3 adds such a test.
- Architecture rule (Domain/Application reference no EF Core or ASP.NET Core) via reflection test.
- Performance (SC-001) via an on-demand load-test script, not CI.

**Local prerequisites**:

| Item | Requirement |
|---|---|
| .NET SDK | 10.0.x |
| Container runtime | Docker Desktop (Windows: WSL 2 backend, **Linux containers** mode) or Docker Engine on Linux. Testcontainers needs access to the Docker API. |
| Resources | ≥ 2 GB RAM available to Docker for SQL Server; ~2 GB disk for the image. |
| CPU architecture | x64. The SQL Server Linux image has no native ARM64 build; Apple Silicon needs x64 emulation (Rosetta in Docker Desktop) and is slower — confirm if any developer uses it. |
| Network | First run pulls the SQL Server image from `mcr.microsoft.com`. |
| Without Docker | Unit tests still run: `dotnet test tests/LeaveManagement.UnitTests`. |

**CI prerequisites**:

| Item | Requirement |
|---|---|
| Runner OS | Linux runner with Docker available (Windows-hosted runners generally cannot run Linux containers). |
| Image | SQL Server image pinned to an explicit tag that matches the production SQL Server major version (production version to be confirmed); pre-pull or cache to save time. |
| Docker access | Runner must allow containers to start. Testcontainers' resource reaper (Ryuk) needs the Docker socket; on restricted runners it can be disabled (`TESTCONTAINERS_RYUK_DISABLED=true`) if the job cleans up containers itself. |
| Pipeline order | build (warnings as errors) → unit tests → integration tests; both test stages gate merge (Constitution workflow). |
| Secrets | None needed: the test container's password is generated per run. |

**Dependency**: the CI platform has not been chosen; the requirements above apply to any platform.

## R14. Read endpoint for the employee's own request (T6)

**Decision**: include `GET /api/v1/leave-requests/{id}`.

- Requires authentication and the same subject-to-employee mapping as submit (R8); no active
  employee → `403`.
- **Resource-level authorization**: the query filters by request id **and** the caller's
  employee id in one database query. A request that does not exist or belongs to someone else
  returns `404` — identical responses, so other employees' request ids cannot be discovered.
- Managers and HR cannot read others' requests through this endpoint; manager/HR views belong to
  the approval feature.
- Read-only, no locks (U locks from submissions do not block it).
- Returns the same response schema as the `201` confirmation and is the target of `Location`.
- Request ids are random GUIDs (not sequential), as a defense in depth against enumeration.

## R15. Duplicate and retried submissions (T3 — Approved: no idempotency key)

**Guarantee**: a duplicate can never reserve balance twice.

- Two identical submissions in flight: the employee lock serializes them; the second sees the
  first's committed request and is rejected with `422 LEAVE_OVERLAP` (overlap is checked before
  balance, see data-model.md rule order).
- A failed submission leaves no request and no reservation (R2), so a retry starts clean.

**Client retry ambiguity (documented, accepted for v1)**: if the client loses the response (for
example, a timeout) it cannot tell whether the original succeeded.

| What the client does | Server outcome | Client must |
|---|---|---|
| Resends the identical request; original had succeeded | `422 LEAVE_OVERLAP` with `overlappingRequestId` | GET that id to confirm it is the original (same dates, type, Pending) |
| Resends; original had failed | `201` (normal success) | nothing special |
| Resends; original still in progress | Waits for the lock, then `422 LEAVE_OVERLAP` | as first row |
| Resends with **changed** dates that no longer overlap | A second, separate request is created | treat as a new request (it is one) |
| Resends; an unrelated existing request overlaps | `422 LEAVE_OVERLAP` naming that other request | GET shows different details → original did not exist |

Limits: a `422 LEAVE_OVERLAP` on retry does not by itself prove the original succeeded; the
client must compare details. This ambiguity is accepted for the initial version.

**Future consideration**: an `Idempotency-Key` header stored with the request, returning the
original `201` on replay. Additive and backward-compatible; revisit if client teams report retry
confusion or a non-interactive integration (bulk/automated submission) appears.

## R16. Missing or unverifiable holiday calendar (B1 — Pending business approval)

**Proposal (not yet policy)**: fail closed. Do not accept a request or reserve balance unless
the public-holiday calendar for the request's leave year is verifiably complete.

- **Completeness signal**: a per-year marker (`PublicHolidayCalendarYear`) recorded by whoever
  maintains the calendar once that year's list is complete (data source: B3). An existing
  holiday row without the marker is **not** proof of completeness.
- **Check**: inside the transaction, before counting days. The request is within one leave year
  (FR-020), so one marker is checked.
- **Marker missing** → `422 LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE` with `leaveYear`; nothing
  reserved. It is a documented rule the employee can act on (choose other dates or retry later),
  hence `422` per T2.
- **Calendar cannot be read** (database failure) → normal fault handling (`503`/`500`); never
  treated as "no holidays".
- **Operational warning**: a `Warning`-level log event `HolidayCalendarUnavailable` with
  `LeaveYear`, `EmployeeId`, `TraceId` — never the reason text — so operators can publish the
  missing calendar.
- **Consequence to accept**: until the next year's calendar is published, employees cannot book
  leave in that year, even if their balance exists (FR-020a). The calendar must be published
  before the first future-year booking is expected.

**If approved**: the spec needs a new functional requirement and acceptance scenario
(e.g., via `/speckit-clarify` or a spec edit) before implementation tasks for it are executed.
**If rejected**: the marker is not built and counting uses whatever holidays exist (risk of
over-reserving, documented in plan.md → Risks).

**Status**: design is ready; implementation is gated on business approval.
