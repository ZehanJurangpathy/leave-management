# Feature Specification: Submit Leave Request

**Feature Branch**: `001-submit-leave-request` (no git repository yet; branch not created)

**Created**: 2026-10-17

**Status**: Draft

**Input**: User description: "Feature: Submit Leave Request — Employees need a reliable way to submit
leave requests through the Leave Management API. The system must validate requests and preserve
accurate leave balances. As an employee, I want to submit a leave request so that my manager can
review it."

## Clarifications

### Session 2026-10-17

- Q: How are leave days counted? → A: Working days (Monday–Friday) only; public holidays are
  excluded.
- Q: When is balance reserved or deducted? → A: Reserved on submission; converted to used on
  approval; released on rejection or cancellation.
- Q: Are overlapping requests allowed? → A: No. A request overlapping the same employee's Pending
  or Approved request is rejected; Rejected and Cancelled requests are ignored.

### Session 2026-10-18

- Q: Is overlap judged on all calendar dates in the range or only on counted leave days? → A: All
  calendar dates, including weekends and public holidays; any shared date blocks the new request.
- Q: How is the leave year defined? → A: Calendar year (1 January – 31 December) for all employees.
- Q: What happens when an employee with no reporting manager submits? → A: Accepted as Pending with
  days reserved, and routed to a fallback reviewer (HR).
- Q: Can an employee request dates in a future leave year? → A: Yes, only if their balance for
  that leave year already exists; otherwise rejected as "balance not yet available".
- Q: What peak number of simultaneous submissions must meet the 2-second target? → A: Up to 50
  simultaneous submissions; the 2 seconds is measured end-to-end within the system, including all
  record-keeping, balance checks, and any waiting caused by simultaneous requests.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Submit a valid leave request (Priority: P1)

An authenticated employee submits a leave request with a leave type, start date, end date, and
reason. The system records the request against that employee, places it in Pending status so their
manager can review it, and returns a confirmation with the full details of the created request.

**Why this priority**: This is the core value of the feature. Without it, no other leave workflow
(approval, rejection, balance tracking) can begin.

**Independent Test**: Can be fully tested by an authenticated employee with sufficient balance
submitting a well-formed request and verifying the response confirms creation, shows Pending status,
and that the request can afterwards be found as belonging to that employee.

**Acceptance Scenarios**:

1. **Given** an authenticated employee with sufficient available balance for the chosen leave type,
   **When** they submit a request with a valid leave type, a start date, an end date on or after the
   start date, and a reason, **Then** the request is created in Pending status and the response
   confirms success.
2. **Given** a successful submission, **When** the employee reads the confirmation, **Then** it
   includes a unique request identifier, the owning employee, leave type, start date, end date,
   number of leave days counted, reason, Pending status, and submission timestamp.
3. **Given** a request whose start date and end date are the same working day that is not a public
   holiday, **When** it is submitted, **Then** it is accepted as a 1-day request.
4. **Given** a request from Friday to the following Tuesday where Monday is a public holiday,
   **When** it is submitted, **Then** it counts 2 leave days (Friday and Tuesday).
5. **Given** a successful submission, **When** the employee's manager later looks for requests to
   review, **Then** the new request is attributable to the submitting employee (review itself is out
   of scope for this feature).
6. **Given** an authenticated employee with no reporting manager and sufficient balance, **When**
   they submit a valid request, **Then** it is created in Pending status with its days reserved,
   and it is designated for review by the fallback reviewer (HR) instead of a manager.

---

### User Story 2 - Enforce available leave balance (Priority: P2)

When an employee submits a request, the system checks the number of leave days requested against
the employee's available balance for that leave type and refuses requests that would exceed it, so
balances stay accurate and employees cannot book more leave than they are entitled to.

**Why this priority**: Accurate balances are the stated business goal. Accepting requests that
cannot be honoured creates downstream disputes and manual correction.

