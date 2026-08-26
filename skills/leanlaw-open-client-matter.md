---
name: leanlaw-open-client-matter
description: Open a new client and matter in LeanLaw, understanding that creating a matter can also
  write customer records into the firm's QuickBooks Online ledger.
api: LeanLaw API
base_url: https://api.leanlaw.io
operations:
- ListClients
- CreateClient
- GetClient
- UpdateClient
- ListPracticeAreas
- ListUsers
- CreateMatter
- GetMatter
- UpdateMatter
- DeleteMatter
generated: '2026-08-25'
method: generated
source: openapi/leanlaw-api-openapi.json + https://platform.leanlaw.io/concepts + https://platform.leanlaw.io/changelog
---

# Open a client and matter in LeanLaw

## Read this first — this flow reaches outside LeanLaw

Since 2026-08-17, for firms using the QuickBooks Online integration, **`CreateMatter` also creates
records in QuickBooks**: the client is added as a QuickBooks customer if not already connected, and
the matter is added as a sub-customer when the firm bills per matter. `CreateClient` on its own does
not reach QuickBooks — the accounting write happens at matter creation.

`DeleteMatter` is **not documented as removing** the QuickBooks customer or sub-customer it created.
This flow is therefore only partially reversible. Confirm with a human before creating matters in
bulk.

## Steps

1. **Check the client does not already exist.** `ListClients`, matching on `name` and on `reference`
   (the firm's own Client ID). Creating a duplicate client duplicates it into QuickBooks downstream.

2. **Create the client if needed.** `CreateClient`. Expect **201**. Include `reference` if the firm
   uses its own client numbering. Contact details are part of the client; you can read them back
   with `GetClient?select=contact`.

3. **Resolve the responsible user.** `ListUsers`. A matter requires exactly one responsible user.
   Originators are a separate, multi-valued field (`originatorIds`).

4. **Resolve the practice area** (optional). `ListPracticeAreas`. If the firm's practice area does
   not exist yet, `CreatePracticeArea` — but check the list first.

5. **Create the matter.** `CreateMatter` with `clientId`, `name`, `responsibleId`, and optionally
   `reference`, `practiceAreaId`, `originatorIds`, `matterType`. Expect **201**.

6. **Verify.** `GetMatter` with `select=meta,ledesConfiguration` to confirm the record and see whether
   LEDES billing is enabled for it.

## Conventions that will bite you

- **`reference` is not the id.** `matterId` and `clientId` are GUIDs. `reference` is the firm's own
  "Matter ID"/"Client ID" as users see it, it is optional, and a firm may use it on matters, clients,
  both or neither.
- **`UpdateMatter` / `UpdateClient` are sparse PUTs.** Only what you send changes. You cannot clear a
  field by omitting it.
- **Custom fields are read-only.** `ListCustomFields` returns the firm's field definitions and
  `select=customFields` returns values on a record, but you cannot set them through the API.
- Responses are wrapped in `data`; lists carry `pagination` with a `total`.

## Retry safety

There is no idempotency key. If `CreateClient` or `CreateMatter` times out, **re-list before
retrying** — a blind retry creates a duplicate client or matter, and for matters that duplicate
propagates into QuickBooks.

## Errors

`400` invalid request · `401` missing key · `403` key not authorized (a read-only key cannot create)
· `429` throttled, back off exponentially, no `Retry-After` is sent · `500` capture
`x-leanlaw-traceid` and report.
