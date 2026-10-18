---

description: "Task list for feature 001-submit-leave-request"
---

# Tasks: Submit Leave Request

**Input**: Design documents from `specs/001-submit-leave-request/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/openapi.yaml](./contracts/openapi.yaml),
[contracts/errors.md](./contracts/errors.md), [quickstart.md](./quickstart.md)

**Tests**: Included. Constitution Principle III makes unit tests for business rules and
integration tests for database and API behavior mandatory. Writing tests before implementation is
encouraged (listed first in each story) but not mandated.

**Organization**: Tasks are grouped by user story (spec.md US1–US4) so each story can be
implemented and tested as an increment. A final gated phase holds the B1 tasks, which are
**BLOCKED** until business approval.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (US1–US4)
- All paths are relative to the repository root `D:\learn-claude-code\leave-management`

## Path Conventions

- Production code: `src/LeaveManagement.{Domain,Application,Infrastructure,Api}/`
- Tests: `tests/LeaveManagement.UnitTests/`, `tests/LeaveManagement.IntegrationTests/`
- Layer references (Constitution I): Domain → none; Application → Domain;
  Infrastructure → Application, Domain; Api → Application, Infrastructure.
  Domain and Application MUST NOT reference EF Core or ASP.NET Core packages.

## Global rules for every task

- Never log, echo in validation messages, or include in exception messages the leave `reason`
  text, identity subjects, tokens, or request bodies (FR-019, research R12).
- Business failures are returned as `Result`/`DomainError` values, never thrown (research R5).
- Use `TimeProvider` for all time; never `DateTime.Now`/`UtcNow` directly.
- Every write path that changes leave requests or balances MUST run through
  `IEmployeeUnitOfWork` (employee lock protocol L1–L6, research R1.3).
- Rule evaluation order in the submit use case is fixed (data-model.md → Submission invariants):
  leave type exists and is active → start not in past → same calendar year → *(B1 only if approved)* calendar complete → working
  days > 0 → no overlap → balance exists → sufficient balance.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Solution skeleton and build configuration.

- [X] T001 Create `global.json` pinning the .NET SDK to `10.0.401` with `"rollForward": "latestFeature"` in global.json
- [X] T002 Create the solution `LeaveManagement.slnx` with projects `src/LeaveManagement.Domain` (classlib), `src/LeaveManagement.Application` (classlib), `src/LeaveManagement.Infrastructure` (classlib), `src/LeaveManagement.Api` (ASP.NET Core empty web project, Minimal APIs), `tests/LeaveManagement.UnitTests` (xUnit), `tests/LeaveManagement.IntegrationTests` (xUnit); add project references exactly as in Path Conventions plus UnitTests → Domain, Application and IntegrationTests → Api, Infrastructure; delete template placeholder files (`Class1.cs`, `UnitTest1.cs`, template weather endpoint) in LeaveManagement.slnx
- [X] T003 [P] Create `Directory.Build.props` with `TargetFramework` `net10.0`, `Nullable` `enable`, `ImplicitUsings` `enable`, `TreatWarningsAsErrors` `true`, `EnforceCodeStyleInBuild` `true`, `AnalysisLevel` `latest`, `ManagePackageVersionsCentrally` `true` in Directory.Build.props
- [X] T004 [P] Create `Directory.Packages.props` with central versions (latest stable 10.x where applicable) for: `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`, `Microsoft.AspNetCore.Authentication.JwtBearer`, `Microsoft.AspNetCore.OpenApi`, `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`, `Microsoft.Extensions.TimeProvider.Testing`, `Microsoft.AspNetCore.Mvc.Testing`, `Testcontainers.MsSql`, `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`; add package references to the projects that need them (EF Core packages only in Infrastructure and Api) in Directory.Packages.props
- [X] T005 [P] Create `.editorconfig` (C# conventions, file-scoped namespaces, `var` when type is apparent, 4-space indent) and `.gitignore` (standard .NET: `bin/`, `obj/`, `*.user`, `.vs/`, `TestResults/`) in .editorconfig
- [X] T006 [P] Create a local tool manifest with `dotnet-ef` (10.x) in .config/dotnet-tools.json

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Domain types, schema, transaction/lock infrastructure, API pipeline, authentication,
and test infrastructure that every story needs.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Domain foundation

- [ ] T007 [P] Create `Result` and `Result<T>` (success flag, `Value`, `Error`; static `Success`/`Failure` factories; no exceptions for business failures) in src/LeaveManagement.Domain/Common/Result.cs
- [ ] T008 [P] Create `DomainError` record (`string Code`, `string Message`, `IReadOnlyDictionary<string, object> Extensions` defaulting to empty; message must never contain reason text) in src/LeaveManagement.Domain/Common/DomainError.cs
- [ ] T009 [P] Create `ErrorCodes` static class with string constants `EMPLOYEE_NOT_FOUND`, `LEAVE_TYPE_INVALID`, `LEAVE_START_IN_PAST`, `LEAVE_SPANS_LEAVE_YEARS`, `LEAVE_ZERO_WORKING_DAYS`, `LEAVE_OVERLAP`, `LEAVE_BALANCE_NOT_AVAILABLE`, `LEAVE_INSUFFICIENT_BALANCE`, `CONCURRENCY_CONFLICT`, `SUBMISSION_TIMEOUT`, `NOT_FOUND` (do NOT add `LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE`; it belongs to the blocked B1 phase) in src/LeaveManagement.Domain/Common/ErrorCodes.cs
- [ ] T010 [P] Create `ReviewRoute` enum (`Manager`, `FallbackReviewer`) in src/LeaveManagement.Domain/Employees/ReviewRoute.cs and `LeaveRequestStatus` enum (`Pending`, `Approved`, `Rejected`, `Cancelled`) in src/LeaveManagement.Domain/LeaveRequests/LeaveRequestStatus.cs
- [ ] T011 [P] Create `Employee` entity: `Id` (Guid, "Primary key. Internal identifier used in logs and responses."), `IdentityIssuer` (string, "Required", ≤ 300), `IdentitySubject` (string, "Required", ≤ 200, "Unique together with IdentityIssuer"), `ManagerId` (Guid?, "Null means no reporting manager (FR-021). Cannot reference itself."), `IsActive` (bool); private setters, constructor for test/seed creation that rejects `ManagerId == Id`, private parameterless constructor for EF in src/LeaveManagement.Domain/Employees/Employee.cs
- [ ] T012 [P] Create `LeaveType` entity: `Id` (Guid), `Code` (string, "Required, unique, stable (e.g., `ANNUAL`)", ≤ 50), `Name` (string, "Required. Display name.", ≤ 100), `IsActive` (bool, "Only active types can be requested (FR-004).") in src/LeaveManagement.Domain/LeaveTypes/LeaveType.cs
- [ ] T013 [P] Create `LeaveBalance` entity properties: `Id`, `EmployeeId`, `LeaveTypeId`, `LeaveYear` (int, calendar year), `EntitlementDays` (int, "≥ 0"), `UsedDays` (int, "≥ 0"), `ReservedDays` (int, "≥ 0"), `RowVersion` (byte[]), and computed read-only `AvailableDays = EntitlementDays − UsedDays − ReservedDays` (not mapped); constructor rejects negative values; behavior (`Reserve`) is added in US1 in src/LeaveManagement.Domain/LeaveBalances/LeaveBalance.cs
- [ ] T014 [P] Create `PublicHoliday` entity: `Id` (Guid), `Date` (DateOnly, "Required, unique"), `Name` (string, "Required", ≤ 100) in src/LeaveManagement.Domain/Calendar/PublicHoliday.cs
- [ ] T015 Create `LeaveRequest` entity properties only (factory added in US1): `Id` (Guid, server-generated), `EmployeeId`, `LeaveTypeId`, `StartDate` (DateOnly), `EndDate` (DateOnly, "≥ StartDate"), `LeaveYear` (int, "= year of StartDate = year of EndDate"), `CountedDays` (int, "> 0"), `Reason` (string, "Required, trimmed, not whitespace-only", ≤ 500), `Status` (`LeaveRequestStatus`), `ReviewRoute` (`ReviewRoute`), `ReviewerEmployeeId` (Guid?, "The manager's Id when `ReviewRoute = Manager`; null for `FallbackReviewer`"), `SubmittedAt` (DateTimeOffset, UTC), `RowVersion` (byte[]), navigation `LeaveType`; private setters and private EF constructor in src/LeaveManagement.Domain/LeaveRequests/LeaveRequest.cs

### Persistence foundation

- [ ] T016 Create `LeaveManagementDbContext` with `DbSet`s `Employees`, `LeaveTypes`, `LeaveBalances`, `PublicHolidays`, `LeaveRequests`; apply configurations from the assembly in src/LeaveManagement.Infrastructure/Persistence/LeaveManagementDbContext.cs
- [ ] T017 [P] Create `EmployeeConfiguration`: table `Employees`; `IdentityIssuer` required max 300; `IdentitySubject` required max 200; unique index `(IdentityIssuer, IdentitySubject)`; optional self-FK `ManagerId` with `DeleteBehavior.Restrict`; check constraint `CK_Employees_NotOwnManager` = `ManagerId IS NULL OR ManagerId <> Id` in src/LeaveManagement.Infrastructure/Persistence/Configurations/EmployeeConfiguration.cs
- [ ] T018 [P] Create `LeaveTypeConfiguration`: table `LeaveTypes`; `Code` required max 50 unique index; `Name` required max 100 in src/LeaveManagement.Infrastructure/Persistence/Configurations/LeaveTypeConfiguration.cs
- [ ] T019 [P] Create `LeaveBalanceConfiguration`: table `LeaveBalances`; int columns `EntitlementDays`, `UsedDays`, `ReservedDays`; ignore `AvailableDays`; `RowVersion` as `IsRowVersion()`; unique index `(EmployeeId, LeaveTypeId, LeaveYear)`; check constraint `CK_LeaveBalances_Amounts` = `UsedDays >= 0 AND ReservedDays >= 0 AND UsedDays + ReservedDays <= EntitlementDays` and `EntitlementDays >= 0`; FKs to Employees and LeaveTypes (Restrict) in src/LeaveManagement.Infrastructure/Persistence/Configurations/LeaveBalanceConfiguration.cs
- [ ] T020 [P] Create `PublicHolidayConfiguration`: table `PublicHolidays`; `Date` as SQL `date` with unique index; `Name` required max 100 in src/LeaveManagement.Infrastructure/Persistence/Configurations/PublicHolidayConfiguration.cs
- [ ] T021 [P] Create `LeaveRequestConfiguration`: table `LeaveRequests`; `StartDate`/`EndDate` as `date`; `Reason` required max 500; `Status` and `ReviewRoute` stored as strings (max 20); `SubmittedAt` `datetimeoffset`; `RowVersion` `IsRowVersion()`; FKs to Employees (`EmployeeId`), Employees (`ReviewerEmployeeId`, optional), LeaveTypes (all Restrict); check constraints: `EndDate >= StartDate`, `CountedDays > 0`, `YEAR(StartDate) = YEAR(EndDate)`, `LeaveYear = YEAR(StartDate)`, and `(ReviewRoute = 'Manager' AND ReviewerEmployeeId IS NOT NULL) OR (ReviewRoute = 'FallbackReviewer' AND ReviewerEmployeeId IS NULL)`; index `IX_LeaveRequests_Overlap` on `(EmployeeId, Status, StartDate, EndDate)` in src/LeaveManagement.Infrastructure/Persistence/Configurations/LeaveRequestConfiguration.cs
- [ ] T022 Create `LeaveManagementDesignTimeDbContextFactory : IDesignTimeDbContextFactory<LeaveManagementDbContext>` (SQL Server provider; connection string from environment variable `LEAVEMANAGEMENT_DESIGN_CONNECTION`, defaulting to a LocalDB connection string; used only by EF tooling — `migrations add` does not connect); add `Microsoft.EntityFrameworkCore.Design` to the Infrastructure project with `PrivateAssets="all"`; then run `dotnet ef migrations add InitialCreate --project src/LeaveManagement.Infrastructure --startup-project src/LeaveManagement.Infrastructure --output-dir Persistence/Migrations` and review the generated SQL for every constraint from T017–T021 in src/LeaveManagement.Infrastructure/Persistence/LeaveManagementDesignTimeDbContextFactory.cs
- [ ] T023 [P] Create `SqlServerErrors` helper that recognises `SqlException.Number` `1222` (lock request timeout) and `1205` (deadlock victim) in src/LeaveManagement.Infrastructure/Persistence/SqlServerErrors.cs
- [ ] T024 [P] Create `PersistenceOptions` (`LockTimeoutMilliseconds` default `5000`, range 100–30000; `EnableSensitiveDataLogging` default `false`) bound from section `Persistence` in src/LeaveManagement.Infrastructure/Persistence/PersistenceOptions.cs

### Application foundation (ports and options)

- [ ] T025 [P] Create port `ICurrentEmployeeLookup` with `Task<Employee?> FindActiveByIdentityAsync(string issuer, string subject, CancellationToken ct)` (returns null when no match or inactive; never matches on email/UPN) in src/LeaveManagement.Application/Abstractions/ICurrentEmployeeLookup.cs
- [ ] T026 [P] Create port `IEmployeeUnitOfWork` with `Task<Result<T>> ExecuteForEmployeeAsync<T>(Guid employeeId, Func<Employee, CancellationToken, Task<Result<T>>> work, CancellationToken ct)`; XML doc states the contract: starts a transaction, sets the lock timeout, locks the employee row FIRST, re-reads it, returns `EMPLOYEE_NOT_FOUND` if missing or inactive, runs `work`, saves and commits only when `work` succeeds, otherwise rolls back; it is the only way write use cases obtain a transaction (rules L1–L6) in src/LeaveManagement.Application/Abstractions/IEmployeeUnitOfWork.cs
- [ ] T027 [P] Create `LeaveOptions` with required `OrganisationTimeZone` (IANA id) and a validator that fails startup when missing or not resolvable via `TimeZoneInfo.FindSystemTimeZoneById`, in src/LeaveManagement.Application/Abstractions/LeaveOptions.cs
- [ ] T028 Create `OrganisationClock` (depends on `TimeProvider` and `IOptions<LeaveOptions>`) exposing `DateOnly Today` in the organisation time zone and `DateTimeOffset UtcNow` in src/LeaveManagement.Application/Common/OrganisationClock.cs
- [ ] T029 Create `AddApplication(IServiceCollection)` registering `OrganisationClock` and `LeaveOptions` validation (`ValidateOnStart`) in src/LeaveManagement.Application/DependencyInjection.cs

### Infrastructure foundation (lock protocol, identity lookup, DI)

- [ ] T030 Implement `EmployeeUnitOfWork : IEmployeeUnitOfWork` using `Database.CreateExecutionStrategy().ExecuteAsync(...)`; per attempt: `ChangeTracker.Clear()`, `BeginTransactionAsync(IsolationLevel.ReadCommitted)`, execute `SET LOCK_TIMEOUT {PersistenceOptions.LockTimeoutMilliseconds}`, load the employee with `FromSql` `SELECT * FROM Employees WITH (UPDLOCK, ROWLOCK) WHERE Id = {employeeId}` (tracked; no HOLDLOCK), return `EMPLOYEE_NOT_FOUND` if null or `!IsActive`, invoke `work`; on success `SaveChangesAsync` once then `CommitAsync`; on failure `RollbackAsync`; add a class comment summarising rules L1–L6 from research.md R1.3 in src/LeaveManagement.Infrastructure/Persistence/EmployeeUnitOfWork.cs
- [ ] T031 [P] Implement `CurrentEmployeeLookup : ICurrentEmployeeLookup` (AsNoTracking query on `IdentityIssuer == issuer && IdentitySubject == subject && IsActive`) in src/LeaveManagement.Infrastructure/Persistence/Stores/CurrentEmployeeLookup.cs
- [ ] T032 Create `AddInfrastructure(IServiceCollection, IConfiguration, IHostEnvironment)`: `LeaveManagementDbContext` with SQL Server using connection string `ConnectionStrings:LeaveManagement` and `EnableRetryOnFailure()`; `EnableSensitiveDataLogging` only when environment is Development AND `Persistence:EnableSensitiveDataLogging` is true; bind `PersistenceOptions` with validation; register `IEmployeeUnitOfWork`, `ICurrentEmployeeLookup` as scoped in src/LeaveManagement.Infrastructure/DependencyInjection.cs

### API foundation (errors, auth, pipeline)

- [ ] T033 [P] Create `DomainErrorProblemMapper` that converts a `DomainError` to an `IResult` per contracts/errors.md: `EMPLOYEE_NOT_FOUND`→403, `NOT_FOUND`→404, `CONCURRENCY_CONFLICT`→409, all `LEAVE_*` business codes (including `LEAVE_TYPE_INVALID`)→422, `SUBMISSION_TIMEOUT`→503; Problem Details with `type` `/problems/{code-in-kebab-case}`, `title`, `detail` (= error message), `errorCode`, `traceId`, and every entry of `DomainError.Extensions` as a top-level extension (integers stay integers) in src/LeaveManagement.Api/ErrorHandling/DomainErrorProblemMapper.cs
- [ ] T034 [P] Create `GlobalExceptionHandler : IExceptionHandler` returning a generic 500 Problem Details (title "An unexpected error occurred.", `traceId`, no exception details), mapping `BadHttpRequestException` (malformed JSON/body) to a 400 Problem Details, and logging the exception with the trace id via `ApiLog` in src/LeaveManagement.Api/ErrorHandling/GlobalExceptionHandler.cs
- [ ] T035 [P] Create `ApiLog` source-generated `LoggerMessage` definitions: EventId 2001 `AuthorizationDenied(string reason)` (Warning; reason is a fixed category such as "NoEmployeeRecord", never the subject), EventId 2002 `UnhandledException(Exception ex)` (Error) in src/LeaveManagement.Api/Logging/ApiLog.cs
- [ ] T036 [P] Create `LeaveAuthenticationOptions` (`Authority`, `Audience`, `SubjectClaim` default `"sub"`; validated at startup, Authority/Audience required outside Development) bound from section `Authentication` in src/LeaveManagement.Api/Authentication/LeaveAuthenticationOptions.cs
- [ ] T037 Create `AuthenticationSetup.AddLeaveAuthentication(...)`: JWT bearer with `MapInboundClaims = false`, `TokenValidationParameters` validating issuer, audience, lifetime, signing key (authority metadata; in Development also accept `dotnet user-jwts` configuration), authorization policy `"Employee"` = authenticated user, fallback policy = authenticated user in src/LeaveManagement.Api/Authentication/AuthenticationSetup.cs
- [ ] T038 [P] Create `ClaimsPrincipalExtensions.TryGetIdentity(this ClaimsPrincipal, string subjectClaim, out string issuer, out string subject)` reading `iss` and the configured subject claim; returns false when either is missing; never reads email/UPN in src/LeaveManagement.Api/Authentication/ClaimsPrincipalExtensions.cs
- [ ] T039 Write `Program.cs`: `AddProblemDetails` (add `traceId` to every problem), `AddExceptionHandler<GlobalExceptionHandler>`, `AddValidation()`, `AddOpenApi("v1")` with bearer security scheme, `AddLeaveAuthentication`, `AddHealthChecks().AddDbContextCheck<LeaveManagementDbContext>(tags: ready)`, `TimeProvider.System` singleton, `AddApplication`, `AddInfrastructure`; pipeline: `UseExceptionHandler`, `UseStatusCodePages()` (with `AddProblemDetails` registered this writes a Problem Details body with `traceId` for every empty-body 4xx/5xx, including the JWT 401 challenge and policy 403), `UseAuthentication`, `UseAuthorization`, `MapOpenApi()` in Development only; health endpoints as Minimal API endpoints (not `MapHealthChecks`, which is excluded from OpenAPI): `GET /health/live` returns 200 `Healthy` without running checks, `GET /health/ready` runs `HealthCheckService.CheckHealthAsync` filtered to the `ready` tag and returns 200 `Healthy` or 503 `Unhealthy` as `text/plain`; both `.AllowAnonymous().WithTags("Health")`; end the file with `public partial class Program;` in src/LeaveManagement.Api/Program.cs
- [ ] T040 [P] Create configuration files: `appsettings.json` with `Authentication` (`Authority`, `Audience` empty, `SubjectClaim` `"sub"`), `Leave:OrganisationTimeZone` empty (TODO B4), `Persistence:LockTimeoutMilliseconds` 5000, no connection string or secrets; `appsettings.Development.json` with `Leave:OrganisationTimeZone` `"Europe/London"` (development placeholder until B4) and logging levels; the user-secrets connection-string setup is documented in README (T099) in src/LeaveManagement.Api/appsettings.json

### Test infrastructure

- [ ] T041 [P] Create `SqlServerContainerFixture` (xUnit collection fixture `"SqlServer"`) using Testcontainers `MsSqlBuilder` with image `mcr.microsoft.com/mssql/server:2022-latest` (add comment: pin to the production major version once T5/SQL Server version is confirmed), start once per run, apply migrations once via `Database.MigrateAsync()`, expose connection string in tests/LeaveManagement.IntegrationTests/Infrastructure/SqlServerContainerFixture.cs
- [ ] T042 [P] Create `TestAuthHandler` (scheme `"Test"`): authenticates only when headers `X-Test-Issuer` and `X-Test-Subject` are present, issuing claims `iss` and `sub`; otherwise no result (→ 401) in tests/LeaveManagement.IntegrationTests/Infrastructure/TestAuthHandler.cs
- [ ] T043 [P] Create `CapturingLoggerProvider` that stores every formatted message and structured state values for assertions in tests/LeaveManagement.IntegrationTests/Infrastructure/CapturingLoggerProvider.cs
- [ ] T044 Create `ApiFactory : WebApplicationFactory<Program>`: uses the container connection string, replaces authentication with the `Test` scheme as default, sets `Leave:OrganisationTimeZone` = `"Etc/UTC"`, `Persistence:LockTimeoutMilliseconds` = `500`, replaces `TimeProvider` with a `FakeTimeProvider` fixed at 2027-01-04T09:00:00Z (exposed for tests), registers `CapturingLoggerProvider`; helper `CreateClientFor(issuer, subject)` in tests/LeaveManagement.IntegrationTests/Infrastructure/ApiFactory.cs
- [ ] T045 [P] Create `TestDataBuilder` with methods to insert employees (with/without manager, active/inactive, issuer+subject), leave types (active/inactive), balances (employee, type, year, entitlement, used, reserved), public holidays, and existing leave requests (any status) directly via `LeaveManagementDbContext` in tests/LeaveManagement.IntegrationTests/Infrastructure/TestDataBuilder.cs
- [ ] T046 [P] Create `DatabaseReset` that deletes all rows in FK-safe order (`LeaveRequests`, `LeaveBalances`, `PublicHolidays`, `LeaveTypes`, `Employees` — managers after reports) before each test class in tests/LeaveManagement.IntegrationTests/Infrastructure/DatabaseReset.cs
- [ ] T047 [P] Write `LayerDependencyTests`: reflection over referenced assemblies asserts Domain references no `Microsoft.EntityFrameworkCore*` or `Microsoft.AspNetCore*`, Application references neither, Domain references no Application/Infrastructure/Api assembly in tests/LeaveManagement.UnitTests/Architecture/LayerDependencyTests.cs
- [ ] T048 [P] Write `HealthCheckTests`: `/health/live` and `/health/ready` return 200 anonymously against the container in tests/LeaveManagement.IntegrationTests/Api/HealthCheckTests.cs
- [ ] T049 [P] Write `SchemaConstraintTests` inserting rows via raw SQL/DbContext and asserting SQL Server rejects: `UsedDays + ReservedDays > EntitlementDays`, negative `ReservedDays`, duplicate `(EmployeeId, LeaveTypeId, LeaveYear)`, `EndDate < StartDate`, `CountedDays = 0`, start/end in different years, `ReviewRoute = 'Manager'` with null reviewer, duplicate `(IdentityIssuer, IdentitySubject)`, `ManagerId = Id` in tests/LeaveManagement.IntegrationTests/Persistence/SchemaConstraintTests.cs
- [ ] T050 Write `EmployeeLockTests`: (a) while transaction A holds the lock via `EmployeeUnitOfWork`, transaction B for the same employee waits and then sees A's committed changes; (b) B for a different employee proceeds without waiting; (c) a plain `AsNoTracking` read of the employee is not blocked; (d) inactive or missing employee → `EMPLOYEE_NOT_FOUND`, no writes in tests/LeaveManagement.IntegrationTests/Persistence/EmployeeLockTests.cs

**Checkpoint**: `dotnet build` has 0 warnings; architecture, health, schema, and lock tests pass.

---

## Phase 3: User Story 1 — Submit a valid leave request (Priority: P1) 🎯 MVP

**Goal**: An authenticated employee submits a request; it is created `Pending` with working days
counted (holidays excluded), its days reserved in the same transaction, routed to the manager or
the fallback reviewer, and returned with `201` + `Location`; the employee can read it back.

**Independent Test**: With an employee who has a manager and 5 available `ANNUAL` days, POST a
Mon–Wed request → `201`, `status=Pending`, `countedDays=3`, `reviewRoute=Manager`, `Location`
resolves via GET to the same request, and the balance shows `ReservedDays = 3`. An employee
without a manager gets `reviewRoute=FallbackReviewer`. (spec US1 #1–6)

### Tests for User Story 1

- [ ] T051 [P] [US1] Write `WorkingDayCalculatorTests`: single working day = 1; Mon–Fri = 5; Fri–Tue with Monday holiday = 2; Sat–Sun = 0; holiday on Saturday not double-counted; range starting/ending on weekend counts only inner working days; 1 Jan–31 Dec of a year with a known holiday set returns the expected count in tests/LeaveManagement.UnitTests/Domain/WorkingDayCalculatorTests.cs
- [ ] T052 [P] [US1] Write `LeaveBalanceTests`: `AvailableDays` = entitlement − used − reserved; `Reserve(5)` with 5 available succeeds and increases `ReservedDays`; `Reserve(6)` with 5 available fails with `LEAVE_INSUFFICIENT_BALANCE` and extensions `requestedDays=6`, `availableDays=5`, leaving state unchanged; `Reserve` with 0 available fails in tests/LeaveManagement.UnitTests/Domain/LeaveBalanceTests.cs
- [ ] T053 [P] [US1] Write `EmployeeTests` and `LeaveRequestTests`: employee with manager → `(Manager, managerId)`, without → `(FallbackReviewer, null)`; `LeaveRequest.Submit` → `Pending`, trimmed reason, `LeaveYear` = start year, `SubmittedAt` from argument; counted days 0 → `LEAVE_ZERO_WORKING_DAYS`; start and end in different years → `LEAVE_SPANS_LEAVE_YEARS`; end before start throws `ArgumentException` (guarded earlier by API validation) in tests/LeaveManagement.UnitTests/Domain/LeaveRequestTests.cs
- [ ] T054 [P] [US1] Create hand-written fakes: `FakeCurrentEmployeeLookup`, `FakeEmployeeUnitOfWork` (records whether commit happened; commits only on success; can return `EMPLOYEE_NOT_FOUND`), `InMemoryLeaveSubmissionStore`, `FakePublicHolidayCalendar`, `InMemoryOwnLeaveRequestQuery` in tests/LeaveManagement.UnitTests/Application/Fakes/
- [ ] T055 [US1] Write `SubmitLeaveRequestHandlerTests` (happy path and US1-scope rules): valid request → success with `Pending`, counted days excluding holidays, reservation on the correct balance, commit recorded; employee without manager → `FallbackReviewer`; unknown/inactive leave type → `LEAVE_TYPE_INVALID`, no commit; no balance row → `LEAVE_BALANCE_NOT_AVAILABLE`, no commit; insufficient → `LEAVE_INSUFFICIENT_BALANCE`, no commit; unknown identity → `EMPLOYEE_NOT_FOUND` in tests/LeaveManagement.UnitTests/Application/SubmitLeaveRequestHandlerTests.cs
- [ ] T056 [P] [US1] Write `GetOwnLeaveRequestHandlerTests`: own request → details; another employee's request id → `NOT_FOUND`; unknown id → `NOT_FOUND` in tests/LeaveManagement.UnitTests/Application/GetOwnLeaveRequestHandlerTests.cs
- [ ] T057 [P] [US1] Write `SubmitLeaveRequestApiTests` (quickstart #1, #2, #12; spec US1 #1–6): `201` with `Location` `/api/v1/leave-requests/{id}`, body has `id`, `employeeId`, `leaveType{code,name}`, `startDate`, `endDate`, `countedDays`, `reason`, `status=Pending`, `reviewRoute`, `submittedAt`; GET on `Location` returns identical body; Fri–Tue with Monday holiday → `countedDays=2`; single working day → 1; employee without manager → `FallbackReviewer`; database shows `ReservedDays` increased by `countedDays`; end date before start date → `400` with an error on `endDate` (never `500`) in tests/LeaveManagement.IntegrationTests/Api/SubmitLeaveRequestApiTests.cs

### Implementation for User Story 1

- [ ] T058 [P] [US1] Implement `WorkingDayCalculator.CountWorkingDays(DateOnly start, DateOnly end, IReadOnlySet<DateOnly> holidays)`: iterate each date inclusive, count Monday–Friday dates not in `holidays`; weekend days are a named constant set `{Saturday, Sunday}` (A12) in src/LeaveManagement.Domain/Calendar/WorkingDayCalculator.cs
- [ ] T059 [P] [US1] Implement `LeaveDateRange` value object (`Start`, `End`; constructor throws `ArgumentException` if `End < Start`; `IsWithinSingleYear`; `LeaveYear` = `Start.Year`; `Overlaps(LeaveDateRange other)` = `Start <= other.End && other.Start <= End`) in src/LeaveManagement.Domain/LeaveRequests/LeaveDateRange.cs
- [ ] T060 [US1] Add `Result Reserve(int days)` to `LeaveBalance`: if `days > AvailableDays` return `LEAVE_INSUFFICIENT_BALANCE` with message "The request needs {days} days but only {available} are available." and extensions `requestedDays`, `availableDays`; else `ReservedDays += days` in src/LeaveManagement.Domain/LeaveBalances/LeaveBalance.cs
- [ ] T061 [US1] Add `(ReviewRoute Route, Guid? ReviewerEmployeeId) DetermineReviewRoute()` to `Employee` (`Manager` + `ManagerId` when set, else `FallbackReviewer` + null) in src/LeaveManagement.Domain/Employees/Employee.cs
- [ ] T062 [US1] Add static `Result<LeaveRequest> Submit(Employee employee, Guid leaveTypeId, LeaveDateRange range, int countedDays, string reason, DateTimeOffset submittedAtUtc)` to `LeaveRequest`: fail `LEAVE_SPANS_LEAVE_YEARS` if not within one year ("Start and end dates must be in the same leave year. Submit one request per year."), fail `LEAVE_ZERO_WORKING_DAYS` if `countedDays <= 0` ("The selected period contains no leave days."); otherwise new `Guid` id, `Status = Pending`, trimmed reason, review route from `employee.DetermineReviewRoute()` in src/LeaveManagement.Domain/LeaveRequests/LeaveRequest.cs
- [ ] T063 [P] [US1] Create ports `ILeaveSubmissionStore` (`GetActiveLeaveTypeByCodeAsync(code)`, `GetBalanceAsync(employeeId, leaveTypeId, leaveYear)` returning a tracked entity or null, `AddRequest(LeaveRequest)`) and `IPublicHolidayCalendar` (`GetHolidayDatesAsync(DateOnly start, DateOnly end)` → `IReadOnlySet<DateOnly>`) in src/LeaveManagement.Application/Abstractions/ILeaveSubmissionStore.cs and src/LeaveManagement.Application/Abstractions/IPublicHolidayCalendar.cs
- [ ] T064 [P] [US1] Create `LeaveRequestDetails` record (all response fields from contracts/openapi.yaml `LeaveRequestResponse`), `SubmitLeaveRequestCommand` (`Issuer`, `Subject`, `LeaveTypeCode`, `StartDate` DateOnly, `EndDate` DateOnly, `Reason`) in src/LeaveManagement.Application/LeaveRequests/LeaveRequestDetails.cs and src/LeaveManagement.Application/LeaveRequests/Submit/SubmitLeaveRequestCommand.cs
- [ ] T065 [P] [US1] Create `SubmitLeaveRequestLog` source-generated log events: EventId 1001 `LeaveRequestSubmitted(Guid leaveRequestId, Guid employeeId, string leaveTypeCode, int countedDays, int leaveYear)` (Information), EventId 1002 `LeaveSubmissionRejected(Guid? employeeId, string errorCode)` (Information) — no reason parameter in src/LeaveManagement.Application/LeaveRequests/Submit/SubmitLeaveRequestLog.cs
- [ ] T066 [US1] Implement `SubmitLeaveRequestHandler.HandleAsync(command, ct)`: (1) `ICurrentEmployeeLookup` → else `EMPLOYEE_NOT_FOUND`; (2) active leave type by code → else `LEAVE_TYPE_INVALID`; (3) `IEmployeeUnitOfWork.ExecuteForEmployeeAsync` with work: build `LeaveDateRange`; *[US3 inserts start-in-past check here]*; spans-years check via `LeaveRequest.Submit` order (check `IsWithinSingleYear` first and return `LEAVE_SPANS_LEAVE_YEARS`); *[B1 slot — not implemented]*; load holidays, count working days, zero → `LEAVE_ZERO_WORKING_DAYS`; *[US3 inserts overlap check here]*; balance for `(employee, type, range.LeaveYear)` → null → `LEAVE_BALANCE_NOT_AVAILABLE` with extension `leaveYear`; `balance.Reserve(countedDays)`; `LeaveRequest.Submit(...)` with `OrganisationClock.UtcNow`; `store.AddRequest`; return `LeaveRequestDetails`; (4) after the unit of work returns, log 1001 on success or 1002 on failure in src/LeaveManagement.Application/LeaveRequests/Submit/SubmitLeaveRequestHandler.cs
- [ ] T067 [P] [US1] Implement `LeaveSubmissionStore : ILeaveSubmissionStore` (leave type query AsNoTracking filtered by `Code` and `IsActive`; balance query tracked by `(EmployeeId, LeaveTypeId, LeaveYear)`; `AddRequest` adds to `LeaveRequests`; no `SaveChanges` — the unit of work saves) in src/LeaveManagement.Infrastructure/Persistence/Stores/LeaveSubmissionStore.cs
- [ ] T068 [P] [US1] Implement `PublicHolidayCalendar : IPublicHolidayCalendar` (AsNoTracking dates where `Date >= start && Date <= end`, returned as `HashSet<DateOnly>`) in src/LeaveManagement.Infrastructure/Persistence/Stores/PublicHolidayCalendar.cs
- [ ] T069 [P] [US1] Create port `IOwnLeaveRequestQuery.GetAsync(Guid requestId, Guid employeeId)` returning `LeaveRequestDetails?` and implement `OwnLeaveRequestQuery` with ONE query filtering `Id == requestId && EmployeeId == employeeId` (AsNoTracking, includes leave type code/name) in src/LeaveManagement.Application/Abstractions/IOwnLeaveRequestQuery.cs and src/LeaveManagement.Infrastructure/Persistence/Stores/OwnLeaveRequestQuery.cs
- [ ] T070 [US1] Implement `GetOwnLeaveRequestHandler.HandleAsync(issuer, subject, requestId)`: employee lookup → `EMPLOYEE_NOT_FOUND`; query → null → `NOT_FOUND` (same result for missing and not owned) in src/LeaveManagement.Application/LeaveRequests/GetOwn/GetOwnLeaveRequestHandler.cs
- [ ] T071 [P] [US1] Create request DTO `SubmitLeaveRequestDto` (`LeaveTypeCode` `[Required]` `[MaxLength(50)]`; `StartDate`, `EndDate` strings `[Required]`; `Reason` `[Required]` `[MaxLength(500)]`; implements `IValidatableObject` that strictly parses `StartDate`/`EndDate` with `DateOnly.TryParseExact(value, "yyyy-MM-dd", CultureInfo.InvariantCulture, ...)` adding "Must be a valid date in yyyy-MM-dd format." per field; when both dates parse and `EndDate < StartDate`, adding "End date must not be earlier than start date." on `endDate` (FR-003); NO properties for employee id, status, counted days, or balance) and response DTO `LeaveRequestResponse` matching contracts/openapi.yaml in src/LeaveManagement.Api/Endpoints/LeaveRequests/SubmitLeaveRequestDto.cs and src/LeaveManagement.Api/Endpoints/LeaveRequests/LeaveRequestResponse.cs
- [ ] T072 [US1] Implement `LeaveRequestEndpoints.MapLeaveRequestEndpoints()`: group `/api/v1/leave-requests` with `RequireAuthorization("Employee")`; `POST /` reads identity via `TryGetIdentity` (missing → 403 `EMPLOYEE_NOT_FOUND`), maps DTO → command, calls handler, success → `TypedResults.Created($"/api/v1/leave-requests/{id}", response)`, failure → `DomainErrorProblemMapper`; `GET /{id:guid}` → 200 or mapped error; declare OpenAPI metadata (`Produces` 201/200 and `ProducesProblem` 400, 401, 403, 404, 409, 422, 500, 503 per contracts/openapi.yaml); call from `Program.cs` in src/LeaveManagement.Api/Endpoints/LeaveRequests/LeaveRequestEndpoints.cs
- [ ] T073 [US1] Register `SubmitLeaveRequestHandler`, `GetOwnLeaveRequestHandler` (scoped) in src/LeaveManagement.Application/DependencyInjection.cs and `ILeaveSubmissionStore`, `IPublicHolidayCalendar`, `IOwnLeaveRequestQuery` (scoped) in src/LeaveManagement.Infrastructure/DependencyInjection.cs
- [ ] T074 [US1] Implement `DevelopmentDataSeeder` (runs only when environment is Development and `Seed:Enabled` is true; idempotent): manager M; employee A (issuer `dotnet-user-jwts`, subject `employee-a`, manager M); employee B (subject `employee-b`, no manager); leave types `ANNUAL` and `SICK` active and `LEGACY` inactive; balances current year A/ANNUAL entitlement 5, B/ANNUAL entitlement 10; a few public holidays in the current year; invoke from `Program.cs` in src/LeaveManagement.Infrastructure/Development/DevelopmentDataSeeder.cs

**Checkpoint**: US1 tests (T051–T057) pass; the MVP can be demonstrated with quickstart #1, #2, #12.

---

## Phase 4: User Story 2 — Enforce available leave balance (Priority: P2)

**Goal**: Balance rules are complete and proven, including future leave years, leave-year
spanning, and simultaneous submissions, with correct `422`/`409`/`503` responses and full rollback.

**Independent Test**: Give an employee a known balance and submit below, equal to, and above it,
plus two simultaneous 3-day requests on a 5-day balance → exactly one `201` and one
`422 LEAVE_INSUFFICIENT_BALANCE` with `requestedDays`/`availableDays`; reserved days never exceed
the entitlement. (spec US2 #1–8, FR-011–FR-013, FR-018, FR-018a, FR-020, FR-020a)

### Tests for User Story 2

- [ ] T075 [P] [US2] Extend `SubmitLeaveRequestHandlerTests` with: request equal to available → success; future-year request with existing next-year balance reserves against that year only (current-year balance unchanged); future-year without balance → `LEAVE_BALANCE_NOT_AVAILABLE` with `leaveYear`; 30 Dec–2 Jan → `LEAVE_SPANS_LEAVE_YEARS` even when one side counts zero days; existing Pending reservation of 6 on 10 → 5-day request fails with `availableDays=4` in tests/LeaveManagement.UnitTests/Application/SubmitLeaveRequestHandlerTests.cs
- [ ] T076 [P] [US2] Write `BalanceRulesApiTests` (quickstart #3, #10, #11; spec US2 #1–3, #5–8): `422` Problem Details with `errorCode`, `traceId`, integer `requestedDays`/`availableDays`; `LEAVE_BALANCE_NOT_AVAILABLE` with `leaveYear`; `LEAVE_SPANS_LEAVE_YEARS`; balances of different leave types never combined; no request row and unchanged balance after every rejection in tests/LeaveManagement.IntegrationTests/Api/BalanceRulesApiTests.cs
- [ ] T077 [P] [US2] Write `ConcurrentSubmissionTests` (balance part): using separate `HttpClient` calls released by a `Barrier`, (a) 5 available, two simultaneous 3-day requests → exactly one `201`, one `422 LEAVE_INSUFFICIENT_BALANCE`, `ReservedDays = 3`; (b) two identical simultaneous requests → exactly one request row and one reservation; (c) two different employees submitting simultaneously → both `201`; repeat each case 5 times in tests/LeaveManagement.IntegrationTests/Concurrency/ConcurrentSubmissionTests.cs
- [ ] T078 [P] [US2] Write `RollbackTests`: register a test `SaveChangesInterceptor` whose `SavedChangesAsync` throws a non-transient exception — i.e., after the `INSERT LeaveRequests` and `UPDATE LeaveBalances` statements have executed inside the open transaction and before commit; assert: response `500`; no `LeaveRequests` row and unchanged `ReservedDays` and `RowVersion` when read on a separate connection; the employee lock is released (a follow-up submission for the same employee completes within the lock timeout); a normal retry then succeeds exactly once (FR-015, FR-018, FR-018a) in tests/LeaveManagement.IntegrationTests/Persistence/RollbackTests.cs
- [ ] T079 [P] [US2] Write `ConflictAndTimeoutTests`: (a) hold the employee lock in a separate open transaction → submission returns `503 SUBMISSION_TIMEOUT` (500 ms test lock timeout) and nothing is saved; (b) inside an `EmployeeUnitOfWork` work delegate, update the same balance through a separate connection (bypassing the protocol) → result `CONCURRENCY_CONFLICT`, mapped to `409`, nothing saved in tests/LeaveManagement.IntegrationTests/Persistence/ConflictAndTimeoutTests.cs

### Implementation for User Story 2

- [ ] T080 [US2] Extend `EmployeeUnitOfWork`: catch `SqlException` with `SqlServerErrors` lock-timeout (1222) → `SUBMISSION_TIMEOUT` ("The request could not be processed in time. Nothing was saved."), catch `DbUpdateConcurrencyException` → `CONCURRENCY_CONFLICT` ("The data changed while your request was processed. Nothing was saved; please retry."); ensure rollback in both cases; leave deadlocks (1205) to the execution strategy retry in src/LeaveManagement.Infrastructure/Persistence/EmployeeUnitOfWork.cs
- [ ] T081 [P] [US2] Create `PersistenceLog` source-generated events: EventId 3001 `ConcurrencyConflict(Guid employeeId)` (Warning), EventId 3002 `LockTimeout(Guid employeeId, int timeoutMs)` (Warning); call them from `EmployeeUnitOfWork` in src/LeaveManagement.Infrastructure/Persistence/PersistenceLog.cs
- [ ] T082 [US2] Verify and, if needed, fix `DomainErrorProblemMapper` so `requestedDays`, `availableDays`, `leaveYear` are emitted as JSON integers and `409`/`503` include `errorCode` (make T076/T079 pass) in src/LeaveManagement.Api/ErrorHandling/DomainErrorProblemMapper.cs

**Checkpoint**: US1 + US2 tests pass; concurrency tests pass on 5 consecutive runs.

---

## Phase 5: User Story 3 — Receive useful feedback for invalid requests (Priority: P3)

**Goal**: Every input problem is reported together in one `400`; business problems (start in
past, zero days, overlap) return the documented `422` codes; overlap is enforced on calendar dates
across leave types, including under simultaneous submissions; retries are recognisable.

**Independent Test**: Submit `{}` → one `400` listing all four fields; submit end-before-start,
bad date format, whitespace reason, inactive leave type → field errors; weekend-only →
`422 LEAVE_ZERO_WORKING_DAYS`; overlapping an existing Pending request of another type →
`422 LEAVE_OVERLAP` with `overlappingRequestId`; nothing is created in any case. (spec US3 #1–8,
FR-002–FR-004, FR-014, FR-016, FR-017, FR-018a)

### Tests for User Story 3

- [ ] T083 [P] [US3] Write `LeaveRequestOverlapPolicyTests`: shared single date blocks; shared weekend-only dates block (existing Fri–Sun, new Sat–Mon); adjacent ranges (existing ends Fri, new starts Mon) allowed; different leave types still block; `Rejected` and `Cancelled` ignored; returns the first overlapping request id in tests/LeaveManagement.UnitTests/Domain/LeaveRequestOverlapPolicyTests.cs
- [ ] T084 [P] [US3] Extend `SubmitLeaveRequestHandlerTests`: start date before `OrganisationClock.Today` (FakeTimeProvider + time zone) → `LEAVE_START_IN_PAST`, start today allowed; overlap → `LEAVE_OVERLAP` with `overlappingRequestId`; retry of an accepted request where the balance is now exhausted returns `LEAVE_OVERLAP` (not insufficient balance) — overlap is evaluated before balance in tests/LeaveManagement.UnitTests/Application/SubmitLeaveRequestHandlerTests.cs
- [ ] T085 [P] [US3] Write `ValidationApiTests` (quickstart #4–#9, #18; spec US3 #1–8): `{}` → 400 with errors for `leaveTypeCode`, `startDate`, `endDate`, `reason` in one response; end before start → error on `endDate` "End date must not be earlier than start date."; `startDate` `12/03/2027` → error on `startDate` plus other field errors reported together; whitespace-only reason → "Reason is required."; 501-character reason → length error; inactive `LEGACY` type → `422 LEAVE_TYPE_INVALID`; malformed JSON → 400 Problem Details; weekend-only → `422 LEAVE_ZERO_WORKING_DAYS`; past start → `422 LEAVE_START_IN_PAST`; overlap with existing Pending of another type → `422 LEAVE_OVERLAP`; exact retry of a created request → `422 LEAVE_OVERLAP` whose `overlappingRequestId` GET returns the original; Rejected/Cancelled request on the same dates does not block; no response body or validation message contains the submitted reason text in tests/LeaveManagement.IntegrationTests/Api/ValidationApiTests.cs
- [ ] T086 [P] [US3] Write `OverlapQueryTests`: store query returns Pending and Approved requests of the employee intersecting the range (any leave type), excludes Rejected/Cancelled, other employees, and non-intersecting ranges in tests/LeaveManagement.IntegrationTests/Persistence/OverlapQueryTests.cs
- [ ] T087 [US3] Extend `ConcurrentSubmissionTests` with overlap cases: two simultaneous requests of DIFFERENT leave types sharing one date (ample balance on both) → exactly one `201`, one `422 LEAVE_OVERLAP`; repeat 5 times in tests/LeaveManagement.IntegrationTests/Concurrency/ConcurrentSubmissionTests.cs

### Implementation for User Story 3

- [ ] T088 [P] [US3] Implement `LeaveRequestOverlapPolicy.FindOverlap(LeaveDateRange range, IEnumerable<LeaveRequest> existing)` → first request with `Status` `Pending` or `Approved` whose range overlaps on calendar dates, else null in src/LeaveManagement.Domain/LeaveRequests/LeaveRequestOverlapPolicy.cs
- [ ] T089 [US3] Add `GetOverlappingActiveRequestsAsync(Guid employeeId, DateOnly start, DateOnly end)` to `ILeaveSubmissionStore` and implement it in `LeaveSubmissionStore` (`EmployeeId == employeeId && (Status == Pending || Status == Approved) && StartDate <= end && EndDate >= start`, uses `IX_LeaveRequests_Overlap`) in src/LeaveManagement.Application/Abstractions/ILeaveSubmissionStore.cs and src/LeaveManagement.Infrastructure/Persistence/Stores/LeaveSubmissionStore.cs
- [ ] T090 [US3] Update `SubmitLeaveRequestHandler` work delegate: first rule `range.Start < OrganisationClock.Today` → `LEAVE_START_IN_PAST` ("The start date cannot be in the past."); after the zero-days check and BEFORE the balance lookup, call the store + `LeaveRequestOverlapPolicy` → `LEAVE_OVERLAP` ("These dates overlap your existing leave request.") with extension `overlappingRequestId`; update the in-memory fake store for the new method in src/LeaveManagement.Application/LeaveRequests/Submit/SubmitLeaveRequestHandler.cs
- [ ] T091 [US3] Complete `SubmitLeaveRequestDto.Validate`: treat whitespace-only `leaveTypeCode`/`reason` as missing ("Leave type is required." / "Reason is required."); reason length checked after trim (max 500, message "Reason must be at most 500 characters."); return ALL errors (never short-circuit); messages never include submitted values in src/LeaveManagement.Api/Endpoints/LeaveRequests/SubmitLeaveRequestDto.cs

**Checkpoint**: US1–US3 tests pass; quickstart #3–#11 and #18 behave as documented.

---

## Phase 6: User Story 4 — Protect employee identity and server-owned data (Priority: P4)

**Goal**: The actor is always the employee mapped from the token's issuer + subject; spoofed
identifiers and server-owned fields have no effect; unauthenticated, unknown, and inactive
callers are refused; employees can only read their own requests.

**Independent Test**: No token → `401`; token for an unmapped subject or an inactive employee →
`403 EMPLOYEE_NOT_FOUND`; same subject with a different issuer → `403`; a body containing another
employee's id, `status=Approved`, `countedDays=1` creates a `Pending` request for the caller with
server-computed days; GET of another employee's request → `404` identical to a random id.
(spec US4 #1–4, FR-007–FR-009, T4, T6)

### Tests for User Story 4

- [ ] T092 [P] [US4] Write `IdentityApiTests` (quickstart #13–#15, #17): 401 without identity headers (POST and GET); 403 `EMPLOYEE_NOT_FOUND` for unmapped subject, for inactive employee, and for a mapped subject presented with a different issuer; spoofed `employeeId` (of another employee), `status`, `countedDays`, `availableDays` in the body are ignored → `201` owned by the caller, `Pending`, server-computed `countedDays`, other employee's balance unchanged; GET of another employee's request and GET of a random GUID both `404` with identical bodies apart from `traceId`; every 401 response has content type `application/problem+json`, a `traceId`, and a `WWW-Authenticate: Bearer` header in tests/LeaveManagement.IntegrationTests/Api/IdentityApiTests.cs
- [ ] T093 [P] [US4] Write `DeactivationRaceTests`: deactivate the employee after `ICurrentEmployeeLookup` succeeds but before the unit of work runs (call `EmployeeUnitOfWork` directly with a deactivated id) → `EMPLOYEE_NOT_FOUND`, no request, no reservation in tests/LeaveManagement.IntegrationTests/Persistence/DeactivationRaceTests.cs
- [ ] T094 [P] [US4] Write `ClaimsMappingTests` for `ClaimsPrincipalExtensions.TryGetIdentity`: default `sub`; configured `oid`; missing `iss` or subject claim → false; email/`upn` claims present but subject missing → false in tests/LeaveManagement.IntegrationTests/Api/ClaimsMappingTests.cs

### Implementation for User Story 4

- [ ] T095 [US4] In `LeaveRequestEndpoints`, log `ApiLog.AuthorizationDenied("NoEmployeeRecord")` whenever an authenticated caller maps to no active employee (missing claims or `EMPLOYEE_NOT_FOUND`), without logging issuer or subject; confirm the minimal API JSON options keep the default of ignoring unknown properties (do not enable `JsonUnmappedMemberHandling.Disallow`) in src/LeaveManagement.Api/Endpoints/LeaveRequests/LeaveRequestEndpoints.cs

**Checkpoint**: All four user stories pass independently and together.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Observability evidence, contract completeness, performance measurement, and final
validation.

- [ ] T096 [P] Write `LoggingRedactionTests`: submit with reason `ZZ-SECRET-REASON-ZZ` for (a) success, (b) 400 validation failure, (c) 422 rejection, (d) forced 500 via interceptor; assert no captured log message or structured value contains the string, and no log contains the test subject value (FR-019, Constitution VII) in tests/LeaveManagement.IntegrationTests/Api/LoggingRedactionTests.cs
- [ ] T097 [P] Write `OpenApiDocumentTests`: in Development environment GET `/openapi/v1.json` → contains `POST /api/v1/leave-requests` with responses 201, 400, 401, 403, 409, 422, 500, 503 and `GET /api/v1/leave-requests/{id}` with 200, 401, 403, 404, a bearer security scheme, and `GET /health/live` and `GET /health/ready` with no security requirement (Constitution IV, VI) in tests/LeaveManagement.IntegrationTests/Api/OpenApiDocumentTests.cs
- [ ] T098 [P] Create load-test console app (no third-party packages; `HttpClient` + `Stopwatch`) that runs 50 concurrent virtual employees submitting valid requests against a configured base URL and token list, then prints p50/p95/max durations for accepted and rejected outcomes; not part of CI; usage in a header comment in tools/LeaveManagement.LoadTest/Program.cs
- [ ] T099 [P] Create root `README.md`: purpose, prerequisites (link quickstart.md and research.md R13 for Docker/CI), user-secrets connection string setup, build/test/run commands, link to the feature spec and constitution, and a "Production blockers" list (B3, B4, identity provider, CI platform/SQL Server version) in README.md
- [ ] T100 Run quickstart.md sections 1–5 (scenarios 1–18) and the load test from T098 against a local database; record pass/fail per scenario and the measured p95 versus SC-001 (< 2 s at 50 concurrent) in specs/001-submit-leave-request/checklists/quickstart-results.md
- [ ] T101 Perform the constitution compliance review: build with 0 warnings, all unit and integration tests green, confirm every write path uses `IEmployeeUnitOfWork` (L1–L6), no reason text in logs, OpenAPI matches contracts/openapi.yaml, and every item of the Release Gate is met; record findings in specs/001-submit-leave-request/checklists/constitution-review.md

---

## Phase 8: B1 — Holiday calendar completeness (⛔ BLOCKED: pending business approval)

**Purpose**: Fail closed when the holiday calendar for the leave year is not verifiably complete
(research.md R16). **Do not start any task in this phase until B1 is approved by the business
AND T102 has updated the spec.** If B1 is rejected, delete this phase.

- [ ] T102 BLOCKED (B1) After approval, add FR-022 ("A request MUST be rejected with `LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE` when the public-holiday calendar for its leave year is not marked complete; nothing is reserved; an operational warning is logged without the reason") and an acceptance scenario to US2, and record the decision under Clarifications in specs/001-submit-leave-request/spec.md
- [ ] T103 BLOCKED (B1) Create `PublicHolidayCalendarYear` entity (`Year` int primary key, `CompletedAt` DateTimeOffset) in src/LeaveManagement.Domain/Calendar/PublicHolidayCalendarYear.cs
- [ ] T104 BLOCKED (B1) Create `PublicHolidayCalendarYearConfiguration` (table `PublicHolidayCalendarYears`, `Year` PK without identity) and generate migration `AddHolidayCalendarCompleteness` (`--startup-project src/LeaveManagement.Infrastructure`, as in T022) in src/LeaveManagement.Infrastructure/Persistence/Configurations/PublicHolidayCalendarYearConfiguration.cs
- [ ] T105 BLOCKED (B1) Add `LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE` to `ErrorCodes` and map it to 422 with extension `leaveYear` in src/LeaveManagement.Domain/Common/ErrorCodes.cs and src/LeaveManagement.Api/ErrorHandling/DomainErrorProblemMapper.cs
- [ ] T106 BLOCKED (B1) Add `IsYearCompleteAsync(int year)` to `IPublicHolidayCalendar` and implement it in `PublicHolidayCalendar` (true only when a marker row exists; database errors propagate, never treated as "no holidays") in src/LeaveManagement.Infrastructure/Persistence/Stores/PublicHolidayCalendar.cs
- [ ] T107 BLOCKED (B1) Insert the calendar-complete rule in `SubmitLeaveRequestHandler` immediately after the spans-years check and before counting days; on failure return the new code with `leaveYear` and log Warning EventId 1003 `HolidayCalendarUnavailable(int leaveYear, Guid employeeId)` (trace id via scope, no reason) in src/LeaveManagement.Application/LeaveRequests/Submit/SubmitLeaveRequestHandler.cs
- [ ] T108 BLOCKED (B1) Add tests (handler unit test, API test returning `422 LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE` with nothing reserved, log assertion without reason), update `TestDataBuilder` and `DevelopmentDataSeeder` to mark used years complete, and add the B1 scenario to quickstart.md in tests/LeaveManagement.IntegrationTests/Api/HolidayCalendarCompletenessTests.cs

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: depends on Setup; BLOCKS all user stories.
- **US1 (Phase 3)**: depends on Foundational. MVP.
- **US2 (Phase 4)**: depends on US1 (uses the submit handler, endpoint, and `LeaveBalance.Reserve`).
- **US3 (Phase 5)**: depends on US1 (extends the handler and DTO). Independent of US2, except T087
  extends the file created in T077 — run T087 after T077.
- **US4 (Phase 6)**: depends on US1 (endpoints exist). Independent of US2/US3.
- **Polish (Phase 7)**: depends on all desired stories.
- **B1 (Phase 8)**: BLOCKED on business approval; when unblocked, depends on US1 and US2.

```text
Setup → Foundational → US1 ─┬─> US2 ─┐
                            ├─> US3 ─┼─> Polish
                            └─> US4 ─┘
                     (B1: blocked; after US2 when approved)
