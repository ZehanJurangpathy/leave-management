# Implementation Plan: Submit Leave Request

**Branch**: `001-submit-leave-request` (no git repository yet) | **Date**: 2026-10-18
(revised after Tech Lead review) | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-submit-leave-request/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Authenticated employees submit a leave request (leave type, start/end date, reason). The server
maps the token to an employee, validates input, counts working days (Mon–Fri minus public
holidays), applies the leave rules (no past start, single calendar year, non-zero days, no
calendar-date overlap with Pending/Approved requests, balance exists and suffices), then creates the
request in `Pending` and reserves its days, all in one transaction. Requests from employees without
a manager are routed to the fallback reviewer (HR). Employees can read back their own request.

Technical approach: .NET 10 Clean Architecture (Domain, Application, Infrastructure, Api) with
Minimal API endpoints `POST /api/v1/leave-requests` and owner-only `GET /api/v1/leave-requests/{id}`.
Concurrency follows an **employee-level lock protocol** (approved T1): every write that affects an
employee's requests or balances first takes an update lock on the employee row, backed by
`rowversion` and a check constraint on the balance. Business-rule failures return `422`, genuine
conflicts `409` (approved T2). No idempotency key in v1 (approved T3). Details:
[research.md](./research.md).

## Technical Context

**Language/Version**: C# on .NET 10 (SDK 10.0.401 installed); nullable enabled; warnings as errors

**Primary Dependencies**: ASP.NET Core Minimal APIs (built-in validation, OpenAPI, Problem
Details, health checks), EF Core 10 + SQL Server provider,
`Microsoft.AspNetCore.Authentication.JwtBearer`

**Storage**: SQL Server (production major version to be confirmed); schema managed by EF Core
migrations in Infrastructure

**Testing**: xUnit; Testcontainers (`MsSql`) for integration tests against real SQL Server;
`Microsoft.AspNetCore.Mvc.Testing` (`WebApplicationFactory`) for API tests

**Target Platform**: Linux or Windows server / container hosting ASP.NET Core (hosting target not
yet chosen; not needed for this feature)

**Project Type**: Web service (REST API)

**Performance Goals**: 95% of submissions return an outcome in < 2 s end-to-end within the system
with up to 50 simultaneous submissions (SC-001)

**Constraints**: no over-reservation or overlapping accepted requests under concurrency (FR-013);
create + reserve atomic (FR-018); duplicates never reserve twice (FR-018a); reason text never
logged (FR-019); employee identity from token only (FR-008)

**Scale/Scope**: mid-sized organisation (up to a few thousand employees); one write endpoint and
one read endpoint

