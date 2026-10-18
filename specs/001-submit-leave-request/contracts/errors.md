# Error Catalogue: Submit Leave Request

All errors are RFC 9457 Problem Details (`application/problem+json`) with `traceId`. Codes are
stable: clients may rely on them; changing one is a breaking contract change (Constitution VI).
See [openapi.yaml](./openapi.yaml) for schemas.

## Status-code policy (T2 — approved)

| Status | Meaning | Client action |
|---|---|---|
| `400` | Input is missing or malformed; all field errors listed together | Fix the fields and resubmit |
| `401` | Not authenticated | Obtain a valid token |
| `403` | Authenticated, but not an active employee | Contact HR/support |
| `404` | Resource not found or not yours (read endpoint) | — |
| `409` | Genuine state or concurrency conflict; nothing saved | Retry (safe) |
| `422` | Well-formed request that breaks a documented business rule | Change the request (dates, type) or act on the rule |
| `500` | Unexpected error; no details exposed | Retry later; quote `traceId` to support |
| `503` | Could not complete in time; nothing saved | Retry later |

Problem Details fields: `type`, `title`, `status`, `detail`, `traceId`; plus `errors` (400) or
`errorCode` and rule-specific extensions (403, 409, 422, 503).

## 400 — Input validation (all field errors reported together)

Returned as `errors: { field: [messages] }`. No `errorCode` per field is required; the field name
and message identify the problem. Messages never echo the reason text.

| Field | Condition | Spec |
|---|---|---|
| `leaveTypeCode` | Missing / blank | FR-002 |
| `startDate`, `endDate` | Missing / blank | FR-002 |
| `startDate`, `endDate` | Not a valid `yyyy-MM-dd` calendar date | US3 #5 |
| `endDate` | Earlier than `startDate` | FR-003, US3 #2 |
| `reason` | Missing / whitespace only | FR-002 |
| `reason` | Longer than 500 characters | A5 |
| *(body)* | Body missing or not valid JSON | — |

## 401 — Not authenticated

Problem Details body (`title` "Unauthorized", `status` 401, `traceId`; no `errorCode`) plus the
`WWW-Authenticate: Bearer` header. Spec: FR-007, US4 #1.

## 403 — No active employee record

| errorCode | Meaning | Spec |
|---|---|---|
| `EMPLOYEE_NOT_FOUND` | The authenticated identity has no active employee record. | US4 #4 |

## 422 — Business rule violations (one reported, in evaluation order)

| errorCode | Meaning | Extensions | Spec |
|---|---|---|---|
| `LEAVE_TYPE_INVALID` | The leave type does not exist or is not currently available. Checked first. | — | FR-004, US3 #3 |
| `LEAVE_START_IN_PAST` | Start date is before today (organisation time zone). | — | A4 (unconfirmed) |
| `LEAVE_SPANS_LEAVE_YEARS` | Start and end are in different calendar years. Submit one request per year. | — | FR-020 |
| `LEAVE_HOLIDAY_CALENDAR_NOT_AVAILABLE` | The public-holiday calendar for that year is not confirmed complete, so leave days cannot be counted reliably. **Pending business approval (B1); not implemented until approved.** | `leaveYear` | B1 |
| `LEAVE_ZERO_WORKING_DAYS` | The period contains only weekends and/or public holidays. | — | FR-010a |
| `LEAVE_OVERLAP` | Shares at least one calendar date with your Pending or Approved request. On a retry after a lost response, GET `overlappingRequestId` and compare details to tell whether it is your original submission. | `overlappingRequestId` | FR-014, FR-018a |
| `LEAVE_BALANCE_NOT_AVAILABLE` | No balance exists yet for this leave type and year. | `leaveYear` | FR-020a |
| `LEAVE_INSUFFICIENT_BALANCE` | Counted days exceed available days. | `requestedDays`, `availableDays` | FR-011 |

## 404 — Not found (read endpoint only)

The request does not exist or is not owned by the caller.

## 409 — Concurrency conflict

| errorCode | Meaning |
|---|---|
| `CONCURRENCY_CONFLICT` | Data changed while the submission was processed (backstop: a writer bypassed the employee lock protocol). Retrying is safe: nothing was saved. |

## 503 — Temporarily unable to process

| errorCode | Meaning |
|---|---|
| `SUBMISSION_TIMEOUT` | Could not obtain the employee lock within 5 s or complete the database work in time. Nothing was saved. |

## 500 — Unexpected error

Generic title and `traceId` only; no exception details.