**Independent Test**: Can be tested by giving an employee a known balance and submitting requests
below, equal to, and above it, then verifying acceptance or rejection and the resulting balance.

**Acceptance Scenarios**:

1. **Given** an employee with 5 available days of a leave type, **When** they submit a request
   counting 5 days of that type, **Then** the request is accepted.
2. **Given** an employee with 5 available days of a leave type, **When** they submit a request
   counting 6 days of that type, **Then** the request is rejected with a message stating the request
   exceeds the available balance, including the days requested and the days available, and no
   request is created.
3. **Given** an employee with balance in one leave type but none in another, **When** they request
   the type with no balance, **Then** the request is rejected; balances of different leave types are
   never combined.
4. **Given** an employee with 5 available days who submits two requests of 3 days each at the same
   moment, **When** both are processed, **Then** at most one is accepted and the other is rejected
   for insufficient balance; the employee is never committed to more days than available.
5. **Given** an employee with 10 available days, **When** they successfully submit a request
   counting 3 days, **Then** 3 days are reserved and their available balance becomes 7.
6. **Given** an employee with 10 available days and an existing Pending request reserving 6 days,
   **When** they submit another request counting 5 days, **Then** it is rejected for insufficient
   balance (4 days available).
7. **Given** it is December and the employee's balance for next year has not been set up, **When**
   they submit a request for dates in January of next year, **Then** it is rejected with a
   "balance not yet available" reason and nothing is reserved.
8. **Given** the employee's balance for next year already exists with 10 days, **When** they
   submit a 3-day request in January of next year, **Then** it is accepted and 3 days are reserved
   against next year's balance; this year's balance is unchanged.

---

### User Story 3 - Receive useful feedback for invalid requests (Priority: P3)

When an employee submits an incomplete or invalid request, the system rejects it without creating
anything and explains every problem clearly enough that the employee can correct and resubmit.

**Why this priority**: Clear feedback reduces support effort and failed attempts, but the feature
still delivers value with basic rejection behaviour.

**Independent Test**: Can be tested by submitting requests with each kind of invalid input and
verifying that each is rejected, no request is created, and the feedback names the offending fields.

**Acceptance Scenarios**:

1. **Given** a request missing one or more of leave type, start date, end date, or reason, **When**
   it is submitted, **Then** it is rejected and the feedback identifies each missing field.
2. **Given** a request whose end date is earlier than its start date, **When** it is submitted,
   **Then** it is rejected with feedback stating that the end date must not be earlier than the
   start date.
3. **Given** a request with an unrecognised or inactive leave type, **When** it is submitted,
   **Then** it is rejected with feedback that the leave type is not valid.
4. **Given** a request with multiple problems, **When** it is submitted, **Then** all input
   problems are reported together in a single response rather than one at a time.
5. **Given** a request with dates that are not valid calendar dates or are in an unrecognised
   format, **When** it is submitted, **Then** it is rejected with feedback identifying the field.
6. **Given** a request that is otherwise valid but falls entirely on weekends and/or public
   holidays (zero leave days), **When** it is submitted, **Then** it is rejected with feedback
   explaining that the period contains no leave days.
7. **Given** an employee with a Pending or Approved request covering 10–12 March, **When** they
   submit a request for 12–14 March, **Then** it is rejected with feedback identifying the
   overlapping existing request.
8. **Given** an employee whose only request covering 10–12 March was Rejected or Cancelled, **When**
   they submit a new request for 10–12 March, **Then** the overlap rule does not block it.

---

### User Story 4 - Protect employee identity and server-owned data (Priority: P4)

The system determines who is submitting from their authenticated identity only. Employees cannot
submit requests on behalf of someone else, set their own status, or influence their balance by
supplying values in the request.

**Why this priority**: This is a mandatory security guarantee (Constitution Principle IV). It is
listed separately so it is verified explicitly, although it is enforced from the first release.