```

### Within each phase

- T022 depends on T016–T021 only (design-time factory, no Api dependency); T041 depends on T022.
- Tests are listed first; write them before or alongside implementation (Constitution III requires
  them before the story is complete).
- Domain types → Application ports/handlers → Infrastructure implementations → API endpoints.
- `SubmitLeaveRequestHandler.cs` is edited by T066 (US1), T090 (US3), and T107 (B1): never in
  parallel.
- `SubmitLeaveRequestHandlerTests.cs` is edited by T055, T075, T084: sequential.
- `EmployeeUnitOfWork.cs` is edited by T030 and T080: sequential.

### Parallel Opportunities

- Setup: T003–T006 together after T002.
- Foundational: T007–T014 (Domain) together; T017–T021 (configurations) together after T015–T016;
  T023–T027 together; T033–T036 and T038 together; T041–T043, T045–T049 together.
- US1: tests T051–T054, T056, T057 together; implementation T058, T059, T063–T065, T067–T069, T071
  together.
- US2: T075–T079 together; T081 alongside T080.
- US3: T083–T086 together; T088 alongside T089.
- US4: T092–T094 together.
- Polish: T096–T099 together.
- After US1, US3 and US4 can proceed in parallel with US2 by different developers.

---

## Parallel Example: User Story 1

```text
# Domain and Application tests together:
Task: "T051 WorkingDayCalculatorTests in tests/LeaveManagement.UnitTests/Domain/WorkingDayCalculatorTests.cs"
Task: "T052 LeaveBalanceTests in tests/LeaveManagement.UnitTests/Domain/LeaveBalanceTests.cs"
Task: "T053 LeaveRequestTests in tests/LeaveManagement.UnitTests/Domain/LeaveRequestTests.cs"
Task: "T057 SubmitLeaveRequestApiTests in tests/LeaveManagement.IntegrationTests/Api/SubmitLeaveRequestApiTests.cs"

