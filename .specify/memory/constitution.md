<!--
Sync Impact Report
==================
Version change: (unversioned template) → 1.0.0
Bump rationale: Initial ratification; all placeholders replaced with project-specific content.

Modified principles (template placeholder → new title):
- [PRINCIPLE_1_NAME] → I. Clean Architecture Boundaries
- [PRINCIPLE_2_NAME] → II. Readable, Purposeful Code
- [PRINCIPLE_3_NAME] → III. Tested Business Rules and Behavior
- [PRINCIPLE_4_NAME] → IV. Server-Side Trust and Security
- [PRINCIPLE_5_NAME] → V. Data Integrity and Concurrency Safety
Added principles:
- VI. Consistent, Documented API
- VII. Safe, Actionable Observability
- VIII. Explicit, Maintainable Design

Added sections:
- Technology Stack & Constraints (was [SECTION_2_NAME])
- Development Workflow & Quality Gates (was [SECTION_3_NAME])
- Governance (filled from [GOVERNANCE_RULES])

Removed sections: none

Follow-up TODOs / items for owner review:
- RATIFICATION_DATE set to 2026-10-17 (date of first adoption); confirm.
- Authentication provider/token format not yet chosen (Principle IV defers it to the first plan).
- Concurrency mechanism defaults to optimistic concurrency (Principle V); confirm.
- Amendment approver defined as "project owner"; confirm who that is.

This report is review scratch material; remove it before committing.
-->

# Leave Management API Constitution

## Core Principles

### I. Clean Architecture Boundaries

The solution MUST follow Clean Architecture with four layers and inward-only dependencies:

- **Domain**: entities, value objects, domain services, and business rules (leave types, balances,
  approval state transitions). It MUST NOT reference EF Core, ASP.NET Core, SQL Server, or any
  other infrastructure or web package.
- **Application**: use cases, orchestration, ports (interfaces) for persistence and external
  services, and request/response models. It depends only on Domain.
- **Infrastructure**: EF Core `DbContext`, entity configurations, migrations, and adapters that
  implement Application ports.
- **API**: Minimal API endpoint definitions, HTTP mapping, authentication/authorization wiring,
  and composition root. Endpoints MUST stay thin: translate HTTP to a use case call and the result
  back to HTTP. Business decisions MUST NOT live in endpoint handlers or EF configurations.

Layer boundaries MUST be enforced by project references; an automated architecture test (or
equivalent build-time check) SHOULD guard against forbidden dependencies.

**Rationale**: Leave policies are the core value of the system and change for business reasons,
not technical ones. Isolating them keeps rules testable without a database or web host and lets
infrastructure evolve without rewriting policy.

**Expectations**: Each plan identifies which layer every new type belongs to. A Domain type that
needs persistence or HTTP knowledge is a design smell that MUST be resolved before merge.

### II. Readable, Purposeful Code

- Code MUST optimize for the next reader: descriptive names in the business vocabulary (e.g.,
  `LeaveRequest`, `ApproveLeave`), small focused methods, and no clever tricks without comment.
- Every abstraction (interface, base class, generic helper, pipeline library) MUST have a current,
  concrete reason to exist: a second implementation, a test seam at a layer boundary, or removal
  of real duplication. Speculative abstractions ("we might need it later") are not permitted.
- Nullable reference types MUST be enabled and compiler warnings treated as errors in all
  projects. A shared formatting/analyzer configuration (`.editorconfig`) MUST be applied.
- Dead code, commented-out code, and unused dependencies MUST be removed rather than kept.

**Rationale**: A small API with indirect, over-layered code is harder to change than one with
plain code. Clean Architecture provides the necessary seams; anything beyond that must earn its
place.

**Expectations**: Reviewers may reject an abstraction by asking "what does this buy us today?".
Libraries such as MediatR, AutoMapper, or generic repositories are allowed only when a plan
justifies them in its Complexity Tracking section.

### III. Tested Business Rules and Behavior

- Every business rule in Domain and every use case in Application MUST have xUnit unit tests
  covering the success path, each rejection reason, and relevant boundaries (e.g., zero or
  insufficient balance, overlapping dates, invalid state transitions).
- Database behavior (mappings, constraints, transactions, concurrency handling) MUST be covered by
  integration tests against a real SQL Server engine (e.g., containerized). The EF Core in-memory
  provider MUST NOT be used to verify persistence behavior.
