---
name: agile-crm-campaign-enrollment
description: >-
  Enrol Agile CRM contacts into a marketing campaign and remove them again — including the two
  places the naming differs from the docs and what the webhook surface will and will not tell you.
api: agile-crm:rest-api
generated: '2026-08-13'
method: generated
source: openapi/agile-crm-campaigns-api-openapi.yml, openapi/agile-crm-contacts-api-openapi.yml, asyncapi/agile-crm-webhooks.yml
operations:
  - listCampaigns
  - enrollContactInCampaign
  - removeCampaignSubscriber
  - getContactByEmail
  - addTagsByEmail
---

# Agile CRM: campaign enrolment

Base URL `https://{domain}.agilecrm.com/dev`. HTTP Basic auth, `Accept: application/json`.

**Campaigns are addressed as `workflows` in every URL.** The docs say campaign, the path says
workflow. There is no `/api/campaigns` collection.

## 1. List the campaigns

`listCampaigns` — `GET /api/workflows?page_size=20&cursor=<opaque>`

Same cursor paging as everywhere else in this API: the count rides on the first array element, the
next cursor on the last, and no cursor on the last element means you are done.

Each campaign carries a `rules` field which is **the entire visual workflow graph serialised as a
JSON string inside a JSON field**. Parse it only if you actually need the graph; it is large and its
internal shape is not documented.

## 2. Confirm the contact exists

`getContactByEmail` — `GET /api/contacts/search/email/{email}`

A 204 means there is no such contact. Enrolment is by email address, so enrolling an address that
has no contact record will not do what you want.

## 3. Enrol

`enrollContactInCampaign` — `POST /api/campaigns/enroll/email`

This is the one campaign path that does *not* say workflow. Enrolment is by email.

## 4. Remove a subscriber

`removeCampaignSubscriber` — `DELETE /api/workflows/remove-active-subscriber/{workflowId}/{contactId}`

Removal is by **ids**, not by email — the campaign (workflow) id from step 1 and the contact id from
step 2. Returns 204.

## 5. Tagging as a campaign trigger

`addTagsByEmail` — `POST /api/contacts/email/tags/add`

Campaign triggers in Agile CRM are usually driven off tags. Tag names are case-sensitive, must start
with a letter, and may contain only underscores and spaces as special characters — `Newsletter` and
`newsletter` will trigger different campaigns.

## What you cannot do over the API

Be explicit about this before designing around it:

- **You cannot create, edit or delete a campaign.** The API is list plus enrol plus remove. Campaigns
  are built in the visual workflow builder.
- **You cannot read a contact's campaign status directly.** `campaignStatus`, `unsubscribeStatus` and
  `emailBounceStatus` are read-only fields on the contact object — fetch the contact to see them.
- **You cannot subscribe to campaign events.** The webhook surface covers only four events, all on
  contacts and deals: *Contact is Created*, *Contact is Updated*, *Opportunity is Created*,
  *Opportunity is Updated*. There is no campaign, email-open, click, bounce or unsubscribe webhook.
- **Webhooks cannot be registered programmatically.** They are configured by a human at
  Admin Settings > API & Analytics > Webhooks, and the delivery is unsigned — no signature header, no
  shared secret, no timestamp — so a receiver cannot verify the payload came from Agile CRM. See
  `asyncapi/agile-crm-webhooks.yml`.

## Errors

200 on success (including creates), 204 on delete or on a read with no matching record, 400 on
malformed input with no field-level detail, 401 on a wrong email or REST client API key. No 429, no
rate-limit headers, no idempotency key — a retried enrolment is not deduplicated for you.
