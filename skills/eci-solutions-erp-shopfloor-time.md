---
name: eci-solutions-erp-shopfloor-time
description: >-
  Record shop-floor labour and production against a job in an ECI ERP — clock an employee in and
  out, write timecard lines, and report part production. Every one of these is a non-reversible
  write.
api: eci-solutions:eci-solutions-platform
base_url: https://api-erp.integrations.ecimanufacturing.com
spec: openapi/eci-solutions-erp-v2-1-openapi.json
generated: '2026-09-06'
method: generated
source: openapi/eci-solutions-erp-v2-1-openapi.json + conventions/eci-solutions-conventions.yml
operations:
  - GET /api/v2.1/employees
  - GET /api/v2.1/jobs
  - GET /api/v2.1/job-operations
  - POST /api/v2.1/timecards
  - POST /api/v2.1/timecards/clock-in
  - POST /api/v2.1/timecards/clock-out
  - POST /api/v2.1/timecard-lines
  - PATCH /api/v2.1/timecard-lines/{id}
  - POST /api/v2.1/jobs/{uniqueID}/part-production
  - PATCH /api/v2.1/job-operations/{id}
---

# Shop-floor time and production in the ECI Manufacturing ERP API

> **Read this first.** None of the writes below can be undone through the API. The ERP API has
> **no DELETE** on jobs, job operations, timecards or timecard lines, and there is **no
> Idempotency-Key header anywhere in ECI's estate**. A retried POST creates a second record. Dedupe
> on your side before you send.

## 1. Resolve the work

```
GET /api/v2.1/jobs?jobID=<code>
GET /api/v2.1/job-operations?jobID=<id>
GET /api/v2.1/employees?employeeID=<code>
```

Take the `uniqueID` from each — that is the handle the write operations want. Remember it is unique
only within this company account.

## 2. Clock in and out

```
POST /api/v2.1/timecards/clock-in
POST /api/v2.1/timecards/clock-out
```

Clock-out is the closing half of the pair, not an undo: it completes the timecard, it does not
remove it. If you clocked in the wrong employee, there is no API call that reverses it — that is a
correction inside the ERP.

## 3. Write time explicitly

```
POST /api/v2.1/timecards            # create the card (employeeID, shiftID, lines)
POST /api/v2.1/timecard-lines       # jobID, jobOperationID, workCenterID, processID
PATCH /api/v2.1/timecard-lines/{id} # correct a line in place — the only safe fix path
```

`PATCH` is the mechanism ECI gives you for corrections. Prefer it over posting a compensating line.

## 4. Report production

```
POST /api/v2.1/jobs/{uniqueID}/part-production
```

Body references `partID`, `partBinID`, `shiftID`, `printDestinationID`. This moves inventory. There
is no dry-run, preview or validate-only mode on any ECI write, so there is no way to rehearse it.

## 5. Advance the operation

```
PATCH /api/v2.1/job-operations/{id}
```

## Conventions that apply to all of the above

- Bearer token from `https://api-user.integrations.ecimanufacturing.com/oauth2/api-user/token`,
  3600s, no refresh token.
- Call `GET /api/v2.1/user-info` first to confirm which ERP product and company the token binds to.
  Resource support varies by ERP — check the *Supported ERPs* note on each resource.
- Dates: pass `ConvertToUTC=yes` where supported; do not assume UTC.
- Errors are RFC 7807-shaped under `application/json`.