- Each endpoint MUST have integration tests through the HTTP pipeline (e.g.,
  `WebApplicationFactory`) verifying status codes, error response shape, and authorization
  outcomes, including at least one forbidden/unauthorized case.
- A bug fix MUST include a test that fails without the fix.
- Tests MUST be deterministic: no reliance on wall-clock time, test ordering, or shared mutable
  state. Time MUST be injected (e.g., `TimeProvider`).

**Rationale**: Leave balances affect pay and trust. Unit tests make rules cheap to verify;
integration tests catch the failures unit tests cannot—SQL translation, constraints, transactions,
and auth wiring.

**Expectations**: Writing tests first is encouraged but not mandated. A feature is not complete
until its unit and integration tests pass in CI. Code-coverage percentages are informative, not a
gate.

### IV. Server-Side Trust and Security

- All input MUST be validated on the server at the API boundary (shape, ranges, formats) and
  business invariants MUST be re-enforced in Domain. Client-side validation is never sufficient.
- Every endpoint MUST require authentication unless explicitly documented as anonymous, and MUST
  apply authorization policies appropriate to the action (e.g., employee, manager, HR/admin).
  Resource-level checks MUST confirm the caller may act on the specific record (e.g., a manager
  approves only requests from their reports; an employee views only their own requests).
- The acting employee's identity MUST be derived from the authenticated principal, never from
  request bodies, query strings, or headers controlled by the client.
- Leave balances, approval status, approver identity, and other server-owned values MUST be
  computed or loaded on the server. Client-supplied values for these fields MUST be ignored or
  rejected.
- Request models MUST be explicit DTOs; EF entities MUST NOT be bound directly from requests
  (prevents over-posting).
- Secrets and connection strings MUST NOT be committed; use user secrets locally and a secure
  configuration source in deployed environments. All data access MUST be parameterized (EF Core
  or parameterized SQL only).

**Rationale**: A leave system is a natural target for privilege escalation (self-approval,
inflating balances, acting as another employee). Trusting the server alone closes these paths.

**Expectations**: Each spec lists roles and who may perform each action. The concrete
authentication mechanism (e.g., JWT bearer with an identity provider) is selected in the first
implementation plan and recorded there.

### V. Data Integrity and Concurrency Safety

- Operations that change a leave balance or request state (submit, approve, reject, cancel,
  adjust) MUST execute as a single atomic unit: all related database writes succeed or none do.
- Concurrent operations on the same balance or request MUST NOT produce lost updates, double
  deductions, or negative balances. The default mechanism is optimistic concurrency (SQL Server
  `rowversion` concurrency tokens) with conflicts surfaced as `409 Conflict`; a plan MAY choose a
  different mechanism if it documents why.
- Approval workflows MUST be modeled as explicit state transitions in Domain. Invalid transitions
  (e.g., approving an already rejected or cancelled request, self-approval, approving without
  sufficient balance) MUST be rejected by Domain logic, not only by the UI or endpoint.
- Database constraints (keys, foreign keys, uniqueness, non-null, check constraints where
  practical) MUST back critical invariants as a last line of defense.
- Schema changes MUST be made through reviewed EF Core migrations; manual production schema edits
  are prohibited.

**Rationale**: Balances are financial-grade data. Race conditions and partial writes are silent,
costly, and hard to reconcile after the fact.

**Expectations**: Each plan touching balances or approvals states its transaction boundary and
concurrency strategy, and includes an integration test that simulates a concurrent conflict.

### VI. Consistent, Documented API

- Endpoints MUST use resource-oriented routes and correct HTTP semantics: `200`/`201`/`204` for
  success (with `Location` on create), `400` for malformed or invalid input, `401` unauthenticated,
  `403` forbidden, `404` not found (or when existence must not be revealed), `409` for state or
  concurrency conflicts, `422` MAY be used for business-rule violations if adopted consistently.
- All error responses MUST use a single format: RFC 9457 Problem Details (`application/problem+json`),
  with validation errors listing field-level messages and a stable, machine-readable error code
  for business-rule failures.
- Unhandled exceptions MUST be converted to a generic `500` Problem Details response with no stack
  traces or internal details.
- Every endpoint MUST be described in the generated OpenAPI document, including request/response
  schemas, possible status codes, and security requirements. The OpenAPI document is part of the
  contract and changes to it MUST be reviewed.
- The API MUST be versioned from the first release (e.g., `/api/v1`); breaking contract changes
  require a new version.