No `NEEDS CLARIFICATION` items remain in Technical Context. Unresolved items are listed under
[Decision Status](#decision-status); none blocks design, and those that block implementation or
production are marked as such.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | How the plan complies | Pre | Post |
|---|---|---|---|
| I. Clean Architecture Boundaries | Four projects with inward references only; Domain has no EF/ASP.NET packages; endpoints only map HTTP ↔ use case; reflection-based architecture test. | ✅ | ✅ |
| II. Readable, Purposeful Code | No MediatR, AutoMapper, or generic repository; narrow ports only where the layer boundary requires them (R11); small hand-written result type (R5); integer balances instead of speculative decimals (R10). | ✅ | ✅ |
| III. Tested Business Rules and Behavior | Unit tests for every Domain rule and both use cases; integration tests on real SQL Server (no in-memory provider); API tests incl. 401/403/404; concurrency tests for every protocol write path; `TimeProvider` injected. | ✅ | ✅ |
| IV. Server-Side Trust and Security | JWT validated against the identity provider; employee mapped from `(issuer, subject)` only; resource-level check on the read endpoint; DTO has no server-owned fields; validation at API + invariants in Domain; parameterized EF only; secrets outside source control. | ✅ | ✅ |
| V. Data Integrity and Concurrency Safety | Single transaction for create + reserve; employee lock protocol L1–L6 for all write paths; `rowversion` + check constraints as backstops; status transitions in Domain; migrations only. **Deviation**: pessimistic lock instead of default optimistic, documented in Complexity Tracking and approved (T1). | ⚠️ justified | ⚠️ justified |
| VI. Consistent, Documented API | `/api/v1`; `201` + `Location`; Problem Details everywhere with `errorCode` + `traceId`; status-code policy and full OpenAPI contract ([contracts/](./contracts/)). | ✅ | ✅ |
| VII. Safe, Actionable Observability | Source-generated structured log events; trace id in logs and errors; no reason/PII/subject in logs; success logged after commit; sensitive EF logging off; health checks; automated "reason not logged" test. | ✅ | ✅ |
| VIII. Explicit, Maintainable Design | Rules as named Domain methods/policies; constructor DI; options validated at startup (time zone, auth settings); no magic numbers. | ✅ | ✅ |
| Stack & Workflow gates | .NET 10, Minimal APIs, EF Core, SQL Server, xUnit as mandated; new packages justified (JwtBearer, Testcontainers, Mvc.Testing); CI gates documented (R13). | ✅ | ✅ |

**Gate result**: PASS (one documented and approved deviation).

## Project Structure

### Documentation (this feature)

```text
specs/001-submit-leave-request/
├── plan.md              # This file
├── research.md          # Phase 0: decisions, rationale, alternatives (R1–R16)
├── data-model.md        # Phase 1: entities, constraints, lifecycle, rule order
├── quickstart.md        # Phase 1: prerequisites, run, validation scenarios
├── contracts/
│   ├── openapi.yaml     # Phase 1: HTTP contract
│   └── errors.md        # Phase 1: status-code policy and error catalogue
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
LeaveManagement.slnx
Directory.Build.props            # net10.0, Nullable, TreatWarningsAsErrors, analyzers
Directory.Packages.props         # central package versions
.editorconfig

src/
├── LeaveManagement.Domain/                  # no package references to EF Core / ASP.NET Core
│   ├── Common/                              # Result, DomainError, error codes
│   ├── Employees/                           # Employee, ReviewRoute
│   ├── LeaveTypes/                          # LeaveType
│   ├── LeaveBalances/                       # LeaveBalance (Reserve, AvailableDays)
│   ├── LeaveRequests/                       # LeaveRequest, LeaveRequestStatus, LeaveDateRange,
│   │                                        # overlap policy
│   └── Calendar/                            # PublicHoliday, working-day calculator
├── LeaveManagement.Application/             # references Domain only
│   ├── Abstractions/                        # ports: current employee, employee unit of work,
│   │                                        # submission store, own-request query,
│   │                                        # holiday calendar; LeaveOptions
│   └── LeaveRequests/
│       ├── Submit/                          # command, handler (use case), result
│       └── GetOwn/                          # query, handler
├── LeaveManagement.Infrastructure/          # references Application + Domain
│   ├── Persistence/
│   │   ├── LeaveManagementDbContext
│   │   ├── Configurations/                  # entity mappings, constraints, indexes
│   │   ├── Migrations/
│   │   ├── EmployeeUnitOfWork               # transaction + lock timeout + employee lock
│   │   └── Stores/                          # other port implementations
│   ├── Development/                         # dev-only seed data
│   └── DependencyInjection
└── LeaveManagement.Api/                     # composition root; references all
    ├── Program
    ├── Endpoints/LeaveRequests/             # endpoint mapping + request/response DTOs
    ├── Authentication/                      # JWT setup, subject→employee accessor, policy
    ├── ErrorHandling/                       # domain error → Problem Details, exception handler
    └── Logging/                             # source-generated log event definitions

tests/
├── LeaveManagement.UnitTests/               # Domain/, Application/, Architecture/
└── LeaveManagement.IntegrationTests/        # Persistence/, Api/, Concurrency/; SQL container
                                             # fixture; test auth handler
```

**Structure Decision**: Clean Architecture with four production projects as mandated by
Constitution I. Two test projects (unit, integration) keep fast tests separate from
Docker-dependent ones; architecture rules live in the unit test project.

## Design

### 1. Domain entities and responsibilities

See [data-model.md](./data-model.md) for fields and constraints.

| Entity | Responsibility in this feature |
|---|---|
| `LeaveRequest` | Created via a factory enforcing its invariants (date order, single year, counted days > 0, reason present, initial `Pending`, review route). Future status transitions happen at most once. |
| `LeaveBalance` | Whole-day `AvailableDays`; `Reserve(days)` rejects when insufficient (requested/available in error). |
| `Employee` | Active flag; review route (`Manager` vs `FallbackReviewer`); concurrency anchor for the lock protocol; identity mapping `(IdentityIssuer, IdentitySubject)`. |
| `LeaveType` | Code, name, active flag. |
| `PublicHoliday` | Date + name; consumed by the working-day calculator. |
| Working-day calculator | Pure Domain service: range + holiday dates → counted days (R3). |
| Overlap policy | Calendar-date intersection against Pending/Approved requests of any type (R4). |

### 2. Application use cases

**Submit leave request** — input: issuer and subject from the authenticated principal, leave type
code, dates, reason (shape-validated). Output: created request or a typed error.

1. Resolve the active employee by `(issuer, subject)` → else `403 EMPLOYEE_NOT_FOUND`.
2. Resolve the active leave type by code → else `422 LEAVE_TYPE_INVALID`.
3. Execute for this employee in a transaction (R1.2): lock timeout, employee lock, re-check
   active.
4. Evaluate rules in the order fixed in
   [data-model.md](./data-model.md#submission-invariants-domain): start not in past → same
   calendar year → *(B1, only if approved)* calendar complete → working days > 0 → no overlap →
   balance exists → `Reserve(countedDays)`.
5. Create `LeaveRequest` (Pending, review route), save once, commit.
6. After commit: log `LeaveRequestSubmitted`; on rejection log `LeaveSubmissionRejected` with the
   error code only.

**Get own leave request** — resolve the employee as above, then query by request id **and**
employee id; not found → `404` (R14). No transaction or lock.

### 3. API endpoints, contracts, authentication, validation

- Contract: [contracts/openapi.yaml](./contracts/openapi.yaml); status-code policy and codes:
  [contracts/errors.md](./contracts/errors.md).
- `POST /api/v1/leave-requests` → `201` + `Location`. `GET /api/v1/leave-requests/{id}` → `200`,
  or `404` when missing or not owned (identical responses).
- Authentication (T4, R8): JWT bearer validated against the identity provider (issuer,
  audience, lifetime, signing keys). Subject → employee mapping: configured claim (`sub` by default,
  `oid` for Entra ID) plus issuer, matched to the unique `(IdentityIssuer, IdentitySubject)` on an
  active employee. Email/UPN never used. No match → `403`.
- Validation (R7): .NET 10 built-in Minimal API validation; dates as strings, strictly parsed;
  all field errors together; unknown properties ignored.
- Handlers only read the principal, map DTO ↔ command/result, and map errors to Problem Details.
- `UseStatusCodePages()` gives empty-body error responses (401 challenge, 403, 404) a Problem
  Details body with `traceId` (Constitution VI).
- Health endpoints `GET /health/live` and `GET /health/ready` are anonymous by design and are
  described in the OpenAPI contract (Constitution IV, VI).

### 4. Working-day calculation

Mon–Fri inclusive range minus public holidays, iterated per day (≤ 366 days within one calendar
year). Weekend set is a Domain constant (A12, unconfirmed). Counted days are a snapshot on the
request (R3).

### 5–7. Atomic reservation, concurrency, and overlap (T1 — approved)

**Guarantee**: for any set of simultaneous submissions by one employee, the accepted set is one
that could have resulted from processing them one at a time.

**Mechanism** (full detail in [research.md → R1](./research.md#r1-concurrency-control-employee-level-lock-protocol-t1--approved)):

1. Each submission opens a READ COMMITTED transaction, sets a 5 s lock timeout, and **first**
   takes an update lock (`UPDLOCK, ROWLOCK`) on the employee row, held until commit or rollback.
2. A second submission for the same employee blocks at step 1 before reading anything.
3. The lock holder reads balance and active requests, applies all rules in Domain, inserts the
   request, and increases `ReservedDays` in one `SaveChanges`, then commits.
4. The waiting submission then reads the **committed** state and is rejected with the correct
   business error (insufficient balance or overlap) when appropriate.

Both checks happen after the lock and before commit, so there is no check-then-act gap. Different
employees lock different rows and proceed in parallel. Plain readers are not blocked (U locks are
compatible with shared locks).

**Lock ordering protocol (L1–L6)**: lock the employee row first; one employee per transaction
(batches: one employee per transaction, or ascending `EmployeeId` before any other row); never
lock an employee after writing requests/balances; no external I/O under the lock; reference-data
maintenance takes no employee locks.

**Write paths bound by the protocol**: submit (this feature); approve/reject and cancel (future);
balance entitlement loads/adjustments/year rollover (B3 source). Employee updates by HR are
serialized automatically because they lock the same row. The protocol is enforced structurally:
write use cases can only obtain a transaction through the "execute for employee" unit of work,
which always takes the lock.

**Backstops**: check constraint `UsedDays + ReservedDays <= EntitlementDays` (non-negative);
`rowversion` → `409 CONCURRENCY_CONFLICT`. Deadlocks with non-followers (1205) are retried by the
execution strategy; lock timeout (1222) → `503 SUBMISSION_TIMEOUT`.

### 8. Transactions, SQL Server safeguards, rollback

- Transaction runs inside the EF Core execution strategy so transient-fault retries replay the
  whole unit, including the lock (R2).
- Rejections return before `SaveChanges`; exceptions dispose the uncommitted transaction →
  rollback. Integration tests inject a failure after reservation and assert no request and no
  reserved days remain.

### 9. Error handling and logging

- Domain errors → Problem Details per the approved policy (T2): `400` input, `403` no employee,
  `404` not owned, `409` genuine conflict, `422` business rule, `503` timeout, `500` generic.
- Logging per R12: fixed event IDs, internal IDs only, never reason text, identity subjects, or
  request bodies; success logged after commit; sensitive EF logging disabled.
- *(B1, only if approved)* `Warning` event `HolidayCalendarUnavailable` with `LeaveYear`,
  `EmployeeId`, `TraceId`.

### 10. Testing strategy

| Level | Project | Focus |
|---|---|---|
| Unit — Domain | UnitTests/Domain | Working days (single day, Fri–Tue with Monday holiday = 2, weekend-only = 0, holiday on weekend, full year); `Reserve` (exact, exceed, zero available); `LeaveRequest` invariants; overlap (shared weekend date blocks, adjacent allowed, Rejected/Cancelled ignored, cross-type blocks); review route. |
| Unit — Application | UnitTests/Application | Both use cases with hand-written fakes: each error code and order; nothing saved on rejection; future-year balance present/absent; past date with fixed `TimeProvider`; no-manager routing; overlap reported before balance on retry; inactive after lock → 403. |
| Architecture | UnitTests/Architecture | Domain/Application reference no EF Core or ASP.NET Core assemblies. |
| Integration — Persistence | IntegrationTests/Persistence | Mappings round-trip; unique and check constraints reject invalid rows (incl. over-reservation); overlap query; rollback after injected failure. |
| Integration — API | IntegrationTests/Api | Quickstart scenarios 1–18: status codes, Problem Details shape and codes, `Location`, 401/403, spoofed fields ignored, own-request `404` for others' ids, OpenAPI lists endpoints, reason absent from logs. |
| Integration — Concurrency | IntegrationTests/Concurrency | Two 3-day requests on 5 days → one `201`, one `422 INSUFFICIENT_BALANCE`, reserved = 3; overlapping different types → one `201`, one `422 OVERLAP`; identical duplicates → one request, one reservation; different employees not blocked; lock timeout → `503`; repeated runs. |
| Performance (on demand) | script, outside CI | 50 concurrent submissions, p95 < 2 s (SC-001). |

Prerequisites for running these locally and in CI: [research.md → R13](./research.md#r13-testing-infrastructure-and-prerequisites-t5).

### 11. External dependencies, assumptions, risks

**Dependencies**: SQL Server (version TBC); identity provider (TBD, R8); Docker for integration
tests (R13); reference data — employees with identity mapping and manager links, leave types,
balances, holidays — supplied outside this feature (B3); organisation time zone (B4).

**Risks**:

| Risk | Impact | Mitigation |
|---|---|---|
| No production source yet for employees, identity mapping, balances, holidays (B3) | Feature works only with seeded data | Production-blocking dependency; dev seed for now |
| Holiday calendar missing for a year while B1 is unapproved | Counted days include holidays → over-reservation | B1 proposal (fail closed) awaiting business approval |
| Future write paths bypass the lock protocol | Concurrency defects in approval/cancellation/balance loads | Protocol enforced via the only available unit of work; concurrency test per write path; DB backstops |
| Lock contention for one employee | Second request waits | Short transactions; 5 s lock timeout → `503` |
| Client retry ambiguity without idempotency key (T3) | Client unsure whether original succeeded | `overlappingRequestId` + GET to confirm (R15); idempotency key as future option |
| Fractional entitlements arrive from the source system | Integer balances cannot represent them | Loader rejects fractions; revisit precision if A2/B3 require (R10) |
| Docker unavailable on a CI platform | Integration tests cannot run | Linux runners with Docker required (R13) |
| Unconfirmed assumptions change | Rule, test, and contract changes | Rules isolated in Domain; contract codes stable |

### 12. Delivery approach (smallest complete increment first)

| Increment | Delivers | Done when |
|---|---|---|
| 0. Foundation | Solution, build props, Api with Problem Details, JWT wiring with subject mapping, health checks, DbContext + first migration, employee unit of work with lock, SQL container fixture, architecture test | Build clean; health check and lock smoke test pass against containerized SQL Server |
| 1. MVP — submit + read own (US1 + core US4) | Input validation, `(issuer, subject)` → employee (401/403), active leave type, working days with holidays, zero-day rejection, balance exists + sufficient + reserve, Pending + review route, `201` + owner-only `GET`, single transaction under employee lock | US1, US4, US2 #1–3, #5 pass; own-request `404` for others |
| 2. Remaining rules (US2/US3) | Overlap (calendar dates, cross-type), spans years, start in past, future-year balance, full error catalogue with extensions | All US2/US3 scenarios and codes covered |
| 3. Concurrency & integrity hardening | Concurrency suite, duplicate test, rollback injection, rowversion → `409`, lock timeout → `503` | Concurrency tests pass repeatedly |
| 4. Observability & contract polish | Log events, reason-redaction test, OpenAPI completeness check, load-test script | Quickstart 1–18 pass; SC-001 measured |
| B1 (gated) | Calendar completeness marker, `422 LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE`, warning log | Only after business approval **and** spec update |

The lock and single transaction are in increment 1, so it is already safe; increment 3 adds the
evidence and backstop handling. No release before increment 3; no production deployment before
B3 and B4 are resolved.

## Decision Status

### Approved (Tech Lead review, 2026-10-18)

| ID | Decision | Where documented |
|---|---|---|
| T1 | Employee-level database lock protocol for all write paths for an employee | research R1 (mechanism, transaction boundary, L1–L6, write paths, enforcement); Design 5–7 |
| T2 | `422` for documented business-rule failures; `409` for genuine state/concurrency conflicts; stable error codes | research R6; contracts/errors.md (policy + catalogue); openapi.yaml |
| T3 | No idempotency key in v1; duplicates cannot reserve twice; retry ambiguity documented; idempotency as future option | research R15; errors.md (`LEAVE_OVERLAP`); quickstart #9, #18 |

### Technical items addressed in this revision (no further approval requested)

| ID | Outcome | Where documented |
|---|---|---|
| T4 | Identity-provider dependency documented; mapping = issuer + configured stable claim (`sub`, or `oid` for Entra ID) → unique active employee; no mutable attributes. **Provider selection remains open** (production dependency). | research R8; data-model Employee |
| T5 | Local prerequisites (Docker Linux containers, resources, x64) and CI prerequisites (Linux runner with Docker, pinned image, Ryuk, gating order) documented. **CI platform and production SQL Server version remain open.** | research R13; quickstart |
| T6 | Owner-only `GET /leave-requests/{id}` included; resource-level authorization by id + employee id in one query; `404` for missing or not owned | research R14; openapi.yaml; quickstart #17 |
| T7 | Reassessed: **whole-day integers** replace `decimal(5,2)`; switch only if half-day leave (A2) or fractional entitlements (B3) are required; fractional loads rejected meanwhile | research R10; data-model LeaveBalance; openapi.yaml |

### Pending business approval

| ID | Item | Status | Effect until approved |
|---|---|---|---|
| B1 | **Proposed** rule: fail closed when the holiday calendar for the leave year is missing or not verifiably complete — `422 LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE`, nothing reserved, `Warning` log without reason text (research R16) | Pending business approval — **not** an approved policy | Not implemented. Design is ready; requires approval **and** a spec update (new FR + scenario) before its tasks run. Risk of counting holidays as leave days remains open. |
| B2 | Assumptions A2 (whole days), A3 (every type has a balance), A4 (no past start; today allowed), A5 (reason ≤ 500), A7 (no notice period), A11 (single holiday calendar), A12 (Sat/Sun weekend) | **Unconfirmed** | Implemented as written in the spec so work can proceed; each is an isolated rule and may change after business review. |

### Dependencies to resolve before production deployment

| ID | Dependency | Why it blocks production |
|---|---|---|
| B3 | Production source and ownership of employees (incl. identity issuer/subject mapping and manager links), leave types, balances (whole-day entitlements), and public holidays (incl. calendar completeness if B1 is approved) — and confirmation that balance-changing loads follow the lock protocol | Without it, no real employee can be mapped or have a balance; the feature only works with dev seed data. |
| B4 | Organisation time zone (IANA id) | The application refuses to start without it; it defines "today" and the leave-year boundary. |
| T4 (open part) | Identity provider selection, issuer, audience, subject claim | Tokens cannot be validated in production. |
| T5 (open part) | CI platform; production SQL Server major version (to pin the test image) | Integration tests must gate merges; test DB should match production. |

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| Pessimistic employee-level row lock instead of the constitution's default optimistic concurrency (Principle V allows a documented alternative; approved as T1) | FR-013/FR-014 require that overlapping requests of **different** leave types never both succeed; those touch different balance rows, so a balance-row token alone cannot detect the conflict. The lock also gives the second submission the correct business error without retries. | Optimistic `rowversion` on `LeaveBalance` only misses cross-type overlap. Optimistic token on the Employee row works but needs a retry loop and an Employee write on every submission. |
