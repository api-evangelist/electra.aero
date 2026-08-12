---
name: Read the Electra.aero newsroom
description: >-
  Retrieve Electra.aero press releases, program milestones and syndicated coverage from the
  company's public content API, page through the full archive, and fetch a single entry.
  Use for company-news monitoring, aerospace/advanced-air-mobility research, or building a
  timeline of the EL2 Goldfinch and EL9 Ultra Short programs.
api: openapi/electra.aero-content-openapi.yml
operations:
  - listNewsEntries
  - getNewsEntry
generated: '2026-08-12'
method: generated
source: >-
  Grounded in openapi/electra.aero-content-openapi.yml, which was itself derived from live
  probes on 2026-08-12. Both operationIds exist verbatim in that specification.
---

# Read the Electra.aero newsroom

Electra.aero has no developer program. It does serve one public JSON API — the Statamic
content surface behind its newsroom — at `https://electra.aero/api`. It is anonymous,
CORS-open, and undocumented by the provider. Treat everything below as observed behavior
with no stability guarantee.

## Before you start

- **No credentials.** Send no `Authorization` header, no API key, no cookie. See
  `authentication/electra.aero-authentication.yml`.
- **Only `news` exists.** Every other collection, plus `/api/globals`, `/api/forms`,
  `/api/users`, `/api/assets/*`, `/api/navs/*` and `/api/taxonomies/*`, returns
  `404 {"message":"Not found."}`. Do not construct other paths.
- **No versioning.** There is no version segment or header. A CMS change can alter the
  payload without notice, so validate the fields you depend on rather than assuming them.

## Step 1 — List entries, newest first (`listNewsEntries`)

```
GET https://electra.aero/api/collections/news/entries?sort=-date&limit=25&page=1
```

Verified query parameters — these four and no others:

| Param | Behavior |
|---|---|
| `sort` | Comma-separated fields; leading `-` descends. `sort=-date` is the useful default. |
| `limit` | Entries per page. `meta.last_page` recalculates against it. |
| `page` | 1-indexed. |
| `fields` | Comma-separated sparse fieldset, e.g. `fields=title,slug,date,permalink`. |

**Do not send `filter[...]`.** No filter is allow-listed; every one is rejected with
`422 {"message":"Forbidden filter: <field>"}`. Fetch and filter client-side instead.

## Step 2 — Page through the archive

The response envelope is `{data, links, meta}`. Follow `links.next` until it is `null`, or
loop `page` from 1 to `meta.last_page`. `meta.total` was 124 on 2026-08-12.

Useful entry fields: `id`, `title`, `slug`, `summary`, `date`, `permalink`, `featured`,
`status`, and `source_url` / `source_title` — the last two are populated on syndicated
coverage and null on Electra's own press releases, which is how you separate first-party
announcements from press pickup.

## Step 3 — Fetch one entry (`getNewsEntry`)

```
GET https://electra.aero/api/collections/news/entries/{id}
```

`{id}` is the UUID from `data[].id`. Never construct or guess an id — resolve it from
step 1. The response is `{data: {...}}` with the same field set as a list item.

## Error handling

- `404 {"message":"Not found."}` — ambiguous. It means *either* "no entry with that id"
  *or* "that resource is not exposed at all". There is no error code to tell them apart.
- `422 {"message":"Forbidden filter: <field>"}` — you sent a `filter[...]` parameter.
- Errors are **not** RFC 9457: content-type is `application/json` and the body carries a
  single `message` string with no `type`, `code`, `detail` or request id. Do not parse for
  problem-details members. See `errors/electra.aero-problem-types.yml`.

## Retries, caching and change detection

- No `RateLimit-*` or `Retry-After` headers are returned and no limit is published. Be a
  good citizen: serialize requests and back off exponentially on any non-2xx.
- Every response is `cache-control: no-cache, private` with no `ETag` and no
  `Last-Modified`, so conditional requests do not work. Poll and compare each entry's
  `last_modified` / `updated_at` against your last-seen value.
- There are no webhooks and no event stream, so polling is the only synchronization
  strategy available.
- All operations are safe `GET`s. There is no write surface and therefore no idempotency
  key — never attempt a write against this API.