# Independent implementation files together:
Task: "T058 WorkingDayCalculator in src/LeaveManagement.Domain/Calendar/WorkingDayCalculator.cs"
Task: "T059 LeaveDateRange in src/LeaveManagement.Domain/LeaveRequests/LeaveDateRange.cs"
Task: "T067 LeaveSubmissionStore in src/LeaveManagement.Infrastructure/Persistence/Stores/LeaveSubmissionStore.cs"
Task: "T068 PublicHolidayCalendar in src/LeaveManagement.Infrastructure/Persistence/Stores/PublicHolidayCalendar.cs"
Task: "T071 DTOs in src/LeaveManagement.Api/Endpoints/LeaveRequests/"
```

## Parallel Example: User Story 2

```text
Task: "T076 BalanceRulesApiTests in tests/LeaveManagement.IntegrationTests/Api/BalanceRulesApiTests.cs"
Task: "T077 ConcurrentSubmissionTests in tests/LeaveManagement.IntegrationTests/Concurrency/ConcurrentSubmissionTests.cs"
Task: "T078 RollbackTests in tests/LeaveManagement.IntegrationTests/Persistence/RollbackTests.cs"
Task: "T079 ConflictAndTimeoutTests in tests/LeaveManagement.IntegrationTests/Persistence/ConflictAndTimeoutTests.cs"
```

## Parallel Example: User Story 3

```text
Task: "T083 LeaveRequestOverlapPolicyTests in tests/LeaveManagement.UnitTests/Domain/LeaveRequestOverlapPolicyTests.cs"
Task: "T085 ValidationApiTests in tests/LeaveManagement.IntegrationTests/Api/ValidationApiTests.cs"
Task: "T086 OverlapQueryTests in tests/LeaveManagement.IntegrationTests/Persistence/OverlapQueryTests.cs"
Task: "T088 LeaveRequestOverlapPolicy in src/LeaveManagement.Domain/LeaveRequests/LeaveRequestOverlapPolicy.cs"
```

## Parallel Example: User Story 4

```text
Task: "T092 IdentityApiTests in tests/LeaveManagement.IntegrationTests/Api/IdentityApiTests.cs"
Task: "T093 DeactivationRaceTests in tests/LeaveManagement.IntegrationTests/Persistence/DeactivationRaceTests.cs"
Task: "T094 ClaimsMappingTests in tests/LeaveManagement.IntegrationTests/Api/ClaimsMappingTests.cs"
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup → Phase 2 Foundational (includes the employee lock and single transaction, so the
   MVP is already concurrency-safe).
