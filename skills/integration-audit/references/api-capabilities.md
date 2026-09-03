# Spiffy API v2 — what the spec doesn't tell you

**The OpenAPI spec is the source of truth for the API surface**, and it is
published live:

- Spec: `https://api.spiffy.co/openapi.json`
- Rendered: `https://developers.spiffy.co/api/getting-started`
- Webhook event catalog: `GET /v2/webhook-event-types` (no auth required)

Endpoints, parameters, filter keys, `include=` values, enums, response schemas,
pagination, rate limits and error codes all come from there. **Do not use this
file for any of that**, and do not carry a mental list of them either — check the
spec for the exact endpoint you are auditing.

The spec is large. Fetch the paths you need rather than loading it whole; for a
list endpoint, the `parameters` array for that path is the authoritative set of
filters and expansions.

This file covers only what a spec structurally cannot state: behaviour, traps,
and the handful of facts that are true but unpublished. Where this file and the
spec disagree about the API surface, **the spec is right and this file is stale
— say so in the report.**

## How to read the spec correctly

**A parameter absent from a path's `parameters` array does not work.** It is not
merely undocumented. Unrecognised filter keys, `include=` values, `search=` and
`sort=` are dropped silently and the request returns 200 with a normal,
unfiltered body. Nothing in the response says so. This is the single most
expensive mistake an integration can make, because the code looks correct while
it pages the entire table.

So: a filter that seems reasonable, or a field that plainly exists on the
object, is not evidence. Only the path's own parameter list is.

**`.in` and `.nin` work on every filterable field**, whether or not that path
enumerates them. `?filter[id.in]=101,102,...&per_page=100` fetches up to a
hundred records by id in one request — the collapse for any loop of single
`GET /v2/{resource}/{id}` calls. The ids must already be known, and chunk at 100
to match `per_page`.

**`include=` on a list endpoint applies per row.** One request with `include=`
beats N follow-up requests every time on quota, but a large `per_page` with
several expansions is the shape most likely to be slow.

## Metering — facts that are not in the spec

Every metered request costs exactly one unit. No endpoint is weighted, and
response size is irrelevant: 100 records in one page costs a hundredth of 100
single-record requests, and `include=` is free.

The unit is counted **before the handler runs**, which produces four consequences
worth checking for in any integration:

- **A burst 429 still costs a monthly request.** The monthly counter increments
  before the per-minute limiter is reached, so a client that hammers the API and
  gets throttled is spending its monthly allowance on the rejections. An
  unbacked-off retry loop burns quota at full speed while receiving nothing.
- **Authentication failures are free; everything after them is not.** A rejected
  credential is refused before metering. A scope failure, a 404, a validation
  error and a server error have all been counted.
- **An empty page costs a full unit.** Paging until a response comes back empty
  spends one extra request per resource per run — use `meta.pagination.has_more`.
- **MCP connector tool calls draw on the same pool.** There is no separate
  allowance; every tool call is one unit against the same monthly quota as HTTP
  traffic, so an assistant left running competes with that account's
  integrations. Help-doc lookups are the exception and are not metered.

## Quota allowances

Both limits are keyed **per account, not per key**. Issuing another key adds no
allowance, and every integration installed on a merchant draws from the same
pool. An integration comfortably inside the limit alone can start failing the
day a second one is installed.

The API reference publishes the current plan allowances. Two things about them
matter for an audit.

**Business is 20,000 and sits below Scale despite sounding larger.** Do not
reason from the plan name.

**Read `X-Monthly-RateLimit-Limit` from a live response rather than any
published table.** An account's allowance can differ from its plan's listed
figure in either direction, and not every account maps to a listed plan — some
have no allowance at all. The header is the only reliable statement of a given
account's limit, and it costs nothing to check.

Published documentation stated 200,000 for the Business plan for a period. If a
developer sized their integration against that figure, their budget is out by an
order of magnitude through no fault of their own, and that is the finding — not
their arithmetic.

Account owners and managers are emailed once per month when usage crosses 80%.

## Measure rather than estimate

`X-Monthly-RateLimit-Remaining` comes back on every response. Two readings an
hour apart give the account's real consumption rate with no arithmetic — and it
captures traffic the codebase cannot show you: other integrations, dashboard
activity, MCP tool calls, non-production environments on the same key. The
merchant's Spiffy settings show the same figure with a projection.

An estimate from source is a **floor**. When the measured total is much larger
than the sum of the call sites you found, that gap is itself a finding.

## Per-resource traps

Behaviour the parameter lists cannot express. Check these before writing up a
finding against any of them:

- **`/v2/checkouts` defaults to `status=active`, and only a top-level `status`
  param clears it.** `?status=expired` replaces the default; `?filter[status]=expired`
  does **not** — the default stays and is ANDed with it, so the query asks for
  rows that are both active and expired and silently returns nothing. This is
  the one place where the two filter syntaxes are not equivalent.
- **`/v2/promos` takes no filters and does not paginate.** It returns every
  active promo in one response. Incremental sync is impossible and inactive
  promos are invisible, so a promo sweep is always a full re-read.
- **`/v2/events?subscription_id` caps at 20 values** and returns 400 above that,
  so the "chunk `.in` at 100" rule does not apply to it. It also expands each id
  backwards along the subscription's swap chain, so the rows returned cover
  earlier subscriptions too — wider than what was asked for.
- **`/v2/events` has no `updated_at`.** The table is append-only. `created_at.gte`
  works, and `id.gte` with `sort=id` is a stricter cursor with no clock-boundary
  risk.
- **Some expansions are always applied**, on `/v2/products/:id`,
  `/v2/affiliates/:id` and `/v2/affiliates/programs/:id`. Passing `include=` for
  those is harmless but buys nothing — don't report its absence as a finding.
- **`sort` honours only the first field** in a comma-separated list. Multi-column
  sort silently does not happen.
- **A small number of write paths bypass the ORM and do not bump `updated_at`**,
  so an `updated_at.gte` delta can miss a row. Pair deltas with a periodic full
  resync.
- **Deletes never appear in a delta.** The row is gone, so an incremental sync
  cannot detect one. Only a periodic reconciliation against the live list will.

## Things developers reliably assume and get wrong

The spec now states these, but they are the assumptions that survive anyway:

- There is no cursor pagination — offset only.
- There are no sparse fieldsets. `fields` is reserved but unimplemented, so
  responses always carry the full resource.
- There are no idempotency keys, so a retried `POST` is a second write.
- There is no caching contract. No `ETag`, `Last-Modified` or `Cache-Control`,
  and nothing short-circuits on a request validator, so a repeated identical
  `GET` is a full query and a full quota unit. **Any caching has to be the
  developer's own** — "add caching" is not useful advice without saying that.
- There are no bulk writes. Every write is one resource per request.

If you catch yourself reasoning "this capability is standard, so it probably
exists" — stop, and check the spec. REST convention is not evidence about this
API.