**Independent Test**: Can be tested by sending requests without authentication, and authenticated
requests that include another employee's identity, a status, or a balance value, and verifying the
outcome is driven only by the authenticated identity and server-held data.

**Acceptance Scenarios**:

1. **Given** a caller who is not authenticated, **When** they attempt to submit a request, **Then**
   the attempt is refused as unauthenticated and nothing is created.
2. **Given** an authenticated employee who includes a different employee's identifier in the
   request, **When** it is submitted, **Then** the identifier is not used: the request is either
   rejected or created for the authenticated employee only, never for the other employee.
3. **Given** an authenticated employee who includes a status (e.g., "Approved") or a balance value
   in the request, **When** it is submitted, **Then** those values are ignored or rejected, the
   request starts in Pending status, and the balance check uses only the server-held balance.
4. **Given** an authenticated caller with no corresponding active employee record, **When** they
   attempt to submit a request, **Then** the attempt is refused and nothing is created.

---

### Edge Cases

- Start date equals end date (single-day request): accepted if it counts at least one leave day.
- Requested period consists entirely of weekends and/or public holidays: counts zero days and is
  rejected (US3 scenario 6).
- Public holiday falling on a weekend: the day is simply not counted (no substitute day is added).
- Requested period starts or ends on a weekend or holiday: only the working days inside the period
  are counted; the dates themselves are accepted as submitted.
- Request equals exactly the available balance: accepted (US2 scenario 1).
- Employee has zero balance for the requested type: rejected for insufficient balance.
- Employee has no balance record at all for the requested leave type and leave year (current or
  future): rejected as "balance not yet available" (FR-020a).
- Two or more simultaneous submissions by the same employee drawing on the same balance: never
  collectively exceed the balance (US2 scenario 4, FR-013).
- Accidental duplicate submission or a retry after a lost confirmation (e.g., network timeout
  after the request was accepted): no second request or reservation is created (FR-014, FR-018a).
- Submission fails part-way (e.g., system error after validation): no request and no reservation
  remain (FR-015, FR-018).
- Request overlapping an existing Pending or Approved request of the same employee, of any leave
  type: rejected (FR-014). Overlap with Rejected or Cancelled requests is allowed.
- Overlap only on non-counted dates (e.g., existing request Fri–Sun, new request Sat–Mon): still
  rejected, because overlap is judged on calendar dates (FR-014). The employee can resubmit as
  Mon–Mon.
- Two simultaneous submissions with overlapping dates, even of different leave types: at most one
  is accepted (FR-013, FR-014).
- Start date in the past: rejected (assumption A4).
- Request spanning two leave years (e.g., 30 December – 2 January): rejected with feedback asking
  the employee to submit one request per leave year (FR-020). The rejection applies even if the
  days in one of the years count zero leave days.
- Reason is extremely long or only whitespace: whitespace-only is treated as missing; length is
  limited (assumption A5).
- Employee has no manager assigned: request is created in Pending, days are reserved, and it is
  designated for the fallback reviewer (HR) (FR-021).
- Leave type exists but is not applicable to the employee (e.g., inactive): rejected as invalid.

## Requirements *(mandatory)*

### Functional Requirements

**Submission**

- **FR-001**: Authenticated employees MUST be able to submit a leave request containing a leave
  type, start date, end date, and reason.
- **FR-002**: The system MUST require all four fields; a field that is absent, empty, or contains
  only whitespace MUST be treated as missing.
- **FR-003**: The system MUST reject a request whose end date is earlier than its start date.
- **FR-004**: The system MUST reject a request whose leave type is not one of the leave types
  currently available to employees.
- **FR-005**: A successfully submitted request MUST be created in Pending status. Clients MUST NOT
  be able to set any other initial status.
- **FR-006**: On success, the system MUST return a confirmation containing the created request's
  unique identifier, owning employee, leave type, start date, end date, counted leave days, reason,
  status, and submission timestamp.

**Identity and authorization**