2. Phase 3 US1.
3. **STOP and VALIDATE**: US1 tests plus quickstart #1, #2, #12.
4. Demo internally. Do not release — see Release Gate.

### Incremental Delivery

1. Setup + Foundational → foundation ready.
2. US1 → MVP demo.
3. US2 → balance and concurrency evidence (not yet releasable: FR-014 overlap is not implemented).
4. US3 → complete feedback and overlap rules.
5. US4 → identity hardening evidence.
6. Polish → logs, contract, performance, quickstart and constitution review.
7. B1 → only if approved.

### Release Gate (first release to any environment beyond development)

All of the following MUST be true:

1. US1–US4 complete (all functional requirements implemented).
2. Passing on real SQL Server (Testcontainers): T049 (constraints), T050 (lock), T077 (balance
   concurrency and duplicates), T078 (rollback), T079 (timeout/conflict), T087 (cross-type overlap
   concurrency), T092 (identity).
3. Error-format and contract items verified by T092 (401 Problem Details) and T097 (OpenAPI incl.
   health endpoints).
4. T101 constitution review recorded with no open violations.

Production additionally requires B3, B4, the identity provider, and the CI platform to be resolved.

### Parallel Team Strategy

1. Team completes Setup + Foundational together.
2. One developer completes US1 (the shared handler).
3. Then: Developer A → US2, Developer B → US3 (coordinate on `ConcurrentSubmissionTests.cs` and the
   handler file), Developer C → US4.

---

## Notes

- Unconfirmed assumptions implemented as written (B2): A2 whole days (T013, T058), A3 every type has
  a balance (T066), A4 no past start (T090), A5 reason ≤ 500 (T071, T091), A7 no notice period (no
  task), A11 single calendar (T014, T068), A12 Sat/Sun weekend (T058). Each lives in one place so a
  business change stays local.
- Production blockers (not tasks in this feature): B3 data sources, B4 organisation time zone,
  identity provider selection (T4), CI platform and production SQL Server version (T5).
- No secrets in source control: connection strings via user secrets; test containers generate their
  own passwords.
- Commit after each task or logical group once a git repository exists.
