# Spiffy API v2 — capability catalog

Generated 2026-08-13 against API v2. Derived from server source, not published
documentation.

**This catalog is closed-world.** If a capability is not listed here, it does not
exist. Do not infer capabilities from REST convention, from other APIs, or from
what would be reasonable. If something a developer needs is absent, that is a
finding for Part 2 of the report, not a gap in this document.

Everything below applies to `/v2` endpoints reached with an account API key or an
OAuth access token. It says nothing about v1.

## Pagination

Every list endpoint that paginates does so with offset pagination:

| Param | Default | Rules |
|---|---|---|
| `page` | `1` | 1-based. Values below 1 are clamped to 1. |
| `per_page` | `50` | Hard maximum 100. Larger values are clamped to 100 silently — no error, no warning. Unparseable values fall back to 50. |

The list envelope is:

```json
{
  "data": [ ... ],
  "meta": {
    "pagination": {
      "page": 2,
      "page_size": 100,
      "total_count": 4213,
      "total_pages": 43,
      "has_more": true
    }
  }
}
```

`has_more` is `page < total_pages`. Loop on `has_more`, not on an empty `data`
array.

Published documentation in some places shows `pagination: { page, limit, total,
pages }`. The server does not emit that shape. The five keys above are what
arrives; code reading `meta.pagination.total` or `.pages` reads `undefined`.

Cost note: every list request runs two queries — a `COUNT(DISTINCT <table>.id)`
over the filtered set, then the page fetch. The count does not get cheaper as the
caller pages deeper, so a full-table page-through costs one count per page on top
of the page itself. Filtering narrows both.

## The silent-ignore contract

**This is the highest-value check in an audit. Run it on every request the
integration makes.**

Unknown query params are dropped. A filter key that is not on the endpoint's
allowed list does not 400 and does not warn — the request returns **200 with the
full, unfiltered list**. `?status=active` against an endpoint with no `status`
filter returns every row, and the caller pages through all of them believing they
are filtered.

This has already caused a production incident. A lister was configured with a
filter key the handler factory does not read, so its whole filter set was
disabled. `?status=` and `?product_id=` became no-ops, and one merchant's daily
sync paged roughly 80,000 subscriptions at 100 per page — about 800 calls per
run, every run, for data it then discarded client-side.

The failure is invisible from the outside:

- The caller's code looks correct; the URL contains the filter.
- The response is a 200 with a well-formed body.
- The caller's logs show successful requests.
- Only the row count and the call volume are wrong, and nothing compares them.

