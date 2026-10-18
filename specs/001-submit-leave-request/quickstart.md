# Quickstart & Validation Guide: Submit Leave Request

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) |
**Contract**: [contracts/openapi.yaml](./contracts/openapi.yaml),
[contracts/errors.md](./contracts/errors.md)

This guide shows how to prove the feature works once it is implemented. It intentionally contains
no implementation code. Project names follow [plan.md → Project Structure](./plan.md#project-structure).

## Prerequisites

- .NET SDK 10.0.x (`dotnet --list-sdks`)
- Docker running with **Linux containers** (Docker Desktop with WSL 2 on Windows), ≥ 2 GB RAM for
  the SQL Server container, x64 (Apple Silicon needs emulation). Full local and CI prerequisites:
  [research.md → R13](./research.md#r13-testing-infrastructure-and-prerequisites-t5).
  Without Docker, only the unit tests can run.
- For manual runs: a local SQL Server (container or LocalDB) and its connection string in user
  secrets for `LeaveManagement.Api` (key `ConnectionStrings:LeaveManagement`) — never committed.

## 1. Build and automated tests

```powershell
dotnet build LeaveManagement.slnx        # must finish with 0 warnings (warnings are errors)
dotnet test tests/LeaveManagement.UnitTests
dotnet test tests/LeaveManagement.IntegrationTests   # requires Docker
```

Expected: all tests pass. Integration tests cover the scenarios in section 4 automatically; the
manual steps below are for demonstration and exploratory checks.

## 2. Prepare a local database (manual runs only)

```powershell
dotnet ef database update --project src/LeaveManagement.Infrastructure --startup-project src/LeaveManagement.Api
```

Seed development reference data with the development seeding step defined in the tasks phase
(employees with and without a manager, leave types `ANNUAL` (active) and one inactive type,
balances for the current and next year where required, and public holidays). Seeding runs only in
the Development environment.

## 3. Run the API and obtain a token

```powershell
dotnet run --project src/LeaveManagement.Api
dotnet user-jwts create --project src/LeaveManagement.Api --name employee-a --claim sub=employee-a
```

Use the printed token as `Authorization: Bearer <token>`. The seeded employee must carry the
same issuer and subject as the token (`IdentityIssuer`, `IdentitySubject`; mapping rule in
[research.md → R8](./research.md#r8-authentication-and-subject-to-employee-mapping-t4)). The OpenAPI document is available at
`/openapi/v1.json` in Development; check that `POST /api/v1/leave-requests` and
`GET /api/v1/leave-requests/{id}` are listed with their status codes.

## 4. Validation scenarios

Use dates in a future week of the current year and adjust for weekends and holidays in your seed
data. "Employee A" has 5 available `ANNUAL` days and a manager; "Employee B" has no manager.

| # | Scenario | Action | Expected outcome | Spec |
|---|---|---|---|---|
| 1 | Happy path | A submits Mon–Wed `ANNUAL` with a reason | `201`, `Location` header, body `status=Pending`, `countedDays=3`, `reviewRoute=Manager`; GET on `Location` returns the same request | US1 #1–2, FR-006 |
| 2 | Holiday excluded | A submits Fri–Tue where Monday is a holiday | `201`, `countedDays=2` | US1 #4, FR-010 |
| 3 | Balance reserved | After #1, A submits 3 more days elsewhere | `422 LEAVE_INSUFFICIENT_BALANCE`, `requestedDays=3`, `availableDays=2` | US2 #2, #6 |
| 4 | Missing fields | Empty JSON object `{}` | `400` with errors for all four fields in one response | US3 #1, #4 |
| 5 | End before start | `endDate` one day before `startDate` | `400`, error on `endDate` | US3 #2 |
| 6 | Bad date format | `startDate` = `12/03/2027` | `400`, error on `startDate`, other field errors also listed | US3 #5 |
| 7 | Weekend only | Sat–Sun | `422 LEAVE_ZERO_WORKING_DAYS` | US3 #6 |
| 8 | Overlap across types | A submits a different leave type overlapping #1 by one date | `422 LEAVE_OVERLAP` with `overlappingRequestId` = #1 | US3 #7, FR-014 |
| 9 | Retry | Repeat request #1 exactly | `422 LEAVE_OVERLAP` referencing #1; balance unchanged | FR-018a |
| 10 | Spans years | 30 Dec – 2 Jan | `422 LEAVE_SPANS_LEAVE_YEARS` | FR-020 |
| 11 | Next year, no balance | Dates next year, no balance row | `422 LEAVE_BALANCE_NOT_AVAILABLE`, `leaveYear` | US2 #7 |
| 12 | No manager | B submits a valid request | `201`, `reviewRoute=FallbackReviewer` | US1 #6, FR-021 |
| 13 | Unauthenticated | No token | `401` | US4 #1 |
| 14 | Unknown identity | Token for a `sub` with no employee | `403 EMPLOYEE_NOT_FOUND` | US4 #4 |
| 15 | Spoofed fields | A sends `employeeId` of B, `status=Approved`, `countedDays=1` | Created for A, `Pending`, server-computed `countedDays` | US4 #2–3, FR-009 |
| 16 | Reason not logged | Submit with reason `ZZ-SECRET-REASON-ZZ`, then search logs | String not found in any log output | FR-019, Constitution VII |
| 17 | Own request only | B calls GET on A's request id from #1; then GET on a random GUID | Both `404` with identical bodies (apart from `traceId`) | T6, R14 |
| 18 | Retry ambiguity | After #9, GET the `overlappingRequestId` | Same dates/type as #1, `Pending` → original confirmed | T3, R15 |

Scenario for B1 (holiday calendar not confirmed complete → `422
LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE`) is **not** included: B1 is pending business approval and
will be added only if approved.

## 5. Concurrency check

Automated in the integration suite (spec US2 #4, FR-013): with 5 available days, fire two 3-day
submissions for the same employee simultaneously → exactly one `201` and one
`422 LEAVE_INSUFFICIENT_BALANCE`; reserved days = 3. Repeat with two overlapping requests of
different leave types → exactly one `201` and one `422 LEAVE_OVERLAP`.

Manual reproduction: send both requests in parallel (e.g., two terminals or a parallel HTTP client)
and confirm the same outcome.

## 6. Performance check (on demand)

Run the load-test script (added in the tasks phase) with 50 concurrent virtual employees each
submitting valid requests against a production-like database. Pass: 95th percentile server-side
duration < 2 s for accepted and rejected submissions (SC-001).
