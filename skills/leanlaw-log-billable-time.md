---
name: leanlaw-log-billable-time
description: Log a billable time entry against the right client and matter in LeanLaw, attributed to
  the right user, using only verified LeanLaw API operations.
api: LeanLaw API
base_url: https://api.leanlaw.io
operations:
- ListUsers
- ListMatters
- ListClients
- CreateTimeEntry
- GetTimeEntry
- ListTimeEntries
- UpdateTimeEntry
- DeleteTimeEntry
generated: '2026-08-25'
method: generated
source: openapi/leanlaw-api-openapi.json + https://platform.leanlaw.io/patterns
---

# Log billable time in LeanLaw

Records time against a matter. Every time entry in LeanLaw belongs to exactly one matter and always
carries a user, so both must be resolved before creating anything.

## Before you start

- Auth: `Authorization: Bearer {apikey}`. The key is firm-scoped, so it does not by itself identify
  who the time belongs to.
- Send `x-leanlaw-userid: {userId}` on time-tracking calls. This is LeanLaw's documented
  recommendation so matter and time-entry lists are scoped to the acting user.
- All responses are wrapped in `data`. Lists carry a `pagination` object; max page size is 1000.

## Steps

1. **Resolve the user.** `ListUsers` — filter by email to get the `userId` for the person the time
   belongs to. Cache it; you need it for both the header and the entry.

2. **Resolve the matter.** `ListMatters` with `x-leanlaw-userid` set returns the matters that user is
   assigned to. Match on `name`, or on `reference` if the firm gave you its own Matter ID — the
   `reference` is the human-facing "Matter ID", not the GUID. If you only have a client name, call
   `ListClients` first and filter matters by the returned `clientId`.
   - If more than one matter matches, STOP and ask. Billing time to the wrong matter bills the wrong
     client.

3. **Create the entry.** `CreateTimeEntry` with the `matterId`, the `userId`, a date and a
   description. Both a date and a description are always required on a billable item.
   - Expect **201** on success, not 200.
   - `clientId` is derived from the matter; you do not need to send it.

4. **Confirm.** `GetTimeEntry` with the returned `timeEntryId`, or `ListTimeEntries` scoped to the
   user and date.

## Correcting an entry

- `UpdateTimeEntry` is a **sparse PUT** — send only the fields you are changing. Omitting a field does
  NOT clear it. This is the opposite of replace semantics.
- `DeleteTimeEntry` removes it. No retention window, soft delete or restore path is published, so
  treat deletion as permanent and confirm with a human first.
- If the entry has a non-null `invoiceId` it has been billed. LeanLaw does not document what deleting
  or editing a billed entry does to the invoice — do not do it unattended.

## Error handling

- `400` invalid request — check required fields (date, description, matterId, userId).
- `401` no key. `403` key lacks write permission, or is invalid. A read-only key cannot create.
- `429` throttled. No `Retry-After` and no rate-limit headers are published, so use exponential
  backoff.
- `500` server error — capture `x-leanlaw-traceid` and report it.

## Retry safety

**There is no idempotency key on this API.** If `CreateTimeEntry` times out or returns 429, do not
blindly retry — you will create a duplicate billable entry. Instead call `ListTimeEntries` filtered
to the user and date and check whether the entry landed before retrying.

## LEDES

If the matter has `ledesConfiguration.enabled` (fetch with `select=ledesConfiguration` on
`GetMatter`), the firm may require LEDES activity and/or task codes on time entries
(`activityCodeRequired`, `taskCodeRequired`). Pull valid codes from `GetCodes`, restricted to the
matter's `codeSetIds`.
