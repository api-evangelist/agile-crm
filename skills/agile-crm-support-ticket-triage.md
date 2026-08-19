---
name: agile-crm-support-ticket-triage
description: >-
  Triage Agile CRM helpdesk tickets — list them through a filter, read the message thread, add an
  internal note, and close the loop back onto the contact record.
api: agile-crm:rest-api
generated: '2026-08-13'
method: generated
source: openapi/agile-crm-helpdesk-api-openapi.yml, openapi/agile-crm-notes-api-openapi.yml, openapi/agile-crm-contacts-api-openapi.yml
operations:
  - listTicketFilters
  - listTickets
  - listTicketMessages
  - addTicketNote
  - createTicket
  - deleteTicket
  - getContactByEmail
  - createNote
---

# Agile CRM: helpdesk ticket triage

Base URL `https://{domain}.agilecrm.com/dev`. HTTP Basic auth, `Accept: application/json`.

The helpdesk paths do not read the way you would guess. Tickets live under `/api/tickets`, but the
endpoint that *adds* a note to a ticket lives under `/api/notes`. Use the operations below rather
than constructing paths.

## 1. Get the filter ids first

`listTicketFilters` — `GET /api/tickets/filters`

Tickets are not listed by a plain collection GET. You list them *through a filter*, so you need a
filter id before you can read anything.

## 2. List tickets

`listTickets` — `GET /api/tickets/filter`

Returns tickets for the requested filter. Agile CRM publishes no field table for the ticket object —
only example payloads — so do not assume a field exists until you have seen it in a live response.

## 3. Read the thread

`listTicketMessages` — `GET /api/tickets/notes/{ticket_id}`

Returns all messages within the ticket.

## 4. Add an internal note

`addTicketNote` — `POST /api/notes/{ticket_id}`

Note the path: adding a ticket note goes to `/api/notes/{ticket_id}`, **not** to anything under
`/api/tickets`.

## 5. Open a ticket

`createTicket` — `POST /api/tickets/new-ticket`

Returns 200 on success, not 201.

## 6. Close the loop on the contact

Helpdesk tickets and CRM contacts are separate objects with no id reference between them in the
published field tables. To connect a ticket to a CRM record, resolve the requester by email and write
a contact note:

`getContactByEmail` — `GET /api/contacts/search/email/{email}` (204 = no such contact)
`createNote` — `POST /api/notes` with `subject`, `description` and `contact_json` (an array of
contact ids). Both `subject` and `description` are mandatory.

## 7. Deleting

`deleteTicket` — `DELETE /api/tickets/{id}` — returns **204**.

On a DELETE, 204 means it worked. On a GET, 204 means the record does not exist. Branch on the verb
before deciding whether a 204 is success or absence.

## Errors

| Status | Meaning |
|---|---|
| 200 | Success, including on create |
| 204 | Deleted (on DELETE) *or* no such record (on GET) |
| 400 | Input in the wrong format — no field-level detail returned |
| 401 | Wrong account email or REST client API key |

There is no 429 and there are no rate-limit headers. There is no idempotency key, so a retried
`createTicket` after a timeout opens a second ticket — read the filter list back before retrying.