**Rationale**: Predictable responses let clients handle errors generically and reduce integration
defects. OpenAPI keeps documentation and behavior aligned.

**Expectations**: Specs and plans define contracts (routes, payloads, status codes) before
implementation; integration tests assert both status code and Problem Details shape.

### VII. Safe, Actionable Observability

- Logging MUST use structured logging via `ILogger` with message templates and named properties,
  not string concatenation.
- Every request MUST carry a correlation/trace identifier that appears in logs and is returned in
  error responses, so a reported error can be located in logs.
- Business-significant events (request submitted, approved, rejected, cancelled, balance adjusted,
  concurrency conflict, authorization denial) MUST be logged at an appropriate level with the
  relevant identifiers.
- Logs and error responses MUST NOT contain secrets, tokens, connection strings, full request
  bodies, or sensitive personal data (e.g., medical reasons for leave). Use internal IDs rather
  than names or emails.
- Health check endpoints MUST be provided for the application and its database dependency.

**Rationale**: Operators need to diagnose failures quickly; employees' personal and health
information must not leak into log stores, which are usually less protected than the database.

**Expectations**: A log line is "actionable" when it answers what happened, to which entity, and
why. The logging sink and tracing backend (e.g., OpenTelemetry exporter) are plan-level choices.

### VIII. Explicit, Maintainable Design

- Business rules MUST be expressed explicitly in named Domain methods, policies, or value objects
  (e.g., `LeaveBalance.Deduct`, `LeaveRequest.Approve`), not hidden in LINQ queries, EF
  configuration, attributes, or endpoint code.
- Dependencies MUST be supplied through constructor injection via the built-in ASP.NET Core
  container. Service locator patterns and static mutable state are prohibited.
- Each class and project MUST have a single clear responsibility; configuration values MUST use
  strongly typed options classes validated at startup.
- Magic numbers and strings for policy values (allowances, carry-over limits, notice periods) MUST
  be named constants, configuration, or data—never inline literals scattered across layers.

**Rationale**: Leave policies change (new leave types, regional rules, carry-over). Explicit,
well-separated rules let a developer find and change one policy in one place with confidence.

**Expectations**: When reviewing, "where is this rule enforced?" must have a single obvious answer.

## Technology Stack & Constraints

- **Runtime/Language**: .NET 10, C# with nullable reference types enabled.
- **Web**: ASP.NET Core Minimal APIs (controllers are not used), built-in OpenAPI support.
- **Persistence**: EF Core with SQL Server; migrations managed in the Infrastructure project.
- **Architecture**: Clean Architecture (Domain, Application, Infrastructure, API) per Principle I.
- **Testing**: xUnit for unit and integration tests; integration tests run against a real SQL
  Server instance.
- **Dependencies**: New third-party packages MUST be justified in the plan, actively maintained,
  and license-compatible. Prefer framework built-ins when they meet the need.
- Deviations from this stack require a constitution amendment.

## Development Workflow & Quality Gates

- Work follows the Spec Kit flow: specify → (clarify) → plan → tasks → implement. Each plan MUST
  include a Constitution Check confirming compliance with all eight principles; any deviation
  MUST be recorded in Complexity Tracking with justification and a simpler alternative considered.
- Before merge, every change MUST pass: build with zero warnings, all unit and integration tests,
  and formatting/analyzer checks.
- Every change MUST be reviewed by at least one other person (or explicitly self-reviewed against
  this constitution on solo work), with reviewers checking layer boundaries, authorization,
  transaction boundaries, error format, and logging hygiene.
- Changes to the public API contract, database schema, or authorization rules MUST be called out
  explicitly in the change description.

## Governance

- This constitution supersedes conflicting practices, conventions, and individual preferences for
  this project. Where a spec or plan conflicts with it, the constitution wins until amended.
- **Amendments**: proposed via a documented change to this file that states the motivation,
  the affected principles, and any migration needed for existing code. Amendments require approval
  by the project owner and take effect when merged.
- **Versioning**: semantic versioning applies to this document—MAJOR for removing or redefining a
  principle in a backward-incompatible way, MINOR for adding a principle or section or materially
  expanding guidance, PATCH for clarifications and wording fixes.
- **Compliance**: every plan's Constitution Check and every code review verify compliance.
  Unjustified complexity or violations block merge. Principles SHOULD be reviewed at least once per
  major release for continued relevance.

**Version**: 1.0.0 | **Ratified**: 2026-10-17 | **Last Amended**: 2026-10-17
