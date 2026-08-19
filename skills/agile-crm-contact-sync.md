---
name: agile-crm-contact-sync
description: >-
  Page the full Agile CRM contact book out of the API and keep it in sync — cursor paging, the
  properties model, and updating contacts without clobbering fields.
api: agile-crm:rest-api
generated: '2026-08-13'
method: generated
source: openapi/agile-crm-contacts-api-openapi.yml, openapi/agile-crm-companies-api-openapi.yml
operations:
  - listContacts
  - listCompanies
  - getContact
  - searchContactsAndCompanies
  - updateContactProperties
  - updateContactProperty
  - changeContactOwner
  - filterContactsDynamic
---

# Agile CRM: contact sync

Base URL `https://{domain}.agilecrm.com/dev`. HTTP Basic auth. Always send
`Accept: application/json` — the default response format is XML.

## Paging is not conventional. Read this first.

`listContacts` — `GET /api/contacts?page_size=20&cursor=<opaque>`

The response is a **bare JSON array**, not an envelope, and the paging metadata is hidden inside the
array elements:

- The **total count** is on the **first** element of the array.
- The **cursor for the next page** is on the **last** element of the array.
- **No cursor on the last element means you have reached the end of the list.**

So the loop is: request a page, read the last element's `cursor`, pass it back as `cursor` on the
next request, stop when the last element has no cursor. Do not look for `next`, `has_more` or a
`meta` block; there is none.

A 204 on this call means the account has no contacts at all.

## Companies page differently again

`listCompanies` — `POST /api/contacts/companies/list`

Companies are the *same entity* as contacts, discriminated by `type: COMPANY`. But this endpoint is
a **POST that takes form parameters**, not a GET with query parameters. Send
`Content-Type: application/x-www-form-urlencoded` with `page_size`, `cursor` and optionally
`global_sort_key` (e.g. `-created_time`) in the body.

`filterContactsDynamic` — `POST /api/filters/filter/dynamic-filter` — is form-encoded the same way,
and takes a `filterJson` body parameter carrying the rule set.

## The properties model

A contact's real data lives in a `properties` array, not in top-level fields:

```json
{"name": "email", "type": "SYSTEM", "subtype": "work", "value": "john@example.com"}
```

- `type` is `SYSTEM` for built-in fields or `CUSTOM` for your own. Case-sensitive.
- Only SYSTEM properties take a `subtype`. `email` → `work|personal`. `phone` →
  `work|home|mobile|main|home fax|work fax|other`. `address` → `home|postal|office`. `website` →
  `URL|SKYPE|TWITTER|LINKEDIN|FACEBOOK|XING|FEED|GOOGLE_PLUS|FLICKR|GITHUB|YOUTUBE`.
- `address` values are a JSON *string* containing an object — a nested encoding you must parse.

Top-level fields that are NOT properties: `id`, `type`, `tags`, `lead_score`, `star_value`,
`contact_company_id`, `owner`.

## Updating without clobbering

`updateContactProperties` — `PUT /api/contacts/edit-properties`

This is a *partial* update: send `id` plus only the properties you are changing. Properties you omit
are left alone.

`updateContactProperty` — `POST /api/contacts/add/property` — updates a single property.

`changeContactOwner` — `POST /api/contacts/change-owner` — reassigns ownership. There is no endpoint
that lists users, so the owner id must come from the `owner` object embedded in a record you already
read.

## Read-only fields

Never send these back; they are computed by Agile CRM: `id`, `campaignStatus`, `unsubscribeStatus`,
`emailBounceStatus`, `owner`, `created_time`, `updated_time`. All time fields are **epoch seconds**.

## Search

`searchContactsAndCompanies` — `GET /api/search?q=<term>&page_size=10&type=COMPANY`

Keyword search across contacts and companies; `type` narrows to `PERSON` or `COMPANY`.

## Sync safety

- There are **no ETags, no If-Match and no updated-since filter**, so you cannot do a conditional
  read or a true incremental sync. Sort by `-created_time` on the form-encoded list endpoints to at
  least get newest-first ordering.
- There is **no idempotency key**. A retried create makes a duplicate contact. Before retrying a
  create, call `getContactByEmail` (`GET /api/contacts/search/email/{email}`) to check.
- There are **no published rate limits and no rate-limit response headers**. Choose a conservative
  request rate yourself; the API will not tell you when to slow down.
- Everything is case-sensitive, including the email you use as the Basic auth username.
