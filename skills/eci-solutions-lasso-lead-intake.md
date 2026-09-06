---
name: eci-solutions-lasso-lead-intake
description: >-
  Capture and enrich a new-home sales lead in Lasso CRM — create a registrant, attach contact
  details, notes and questions, assign a sales rep, book an appointment, and wire the registrant to
  an external system so webhook events flow.
api: eci-solutions:eci-solutions-lasso-crm
base_url: https://api.lassocrm.com/v1
spec: openapi/eci-solutions-lasso-crm-openapi.yml
generated: '2026-09-06'
method: generated
source: openapi/eci-solutions-lasso-crm-openapi.yml + asyncapi/eci-solutions-lasso-registrant-webhooks.yml
operations:
  - post_registrants
  - get_registrants_search
  - put_registrants_by_id
  - post_registrants_emails
  - post_registrants_phones
  - post_registrants_addresses
  - post_registrants_notes
  - post_registrants_questions
  - put_registrants_assigned_sales_reps
  - put_registrants_rating
  - post_registrants_appointments
  - post_registrants_appointments_email_confirmation
  - post_registrants_integrations
  - get_inventory_search
---

# Lead intake in Lasso CRM

## Auth

Bearer JWT API key on every request. Keys are **per project or per location** and come from a Lasso
business contact — there is no self-serve issuance. Rate limit: **1,000 requests per minute per
key**, stated in the API description. No `RateLimit-*` or `Retry-After` header is published, so
enforce your own budget.

```
Authorization: Bearer <api key>
```

## 1. Do not create a duplicate

There is **no `DELETE /registrants/{registrantId}`**. A registrant you create cannot be removed
through the API — only its sub-records can. Search first:

```
get_registrants_search   GET /registrants/search      # supports email, phone, nickname, fuzzy, smartSearch
```

## 2. Create

```
post_registrants         POST /registrants
```

`registrationDate` is settable and defaults to the current UTC datetime. Notes with no `createdBy`
are attributed to "Lasso Autobot". Field lengths are enforced — over-long email, phone, address or
sales-detail values return **400**, and a non-numeric value in a numeric field returns **406**.

## 3. Enrich

```
post_registrants_emails             POST   /registrants/{registrantId}/emails
post_registrants_phones             POST   /registrants/{registrantId}/phones
post_registrants_addresses          POST   /registrants/{registrantId}/addresses
post_registrants_notes              POST   /registrants/{registrantId}/notes
post_registrants_questions          POST   /registrants/{registrantId}/questions
put_registrants_rating              PUT    /registrants/{registrantId}/rating
put_registrants_source_type         PUT    /registrants/{registrantId}/source-type
put_registrants_follow_up_process   PUT    /registrants/{registrantId}/follow-up-process
put_registrants_assigned_sales_reps PUT    /registrants/{registrantId}/assigned-sales-reps
```

`PUT /registrants/{registrantId}/person/contact-information` replaces the whole contact block at
once; the list must name a primary Address/Email/Phone or you get a 400. Note a defect in the
published spec: the operationIds on this path pair are **swapped** — the GET carries
`put_registrants_contact_information` and the PUT carries `get_registrants_contact_information`.
Bind on method + path here, not on operationId.

These sub-records **are** individually deletable — `delete_registrants_emails_by_id`,
`delete_registrants_phones_by_id`, `delete_registrants_addresses_by_id`,
`delete_registrants_notes_by_id`, `delete_registrants_relationships_by_id` — which is the only undo
path in this API. No window is documented for any of them.

## 4. Appointments

```
post_registrants_appointments                  POST /registrants/{registrantId}/appointments
post_registrants_appointments_email_confirmation
                                               POST /registrants/{registrantId}/appointments/{appointmentId}/email-confirmations
```

Sending the confirmation emails a real person. Treat it as irreversible.

## 5. Wire the webhook

```
post_registrants_integrations   POST /registrants/{registrantId}/integrations   # body: { externalId }
```

The integration type is inferred from the API key. The endpoint URL and any static `customParams`
are configured on the Lasso **project-level webhook configuration page**, not through the API.

After that, updates push a `RegistrantWebhookEvent`: `timestamp`, `id`, `externalId`,
`action` (`inserted`|`updated`|`deleted`), `customParams`, `changed[]`, `entity`. On `deleted`,
`entity` is null — use `id`/`externalId`. On `updated`, `changed[]` names only the fields that
triggered it, drawn from a fixed list (`person, rating, excludeFromTraffic, registrationDate,
followUpProcess, sourceType, secondarySourceType, questions, notes, assignedSalesReps, history`).
**Internal metadata changes fire webhooks but never appear in `changed[]`**, so expect update events
with an empty `changed` array and skip them rather than treating them as corrupt. No payload
signing, retry policy or replay endpoint is documented.

## 6. Inventory

```
get_inventory_search   GET /inventory/search
get_inventory_by_id    GET /inventory/{inventoryId}
put_inventory_pricing  PUT /inventory/{inventoryId}/pricing
post_inventory_pricing_revisions
                       POST /inventory/{inventoryId}/pricing-revisions
```

Pricing revisions live on their own routes — they were moved off the `Pricing` model in v1.0.31,
and `planPrice` was removed from `Inventory.Pricing` in the same release.