So checking the literal filter keys against [Filters by resource](#filters-by-resource)
is mandatory, not optional. Any key not in that table is dead weight. The same
contract applies to `include=` values (unknown expansions are dropped) and to
`search=` on endpoints that do not support it.

## Filtering syntax

Two equivalent syntaxes, both accepted on every filterable list:

```
?status=active
?filter[status]=active
?updated_at.gte=2026-01-01T00:00:00Z
?filter[updated_at.gte]=2026-01-01T00:00:00Z
?filter[status.in]=active,past_due
?filter[email.contains]=@example.com
```

Operators, appended to the field name after a dot:

| Operator | Meaning | Notes |
|---|---|---|
| `.eq` | equals | The default when no operator is given |
| `.ne` | not equals | |
| `.gt` `.gte` `.lt` `.lte` | range comparisons | |
| `.contains` | substring match | **Text fields only.** Ignored on number, enum, boolean and date fields |
| `.notcontains` | substring non-match | Text fields only |
| `.in` | value in list | Comma-separated: `.in=a,b,c` |
| `.nin` | value not in list | Comma-separated |
| `.null` | is / is not null | `.null=true` → `IS NULL`; `.null=false` → `IS NOT NULL` |

There is no `between`. Use `.gte` plus `.lte` on the same field.

Multiple filters, and multiple operators on one field, combine with AND. There is
no OR across different fields.

Value handling worth knowing:

- Digit-only strings are coerced to numbers, `true`/`false` to booleans, and the
  literal `null` to SQL null.
- An empty value (`?status=`) is skipped entirely — it does not filter and does
  not error.
- Dates and datetimes must be **ISO 8601, UTC** (`2026-01-15T00:00:00Z`). A Unix
  timestamp is rejected with a 400 naming the field and the bad value.
- Number-typed filters reject non-numeric input with a 400 (`?order_id=abc`).
- Enum-typed filters reject unknown labels with a 400 that lists the valid values.
  That error message is the cheapest way to discover an enum's legal values.
- Text `.eq`, `.contains` and `.notcontains` are case-insensitive. `.ne` and the
  range operators are case-sensitive.

## Filters by resource

Exact allowed filter keys, per list endpoint. A key not in this table is silently
ignored (see [The silent-ignore contract](#the-silent-ignore-contract)).

| List endpoint | Filter keys | Incremental sync |
|---|---|---|
| `GET /v2/customers` | `id`, `email`, `name_first`, `name_last`, `company_name`, `created_at`, `updated_at` | `updated_at` |
| `GET /v2/orders` | `id`, `customer_id`, `checkout_publish_id`, `checkout_id`, `currency`, `display_total`, `promo_id`, `created_at`, `updated_at` | `updated_at` |
| `GET /v2/subscriptions` | `id`, `status`, `order_id`, `customer_id`, `email`, `product_id`, `product_option_price_id`, `created_at`, `updated_at`, `next_payment_at` | `updated_at` |
| `GET /v2/paymentplans` | `id`, `status`, `order_id`, `customer_id`, `email`, `created_at`, `updated_at`, `next_payment_at` | `updated_at` |
| `GET /v2/payments` | `status`, `order_id`, `customer_id`, `gateway_id`, `currency`, `created_at` | **none — `created_at` only** |
| `GET /v2/products` | `id`, `name`, `stripe_product_id`, `is_taxable`, `is_commissionable`, `is_subscription`, `use_options`, `created_at`, `updated_at` | `updated_at` |
| `GET /v2/affiliates` | `id`, `email`, `name_first`, `name_last`, `slug`, `paypal_email`, `is_ready_for_payout`, `created_at`, `updated_at` | `updated_at` |
| `GET /v2/checkouts` | `status`, `product_id`, `created_at`, `updated_at` | `updated_at` |
| `GET /v2/events` | `id`, `type`, `name`, `user_id`, `customer_id`, `order_id`, `order_item_id`, `subscription_id`, `paymentplan_id`, `payment_id`, `notification_id`, `created_at` | none — `created_at` only |
| `GET /v2/customers/:customer_id/cards` | `id`, `type`, `last_four`, `is_active`, `created_at` | none |
| `GET /v2/promos` | none — see below | none |
| `GET /v2/affiliates/programs` | none | none |
| `GET /v2/affiliates/programs/:program_id/affiliates` | none | none |
| `GET /v2/affiliates/programs/:program_id/links` | none | none |
| `GET /v2/affiliates/programs/:program_id/options` | none | none |
| `GET /v2/affiliates/programs/:program_id/prices` | none | none |
| `GET /v2/affiliates/payouts` | none | none |
| `GET /v2/webhook-endpoints` | none | none |
| `GET /v2/webhook-events` | `webhook_endpoint_id`, `status`, `event_type` — plain params only, **not** `filter[...]` syntax | none |
| `GET /v2/webhook-endpoints/:id/events` | `status` — plain param only, **not** `filter[...]` syntax | none |
| `GET /v2/webhook-event-types` | none | none |

Exceptions and traps in that table:

- **`/v2/payments` has no `updated_at` filter.** It filters on `created_at` only.
  A refund, dispute or status change on an older payment is therefore invisible to
  a `created_at`-based delta — the row's creation date has not moved. Payment state
  changes must come from webhooks, or from a periodic full resync.
- **`/v2/promos` does not paginate and takes no filters.** It returns every active
  promo for the account in one `{ "data": [...] }` response with no `meta`. Do not
  send `page`/`per_page`; they do nothing.
- **`/v2/webhook-endpoints` returns a bare JSON array**, not the `{ data }`
  envelope, and does not paginate.
- **The webhook event lists read plain query params.** `?status=failed` works;
  `?filter[status]=failed` does not. `status` accepts only `pending`, `delivered`,
  `failed` — any other value is ignored and the list comes back unfiltered.
- **`/v2/checkouts` defaults to `status=active`.** Sending any `status` filter
  replaces that default rather than narrowing it. `product_id` supports equality
  only; an operator suffix on it returns 400.
- **`/v2/events` `created_at` only.** Events are append-only, so `created_at.gte`
  is the correct delta key here.

Enum values accepted by the status filters:

| Filter | Values |
|---|---|
| `/v2/subscriptions?status=` | `pending`, `trialing`, `active`, `past_due`, `canceling`, `canceled`, `unpaid`, `swapped` |
| `/v2/paymentplans?status=` | `active`, `past_due`, `unpaid`, `completed`, `defaulted` |
| `/v2/payments?status=` | `pending`, `succeeded`, `failed`, `refunded`, `disputed`, `dispute_refunded` |
| `/v2/checkouts?status=` | `active`, `expired`, `deleted` |
| `/v2/events?type=` | `user`, `customer`, `system` |

`/v2/events?name=` is an enum too, with roughly seventy values. Send a deliberate
bad value once and read the 400 — it lists them all.

### Incremental sync

For the six resources with an `updated_at` filter, `?filter[updated_at.gte]=<last
run>` combined with `sort=updated_at` is the intended sync pattern and usually
turns a full page-through into a handful of calls per day.

Two caveats to carry into any recommendation:

- A small number of write paths bypass the ORM and do not bump `updated_at`, so a
  delta can miss a row. Pair the delta with a periodic (e.g. weekly) full resync.
- Hard deletes never appear in a delta. The periodic full resync covers them.

## Include matrix

`?include=a,b` expands related data into each item. `?expand=` is accepted as an
alias. Values are comma-separated and case-insensitive; unknown values are
silently dropped.

| Endpoint | Available `include=` values |
|---|---|
| `GET /v2/customers`, `GET /v2/customers/:id` | `cards`, `fields`, `stats`, `integrations`, `integration_mappings`, `subscriptions` |
| `GET /v2/orders`, `GET /v2/orders/:id` | `fields`, `integrations`, `items`, `payments`, `subscriptions`, `paymentplans`, `customer`, `affiliate`, `checkout`, `checkoutview` |
| `GET /v2/subscriptions`, `GET /v2/subscriptions/:id` | `payments`, `usage`, `custom_fields`, `customer`, `order` |
| `GET /v2/paymentplans`, `GET /v2/paymentplans/:id` | `items`, `payments`, `order`, `customer`, `custom_fields`, `card`, `gateway` |
| `GET /v2/payments`, `GET /v2/payments/:id` | `customer`, `order` |
| `GET /v2/affiliates/:id` | `customer`, `cards` (plus `stats` and `gravatar`, which are always included) |
| `GET /v2/checkouts/:id` | `items` — always included; the param is unnecessary |
| `GET /v2/products/:id` | `checkouts`, `integration_mappings` — both always included; the param is unnecessary |
| `GET /v2/affiliates/programs/:program_id` | `stats` — always included |

Lists that support `include=`: customers, orders, subscriptions, paymentplans,
payments. That is the complete set.

Lists that do **not** support `include=` — the param is accepted and ignored:
products, affiliates, checkouts, events, promos, customer cards, every affiliate
sub-resource list, and every webhook list.

Cost note: expanders run per item, server-side, after the page is fetched. A page
of 100 with three includes runs three expansions a hundred times. That is still
the right trade against 100 follow-up round trips — saving those round trips is
the entire point — but it is not free, and a large `per_page` combined with
several includes is the shape most likely to be slow. If only some items need the
expansion, a smaller page or a targeted single-resource `GET` may cost less
overall.

## Search support

`search=` performs a case-insensitive substring match across a fixed set of
columns for that endpoint. It is honoured on exactly four endpoints:

| Endpoint | Fields searched |
|---|---|
| `GET /v2/customers` | first name, last name, email |
| `GET /v2/affiliates` | first name, last name, email, slug, PayPal email |
| `GET /v2/checkouts` | internal name, offer name |
| `GET /v2/customers/:customer_id/cards` | last four, cardholder first name, cardholder last name |

**Everywhere else `search=` is silently ignored** and the endpoint returns the
unfiltered list. There is no search on orders, subscriptions, paymentplans,
payments, products, promos, events, or any webhook list.

One behavioural detail: a multi-word search ORs every term across every field, so
`search=john smith` matches rows containing *either* word. It broadens the result
set rather than narrowing it. To look up one record by a known value, use an exact
filter (`?email=`) instead of `search=`.

## Sorting

`?sort=field` sorts ascending; `?sort=-field` sorts descending.

- **One field only.** Extra comma-separated fields are parsed and discarded — only
  the first is applied.
- Sortable fields are the filterable fields for that endpoint, with two exceptions
  that are filter-only and not sortable: `checkout_id` on orders and `product_id`
  on subscriptions. `product_id` on checkouts is also a join-based filter rather
  than a column on the resource — filter by it, do not sort by it.
- **An invalid or unsupported sort field is silently ignored** and the endpoint
  falls back to its default sort. No 400. Code that depends on a particular
  ordering should verify the ordering it actually gets.
- Defaults are `created_at` descending (newest first) on every resource list. The
  exception is `/v2/webhook-event-types`, which defaults to `type` ascending and
  accepts only `type` or `description` as sort fields.

For a delta sync, `sort=updated_at` with `filter[updated_at.gte]` gives stable
forward paging; the default descending `created_at` does not.

## Not supported

These do not exist in v2. Do not recommend them, and do not read their absence in
a developer's code as an oversight on their part.

- **Sparse fieldsets.** `fields` is reserved as a query param name — meaning it can
  never be a filter key — but nothing implements it. Responses always carry the
  full serialized resource. There is no way to ask for fewer fields.
- **Cursor / keyset pagination.** Offset only. There is no `cursor`, `starting_after`,
  `ending_before` or `next` link anywhere in the envelope.
- **Idempotency keys.** No endpoint reads an `Idempotency-Key` header. A retried
  POST is a second write. Retry logic on writes needs application-level
  deduplication.
- **Conditional requests and caching.** v2 sets no `Cache-Control`, `ETag` or
  `Last-Modified` semantics of its own, and no handler short-circuits on request
  validators. A repeated identical GET costs a full query and a full quota unit.
- **Bulk / batch writes.** Every write is one resource per request.
- **A general bulk export or CSV endpoint.** See below for the one exception.

## Bulk reads

There is one endpoint in v2 that returns a whole dataset in a single call:

```
GET /v2/affiliates/payouts/:payout_id/download
```

It returns `text/csv` containing every commission row in that payout, with no
pagination and no `per_page` ceiling — one request and one quota unit, instead of
one request per page of `GET /v2/affiliates/payouts`. Use it whenever the goal is
the full commission set for a payout rather than a filtered slice.

No equivalent exists for customers, orders, subscriptions, paymentplans, payments,
products or events. For those, the cheapest full read is `per_page=100` plus the
narrowest filter that satisfies the use case, and for ongoing sync it is
`updated_at.gte` deltas plus webhooks. If a developer needs a bulk export of one
of those resources, that is a genuine gap and belongs in the report — do not
invent an endpoint for it.

## Rate limits and quotas

Two independent limiters run on v2 requests, both keyed by **account**. They meter
traffic authenticated with an account API key. An approved third-party developer
application is metered against its own allocation, set when developer access is
granted; the limits below are still the right numbers to design against, because
any integration a merchant installs with their own key lands squarely on them.

### Per-minute limit

- 100 requests per 60-second fixed window.
- **Keyed per account, not per key.** Every key issued by a merchant draws on the
  same 100/minute budget, so two integrations installed on one account contend
  with each other — and with anything else that account is running. An integration
  that is comfortably inside the limit alone can start failing the day a second
  one is installed.
- Fixed window, not sliding: the counter resets at the top of each window, so a
  burst that straddles a boundary can pass while the same burst inside one window
  fails.

Headers on every response:

| Header | Meaning |
|---|---|
| `X-RateLimit-Limit` | Requests allowed in the window (100) |
| `X-RateLimit-Remaining` | Requests left in the current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |

On breach: **429** with `Retry-After` in seconds and body
`{ "error": { "code": "rate_limit_exceeded", "message": "Too many requests" } }`.
Honour `Retry-After`; a fixed backoff that is shorter than it just burns quota.

### Monthly quota

Counted per account per calendar month (UTC), reset on the first of the month.

Headers on every response:

| Header | Meaning |
|---|---|
| `X-Monthly-RateLimit-Limit` | The account's monthly allowance |
| `X-Monthly-RateLimit-Remaining` | Requests left this month |
| `X-Monthly-RateLimit-Reset` | Unix timestamp of the first of next month (UTC) |

On breach: **429** with code `monthly_limit_exceeded` and **no `Retry-After`
header**. Retrying does not help; the quota returns at the month boundary given by
`X-Monthly-RateLimit-Reset`. Client code that treats every 429 identically will
retry-loop for the rest of the month. Branch on the error `code`.

Allowances by plan tier, as enforced in code:

| Tier | Monthly requests |
|---|---|
| `pro_yr`, `pro_mo` | 10,000 |
| `scale_yr`, `scale_mo` | 50,000 |
| `highvol_yr`, `highvol_mo` | 20,000 |
| `founders_yr`, `founders_mo` | 10,000 |
| `compd`, `compd_native` | 10,000 |
| `sandbox` | 10,000 |
| `inactive` | 0 — every request 429s |

An account whose tier is not in that list is treated as 0 until an override is set.

Documentation elsewhere states 200,000 for High Volume. The enforced value is
20,000. Use 20,000 in any calculation; a sizing exercise built on 200,000 will be
off by an order of magnitude.

An account can also carry an individual override that supersedes its tier, in
either direction. **Read `X-Monthly-RateLimit-Limit` from a live response rather
than assuming the tier value** — it is the only reliable source for a given
account. Account owners and managers are emailed once per month when usage crosses
80%.

### What this means for call budgeting

At the 10,000/month tier, sustained polling has to fit in roughly 13 calls per
hour across the whole account, for every integration installed on it. A five-minute
cron that pages three resources is already over budget before a second integration
exists. The arithmetic that matters in an audit is:

```
calls per run x runs per day x 30  vs  the account's monthly allowance
```

Every page of a paginated read counts as one call. So does every 429, and every
request whose filters were silently ignored.
