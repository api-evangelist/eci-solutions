---
name: eci-solutions-erp-incremental-sync
description: >-
  Pull an incremental delta out of an ECI ERP (Deacom, M1, Macola or JobBOSS²) through the
  product-agnostic ECI Manufacturing ERP API using the rowVersion change cursor. This is the only
  incremental mechanism ECI publishes on the ERP side — there are no ERP webhooks.
api: eci-solutions:eci-solutions-platform
base_url: https://api-erp.integrations.ecimanufacturing.com
spec: openapi/eci-solutions-erp-v2-1-openapi.json
generated: '2026-09-06'
method: generated
source: openapi/eci-solutions-erp-v2-1-openapi.json + conventions/eci-solutions-conventions.yml
operations:
  - GET /api/v2.1/user-info
  - GET__api_v2.1_view_jobs
  - GET__api_v2.1_view_sales-orders
  - GET__api_v2.1_view_sales-order-lines
  - GET__api_v2.1_view_parts
  - GET__api_v2.1_view_organizations
  - GET__api_v2.1_view_timecard-lines
---

# Incremental sync from an ECI ERP

## Before anything

1. Get a token. `POST https://api-user.integrations.ecimanufacturing.com/oauth2/api-user/token`
   with the client-credentials grant and scope `openid`. The token is valid for exactly 3600
   seconds and **there is no refresh token** — cache it and re-present the client id and secret
   when it expires.
2. Learn what you are connected to. `GET /api/v2.1/user-info` returns `userId`, `companyId`,
   `companyCustomerId` and `productId`. Do this first every run: the ERP behind this token may be
   Deacom, M1, Macola or JobBOSS², resource support varies by product, and **`companyId` does not
   uniquely identify the ERP instance**.
3. Expect 403 if the API has not been activated for the company. ECI activates APIs manually.

## The cursor

Every `view` resource carries `rowVersion`, an int64 that only ever increases: any new or updated
record gets a value higher than every other record in that source. That is the delta mechanism.

```
GET /api/v2.1/view/jobs?rowVersion[gt]=<last seen>&sort=rowVersion&take=200&count=2
```

- `rowVersion[gt]=` is the filter grammar (`field[operator]=value`). Operators available:
  `eq ne gt gte lt lte in notin`.
- `sort=rowVersion` keeps the page order stable so the highest value on the page is a safe new
  watermark. `-rowVersion` reverses it.
- `take` / `skip` page. Default `take` is 200.
- `count=2` returns both the page and the total matching count; `count=1` returns the count alone
  with an empty `data`; `count=0` (the default) returns data only.

Persist the maximum `rowVersion` you saw, not the count of records. Resume from it next run.

## Fields and joins

Most GETs deliberately return a **subset** of fields. Ask for what you need:

```
GET /api/v2.1/view/sales-order-lines?fields=salesOrderID,partID,quantity,rowVersion
GET /api/v2.1/view/timecard-lines?expand=timecard(expand=employee)
```

`expand=` is nestable and takes an inner `fields=` expression. Use it instead of N+1 lookups —
see `data-model/eci-solutions-data-model.yml` for which relationships exist.

## Time zones — read this before you compare dates

The ERP API returns dates in the source ERP's own zone, which varies by product and sometimes by
data source. Pass `ConvertToUTC=yes` on resources that support it. Note that the two sibling APIs
disagree by default: **JobBOSS² is UTC, M1 is local time.**

## Identity

`uniqueID` is unique per resource per company account — **not globally unique**, and its underlying
type differs by ERP (Macola GUID, M1 GUID, JobBOSS² Int32). Key your local store on
(companyId, resource, uniqueID), never on uniqueID alone.

## Failure handling

- 401 — token expired or missing. Re-mint; there is no refresh token.
- 403 — API not activated for this company. This is a commercial action, not a retry.
- 404 — the id does not exist in this ERP instance.
- 429 — only the Financial v2 API declares it, but the Authentication API is documented at 100
  requests/minute and the Management API at 500/minute per source IP. **No `Retry-After` or
  `RateLimit-*` header is published anywhere in the ECI estate**, so back off on your own schedule.
- Error bodies are RFC 7807-*shaped* (`type/title/status/detail`) but are served as
  `application/json`, never `application/problem+json`. Do not content-negotiate for problems.