- **FR-007**: The system MUST refuse submissions from unauthenticated callers.
- **FR-008**: The system MUST associate every request with the employee derived from the
  authenticated identity. Any employee identifier supplied by the client MUST NOT determine
  ownership.
- **FR-009**: Server-owned values (status, counted days, available balance, owning employee,
  submission timestamp) supplied by the client MUST be ignored or rejected and MUST NOT influence
  the outcome.

**Leave day counting and balance**

- **FR-010**: The system MUST compute the number of leave days for a request on the server as the
  count of working days (Monday–Friday) from the start date to the end date inclusive, excluding
  dates on the organisation's public holiday calendar. Client-supplied day counts MUST NOT be used.
- **FR-010a**: The system MUST reject a request whose counted leave days equal zero.
- **FR-011**: The system MUST reject a request whose counted leave days exceed the employee's
  available balance for that leave type, and MUST tell the employee the days requested and the days
  available.
- **FR-012**: A successful submission MUST reserve the request's counted leave days against the
  employee's balance for that leave type, reducing the available balance immediately. Available
  balance MUST equal entitlement minus days used minus days reserved by Pending requests.
  Converting a reservation to used days on approval, and releasing it on rejection or
  cancellation, belong to the approval and cancellation features.
- **FR-012a**: Each request's reservation MUST change state at most once: it is either converted
  to used days (on approval) or released (on rejection or cancellation), never both and never
  twice, even if the same action is repeated or sent simultaneously. This feature creates the
  reservation; the approval and cancellation features MUST honour this rule.
- **FR-013**: When several submissions by the same employee are processed at the same time (of the
  same or different leave types), the outcome MUST be the same as if they had been processed one
  after another: the total days committed never exceed the available balance for any leave type
  and leave year, and no two accepted requests overlap (FR-014). Any submission that would break
  this MUST be rejected with feedback that the employee can act on (e.g., retry or adjust the
  request), and no partial request may be created.
- **FR-014**: The system MUST reject a request whose date range shares at least one calendar date
  (including weekends and public holidays, whether or not the date counts as a leave day) with an
  existing Pending or Approved request of the same employee, regardless of leave type. Rejected
  and Cancelled requests MUST NOT block a new request. The feedback MUST identify the overlapping
  request.
- **FR-020**: The leave year is the calendar year (1 January – 31 December) for all employees. A
  request's start and end dates MUST fall in the same leave year; otherwise it MUST be rejected
  with a business-rule reason. The balance checked and reserved is the employee's balance for that
  leave type and leave year.
