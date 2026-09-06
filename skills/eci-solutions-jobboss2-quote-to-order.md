---
name: eci-solutions-jobboss2-quote-to-order
description: >-
  Drive the quote-to-order flow in JobBOSS² — create a customer, quote a part with line items,
  convert to an order with line items, routings and releases, and track it on the shop floor.
api: eci-solutions:eci-solutions-jobboss2
base_url: https://api-jb2.integrations.ecimanufacturing.com
spec: openapi/eci-solutions-jobboss2-openapi.json
generated: '2026-09-06'
method: generated
source: openapi/eci-solutions-jobboss2-openapi.json
operations:
  - POST /api/v1/customers
  - GET /api/v1/customers/{customerCode}
  - POST /api/v1/shipping-addresses
  - POST /api/v1/quotes
  - POST /api/v1/quotes/{quoteNumber}/quote-line-items
  - PATCH /api/v1/quotes/{quoteNumber}/quote-line-items/{itemNumber}
  - POST /api/v1/orders
  - POST /api/v1/orders/{orderNumber}/order-line-items
  - POST /api/v1/orders/{orderNumber}/order-line-items/{itemNumber}/order-routings
  - POST /api/v1/orders/{orderNumber}/order-line-items/{itemNumber}/releases
  - GET /api/v1/shopview/get-jobs
---

# Quote to order in JobBOSS²

> JobBOSS² API access is available **only to hosted (Cloud) JobBOSS² customers**, and ECI must
> activate the API for the company before any call succeeds. The published spec declares **no
> operationIds**, so operations are addressed here by method and path — that is what the contract
> gives you.

## 1. Customer

```
POST /api/v1/customers
GET  /api/v1/customers/{customerCode}
POST /api/v1/shipping-addresses
GET  /api/v1/shipping-addresses/{customerCode}/{location}
```

## 2. Quote

```
POST  /api/v1/quotes
POST  /api/v1/quotes/{quoteNumber}/quote-line-items
PATCH /api/v1/quotes/{quoteNumber}/quote-line-items/{itemNumber}
GET   /api/v1/quotes/{quoteNumber}/quote-line-item/{itemNumber}
```

**Currency.** `Quote` and `QuoteLineItem` carry paired fields — `price1`/`price1Foreign`,
`miscCharge`/`miscChargeForeign`. The non-`Foreign` field is the *company's* currency; the
`Foreign` field is the *customer's*. On POST or PATCH set exactly **one** of the pair; JobBOSS²
computes the other from the exchange rate. Sending both is not supported.

## 3. Order

```
POST  /api/v1/orders
PATCH /api/v1/orders/{orderNumber}
POST  /api/v1/orders/{orderNumber}/order-line-items
PATCH /api/v1/orders/{orderNumber}/order-line-items/{itemNumber}
POST  /api/v1/orders/{orderNumber}/order-line-items/{itemNumber}/order-routings
POST  /api/v1/orders/{orderNumber}/order-line-items/{itemNumber}/releases
```

There is no DELETE anywhere in this contract and no idempotency key. A retried POST creates a
duplicate order, line or release. Read back with the GET on the same path before retrying.

## 4. Shop floor and material

```
GET  /api/v1/shopview/get-jobs
GET  /api/v1/shopview/kpi/jobs-past-due
POST /api/v1/issuing-materials/stock-to-job-transfer
POST /api/v1/time-tickets
POST /api/v1/time-ticket-details
```

## Query conventions

- Filter: `fieldName[operator]=value`, operators `eq ne gt gte lt lte in notin null`. Equality may
  omit the operator: `?customerCode=ACME`. `in`/`notin` take pipe-separated lists.
  **Do not use `[NULL]` on `revisedDate`** — that field can never be null and the call returns 500.
- Page: `?take=100&skip=200`. Default `take` is **200**.
- Sort: `?sort=+lastName,-createDate`.
- Fields: `?fields=fieldOne,fieldTwo` — the default response is a documented subset, not everything.
- Dates: JobBOSS² is **UTC** in and out (`yyyy-MM-ddTHH:mm:ssZ` or `yyyy-MM-dd`).

## Errors

```json
{ "Error": { "Title": "Not Found", "Status": 404, "Detail": "", "TraceId": "" } }
```

PascalCase, wrapped in `Error`, and unique to JobBOSS² — the other ECI APIs use lowercase
ProblemDetails. Quote the `TraceId` to ECI support (`mfg-integration-support@ecisolutions.com`).
