---
name: agile-crm-lead-to-deal
description: >-
  Turn an inbound lead into a tracked Agile CRM deal — find or create the contact, tag and score it,
  then open a deal on the right track and attach a follow-up task.
api: agile-crm:rest-api
generated: '2026-08-13'
method: generated
source: openapi/agile-crm-contacts-api-openapi.yml, openapi/agile-crm-deals-api-openapi.yml, openapi/agile-crm-tasks-api-openapi.yml, openapi/agile-crm-tracks-api-openapi.yml
operations:
  - getContactByEmail
  - createContact
  - addContactTags
  - addLeadScore
  - listTracks
  - createDeal
  - createTask
---

# Agile CRM: lead to deal

Base URL `https://{domain}.agilecrm.com/dev`. Send `Accept: application/json` on every request or you
get XML. Authenticate with HTTP Basic: username = account email, password = the REST client API key.

## 1. Look for the contact before creating one

`getContactByEmail` — `GET /api/contacts/search/email/{email}`

A **204 means the contact does not exist**, not that the call failed. Agile CRM uses 204 in place of
404 on reads. Only treat 401 and 400 as errors here.

## 2. Create the contact if it was not found

`createContact` — `POST /api/contacts`

- `first_name` is the only mandatory property.
- Properties go in a `properties` array of `{name, type, subtype, value}` objects. `type` is `SYSTEM`
  for built-in fields and `CUSTOM` for your own — **case-sensitive**.
- Valid subtypes for `email` are `work` and `personal`; for `phone`, `work|home|mobile|main|home fax|work fax|other`.
- Leave `type` off, or set it to `PERSON`. Setting it to `COMPANY` creates a company on this same
  endpoint.
- Success is **200, not 201**, and the body is the created contact — read its `id` from there.
- A **406 means the tenant is at its plan contact ceiling** (1,000 Free / 10,000 Starter /
  50,000 Regular). Do not retry; it will never clear on its own.

**There is no idempotency key.** If this call times out, do not blindly retry — re-run step 1 first
to see whether the contact was in fact created, or you will create a duplicate.

## 3. Tag and score the new contact

`addContactTags` — `PUT /api/contacts/edit/tags`
`addLeadScore` — `POST /api/contacts/add-score`

Tag names must start with a letter and may contain only underscores and spaces as special
characters. Tags are case-sensitive, so `Inbound` and `inbound` become two different tags.

## 4. Resolve the track before opening the deal

`listTracks` — `GET /api/milestone/pipelines`

A deal's `milestone` must be one of the milestones defined on the track you are going to reference,
and milestones differ per track. Read the track first; do not assume a default milestone name.

## 5. Create the deal

`createDeal` — `POST /api/opportunity`

Deals are called deals everywhere except the URL, which says `opportunity`.

Required: `name`, `expected_value`, `probability`, `milestone`, `pipeline_id` (the track id from
step 4) and `owner_id`. Relate the contact with `contact_ids` — an array of contact ids.

`owner_id` is the id of a domain user, and **Agile CRM exposes no endpoint that lists users**. Take
the owner id from the `owner` object embedded in a contact or deal you have already read.

Success is 200 with the created deal in the body.

## 6. Attach the follow-up task

`createTask` — `POST /api/tasks`

Required: `type`, `priority_type`, `due`, `subject`. All enums are case-sensitive:

- `type`: `CALL`, `EMAIL`, `FOLLOW_UP`, `MEETING`, `MILESTONE`, `SEND`, `TWEET`, `OTHER`
- `priority_type`: `HIGH`, `NORMAL`, `LOW`
- `status`: `YET_TO_START`, `IN_PROGRESS`, `COMPLETED`

`due` is **epoch seconds**, not ISO 8601. Relate the contact with `contacts`, an array of contact ids.

## Error handling for the whole flow

| Status | Meaning | What to do |
|---|---|---|
| 200 | Success, including on create | Read the object from the body |
| 204 | On a GET: no such record. On a DELETE: deleted | Branch on the verb |
| 400 | Input in the wrong format | No field-level detail is returned; re-check the enums and epoch dates |
| 401 | Wrong email or API key | Check the tenant subdomain and that you used the *first* key on the API screen |
| 406 | Contact ceiling exceeded | Plan quota — upgrade or delete contacts; retrying will not help |

No 429 and no rate-limit headers exist. Self-throttle.