- **FR-020a**: Requests for a future leave year MUST be accepted only if the employee's balance for
  that leave type and leave year already exists. If it does not exist, the request MUST be
  rejected with a distinct "balance not yet available" business-rule reason (not "insufficient
  balance").

**Review routing**

- **FR-021**: A request MUST be designated for review by the submitting employee's reporting
  manager. If the employee has no reporting manager at submission time, the request MUST still be
  accepted (Pending, days reserved) and MUST be designated for review by the fallback reviewer
  (HR). Missing a manager MUST NOT block submission.

**Feedback and integrity**

- **FR-015**: A rejected submission MUST NOT create a request or change any balance.
- **FR-016**: Rejection feedback MUST identify each invalid field and the reason, and MUST report
  all input-validation problems in one response.
- **FR-017**: Feedback for business-rule rejections (invalid or inactive leave type, insufficient
  balance, balance not yet available, overlap, zero-day period, spanning leave years) MUST be
  distinguishable from input-format errors and carry a stable reason a client can recognise.
- **FR-018**: Creating the request and reserving its days MUST succeed or fail together.
- **FR-018a**: A submission MUST NOT reserve days more than once. A submission that fails at any
  point leaves no reservation; a repeated or retried submission of a request that was already
  accepted MUST NOT create a second request or a second reservation. Whether a retry is reported
  as an overlap rejection or as the original confirmation is a planning decision, provided the
  employee can tell that the original request exists.
- **FR-019**: Each successful and rejected submission MUST be recorded in operational logs with the
  request and employee identifiers and outcome, without recording the reason text or other
  personal details.

### Key Entities

- **Employee**: A person who can request leave. Identified through their authenticated identity.
  Has at most one reporting manager (who reviews their requests) and leave balances per leave type
  and leave year.
- **Fallback Reviewer (HR)**: The reviewer designated for requests from employees who have no
  reporting manager.
- **Leave Type**: A category of leave (e.g., annual, sick) that employees may request. Each type
  has its own balance; types can be active or inactive.
- **Leave Balance**: For one employee, leave type, and leave year (calendar year): entitlement,
  days used (approved), and days reserved (pending). Available = entitlement − used − reserved.
  Owned and maintained by the server only.
- **Public Holiday**: A date on the organisation's holiday calendar that is not counted as a leave
  day.
- **Leave Request**: A request by one employee for one leave type over a date range, with a reason,
  a counted number of leave days, a status (starting at Pending), and a submission timestamp.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An employee can submit a valid leave request and receive confirmation in a single
  attempt. With up to 50 submissions in progress at the same time, 95% of submissions (accepted or
  rejected) receive their outcome within 2 seconds, measured from when the system receives the
  submission until it returns the outcome, including all record-keeping, balance checks, and any
  waiting caused by simultaneous requests. Time spent on the employee's own network or device is
  excluded.
- **SC-002**: 100% of accepted requests are attributed to the authenticated submitting employee;
  zero requests can be created for another employee in security testing.
- **SC-003**: Zero accepted requests leave an employee's reserved plus used days above their
  entitlement, including under simultaneous submissions of the same employee.
- **SC-004**: 100% of rejected submissions identify every input problem in a single response, so
  an employee needs at most one correction cycle for input errors.
- **SC-005**: 100% of successfully submitted requests start in Pending status.
- **SC-006**: Leave-day counts and balance changes for submitted requests require zero manual
  corrections in the first month of use due to system calculation errors.

## Assumptions

Assumptions below are reasonable defaults chosen to keep the specification testable. Items marked
**(review)** are business rules the owner should confirm; they can be changed during
`/speckit-clarify` without affecting the rest of the spec.

- **A1**: Employees authenticate through the organisation's existing identity mechanism, and every
  authenticated identity maps to at most one employee record. Choosing that mechanism is a planning
  decision.
- **A2**: Leave is requested in whole days only; half-day or hourly leave is out of scope.
  **(review)**
- **A3**: Leave types and employee balances already exist and are maintained elsewhere (e.g., by HR
  or a later feature). Accrual, carry-over, and balance administration are out of scope. Every leave
  type in scope has a balance; unlimited or untracked types (e.g., unpaid leave) are out of scope.
  **(review)**
- **A4**: Requests whose start date is before the current date (in the organisation's time zone)
  are rejected; retroactive leave recording is out of scope. **(review)**
- **A5**: The reason is free text limited to 500 characters. **(review)**
- **A6**: Splitting a request that spans two leave years into one request per year automatically
  is out of scope; the employee submits each part separately (see FR-020).
- **A7**: There is no minimum notice period or maximum request length beyond the balance limit.
  **(review)**
- **A8**: Who exactly acts as the fallback reviewer (an HR role or group), and what happens to an
  already-submitted request if the employee's manager changes later, are defined by the approval
  feature. This feature only records the designated reviewer at submission (FR-021).
- **A9**: Notifying the manager, approving, rejecting, cancelling, and editing requests are out of
  scope for this feature.
- **A10**: All dates are calendar dates without time of day, interpreted in a single organisation
  time zone.
- **A11**: A single organisation-wide public holiday calendar applies to all employees and is
  maintained outside this feature (e.g., by HR). Regional calendars are out of scope. **(review)**
- **A12**: Saturday and Sunday are non-working days for all employees; individual work patterns
  (part-time, shift work) are out of scope. **(review)**
