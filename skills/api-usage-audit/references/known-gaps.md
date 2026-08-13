# Known Spiffy API v2 gaps

Generated 2026-08-13

This file is the taxonomy behind Part 2 of the integration diagnostic — the part
addressed to Spiffy rather than to the developer. It lists the places where our
API v2 does not give an integrator what they need, so that a report can name a
gap by ID instead of describing it from scratch.

## ID stability rule

Every gap has a permanent ID. IDs are never reused and never renumbered.
Aggregation across many submitted reports depends on `SPF-GAP-003` meaning the
same thing in a report written this month and one written two years ago, so the
numbers have to be durable even when the list around them changes.

When we close a gap, the entry **keeps its ID** and gains a `Resolved in <rev>`
note. It is not deleted and the list is not compacted. An older report that cites
a since-closed gap stays readable, and the reader can see both what was missing
at the time and when it stopped being missing.

New gaps are appended with the next unused number, in discovery order. There is
no significance to the ordering.

## Scope

API v2 only. v1 behavior, dashboard behavior, and the automation "Send Webhook"
action are out of scope for these IDs.

## A note on tone before you use these

Every entry below describes something **we** did not build. A developer who wrote
a 500-call pagination loop, a nightly full resync, or a client-side join was
responding correctly to the API they were given. When a finding maps to a gap
here, report it as our gap. Do not phrase it as a mistake, do not suggest they
"should have" used something that does not exist, and do not soften our side of
it.

---

### SPF-GAP-001 — No cursor pagination; `per_page` capped at 100

**What's missing:** List endpoints paginate by offset only — `?page=N&per_page=M`.
`per_page` is clamped to a maximum of 100 (default 50), and the clamp is silent:
`per_page=1000` returns 100 rows with a 200 and no warning. Pagination metadata is
`{ page, page_size, total_count, total_pages, has_more }`; there is no cursor,
`next` token, or `starting_after` parameter. Every list request also runs a
`COUNT(DISTINCT <table>.id)` over the whole filtered set before fetching the page,
so the count cost is paid again on each page rather than once per sweep, and deep
offsets get slower as the page number climbs.

**Workaround developers use:** A sequential page loop that runs until `has_more`
is false. A 50,000-record resource is at least 500 round trips at the maximum page
size, each one paying the count, and the loop cannot be parallelized safely
because rows shift between pages as data changes underneath it.

**What they'd do instead:** Cursor pagination keyed on `(created_at, id)` with an
opaque `next_cursor` in the response and no count on the hot path — constant-cost
pages, stable under concurrent writes, resumable after a crash without replaying
from page 1. Plus a larger `per_page` ceiling for bulk backfill, where the caller
is explicitly asking for volume.

**How to spot it:** A `while (hasMore)` or `for (let page = 1; ...; page++)` loop
wrapping a fetch of `/v2/<resource>?page=${page}&per_page=100`, exiting on
`meta.pagination.has_more === false` or on `data.length < per_page`. Also look for
`per_page` set above 100 anywhere in the codebase — a `per_page=500` constant means
the author believes their loop runs a fifth as many iterations as it actually
does, and that arithmetic error usually propagates into their own capacity
estimates. Both shapes are correct code against this API; neither is a developer
error.

---

### SPF-GAP-002 — No sparse fieldsets

**What's missing:** `fields` is reserved in the query-param parser — it is
excluded from filter parsing, so it never errors — but nothing reads it. There is
no way to ask for a subset of an object's attributes. Every response is the full
serialized object, plus whatever `include=` expansions were requested.

**Workaround developers use:** Fetch full objects and throw most of them away
client-side. This is most painful on change-detection sweeps, where the caller
wants `id` and `updated_at` and receives forty fields per row for tens of
thousands of rows, paying the bandwidth, the JSON parse, and the memory on every
poll.

**What they'd do instead:** `?fields=id,updated_at,status` to make a delta check
cheap, then a full fetch only for the ids that actually moved. That turns a heavy
sweep into a light one without changing call count.

**How to spot it:** A call whose result is immediately narrowed — a
`.map(o => ({ id: o.id, updated_at: o.updated_at }))` or a destructure keeping
three keys of a large object directly after the fetch. Also grep the query strings
for a literal `fields=`: because the param is reserved rather than rejected, a
developer who assumed it worked gets a 200 with a full payload, and the dead
parameter can sit in production code indefinitely looking like it works. If you
find one, that is our failure to signal, not theirs to detect.

---

### SPF-GAP-003 — No `updated_at` filter on `/v2/payments`

**What's missing:** The payments list accepts `status`, `order_id`, `customer_id`,
`gateway_id`, `currency`, and `created_at` — and no `updated_at`. Its sibling
resources (orders, customers, subscriptions, payment plans, products, affiliates,
checkouts) all do support `updated_at` range filters, so payments is the odd one
out. A payment that is refunded, disputed, or transitions status months after it
was created keeps its original `created_at` and is therefore invisible to any
incremental sweep over that resource.

Compounding it: an unrecognized filter key is dropped silently. A request for
`/v2/payments?updated_at.gte=...` does not 400 — it returns the **full unfiltered
list** with a 200. So the failure mode here is not "the filter is missing", it is
"the filter looks like it worked and quietly returns everything".

**Workaround developers use:** Either a periodic full resync of the payments
resource, or a wide rolling `created_at` window (re-pulling the last 30–90 days
every night) to catch late status changes, or polling `/v2/events` for
payment-related activity and re-fetching those payments by id. All three cost
substantially more calls than a delta sweep would.

**What they'd do instead:** `?updated_at.gte=<last_sync>` on `/v2/payments`,
identical to what already works on orders and subscriptions, reducing a nightly
full resync to a handful of pages.

**How to spot it:** A payments sync whose date filter is `created_at.gte` /
`created_at.lte` where every other resource in the same codebase uses
`updated_at.gte` — the inconsistency is usually a comment or a variable name away
from being explicit ("payments have no updated_at filter"). Also look for a wide
lookback window (`now - 90 days`) applied only to the payments call, and for
`updated_at` appearing in a payments query string at all: silently ignored,
returning everything.

---

### SPF-GAP-004 — `GET /v2/promos` returns everything, unpaginated and unfilterable

**What's missing:** The promos list is not built on the standard list handler. It
returns every active promo for the account in a single unpaginated array, with no
`page`/`per_page`, no filters, no `search`, no `sort`, and no `include`. It also
computes a usage tally per promo, so response time grows with the number of promos
on the account. There is no way to ask for one promo by code, or for only the
promos changed since a timestamp; the only narrower call is
`GET /v2/promos/{id}` when the id is already known.

**Workaround developers use:** Fetch the entire promo set on every run and filter
in memory — typically to find one code, or to diff against a local mirror. On
accounts with a large promo catalog this is a heavy call made frequently for a
small answer.

**What they'd do instead:** The same list contract as every other resource:
pagination, `code` and `is_active` filters, `updated_at` range filtering, and a
lookup by code.

**How to spot it:** A call to `/v2/promos` with no query string at all, followed
immediately by `.find(p => p.code === ...)` or a `.filter(...)` in application
code. Any in-memory search over a promo array right after the fetch is this gap,
not a developer shortcut — there is no server-side alternative to point them at.

---

### SPF-GAP-005 — `search=` is supported on only 4 of the ~10 list endpoints

**What's missing:** `search` is accepted as a reserved query param everywhere but
is only wired up on customers, checkouts, affiliates, and customer cards. On the
endpoints (orders, payments, subscriptions, payment plans, products, promos,
events) the parameter is parsed, found to have no configured search fields, and
dropped. The request returns 200 with the full unfiltered list.

Products is the sharpest edge: the underlying model does define searchable fields
(name, description), but the v2 list endpoint does not pass them through, so
`/v2/products?search=widget` returns every product with no indication that the
search did nothing.

**Workaround developers use:** Paginate the whole resource and match client-side,
or approximate with `name.contains=` where a suitable text filter happens to
exist. Some integrations maintain a local mirror of a resource purely to be able
to search it.

**What they'd do instead:** `search=` on the resource, or — where full-text is not
warranted — an honest 400 telling them the parameter is not supported here, so the
gap surfaces at development time instead of as a silently oversized response bill.

**How to spot it:** A `search=` parameter in a request to any endpoint other than
`/v2/customers`, `/v2/checkouts`, or `/v2/affiliates`. It will look like working
code and produce plausible-looking output, because the caller usually filters the
result again locally. Also look for a full-resource pull feeding a local
`.filter(x => x.name.includes(term))` — that is the workaround shape.

---

### SPF-GAP-006 — No `include=` on the products, affiliates, checkouts, or events lists

**What's missing:** `include=` expansions are available on the orders, customers,
subscriptions, payments, and payment-plans **list** endpoints, but the products,
affiliates, checkouts, and events lists accept no expansions at all. Products and
affiliates do support expansions on their single-resource `GET /:id` route — so
the related data exists and is reachable, just not in bulk. (Promos support
neither; see SPF-GAP-004.)

**Workaround developers use:** List the collection, then loop the ids and call
`GET /v2/<resource>/{id}?include=...` once per row. A 200-product catalog needing
option or checkout data becomes 1 list call plus 200 detail calls per sync — a
client-side N+1 that the developer can see but cannot avoid.

**What they'd do instead:** The same `include=` support the other list endpoints
already have, so one paginated sweep returns the related data inline.

**How to spot it:** A list call followed by an iteration over `data` issuing a
per-item detail fetch — `for (const p of products) await get('/v2/products/' + p.id + '?include=...')`,
`Promise.all(items.map(i => fetchDetail(i.id)))`, or a `p-limit` / `p-map` /
concurrency-limited pool wrapped around a detail fetch. The concurrency limiter is
a strong tell: it means the developer already hit our burst limit doing this and
engineered around it. That is a well-built integration compensating for our gap,
not a performance mistake.

---

### SPF-GAP-007 — No created/updated/deleted events for customers, products, promos, checkouts, or affiliates

**What's missing:** The webhook event catalog is transactional. It covers orders,
payments, subscriptions, payment plans, and cards in depth, plus a few narrow
events on other objects: `customer:marketing:opted_in`, `customer:note:added`,
`product:purchased`, `promo:applied`, and `affiliate:registered`. There is no
`customer:created` or `customer:updated`, no `product:created`/`updated`, no
`promo:created`/`updated`, no `checkout:*` events at all, and no
`affiliate:updated`.

So an integration that mirrors any of those objects — a CRM sync, a catalog
mirror, a coupon mirror — **must** poll. There is no event-driven path available
to it, at any volume.

**Workaround developers use:** A scheduled `updated_at.gte` sweep per resource,
usually nightly or hourly, sized by their freshness requirement rather than by
change volume. In practice this is often the single largest line item in an
integration's call budget, and it scales with catalog size rather than with the
rate of actual change.

**What they'd do instead:** Subscribe to `customer:updated` / `product:updated` /
`promo:updated` and drop the sweep to a low-frequency safety net.

**How to spot it:** A cron, `setInterval`, queue schedule, or workflow trigger
that calls a list endpoint with `updated_at.gte=<last_sync>` for a resource that
has no webhook event covering it, alongside a webhook handler in the same codebase
that handles order- and subscription-family events. The mixed architecture —
events where we offer them, polling where we do not — is the signature. The
polling half is the correct design given the catalog, and should be reported as
our gap.

---

### SPF-GAP-008 — No webhook backfill; replay is limited to events already queued to your endpoint

**What's missing:** When an event fires, we insert a delivery row for each
endpoint that is **active and subscribed at that moment**. An endpoint that was
created later, was disabled, or was not subscribed to that event type at fire time
gets nothing — the event is not stored for it and cannot be produced afterwards.

There is a manual retry for a delivery that already exists
(`POST /v2/webhook-endpoints/{id}/events/{event_id}/retry`, which resets an
existing failed event back to pending), and endpoints auto-retry with backoff.
What does not exist is any way to say "send me everything from the last 6 hours"
or "replay this event type over this date range" for events that were never queued
to you.

Two consequences integrators hit: a receiver that was down long enough to exhaust
the retry schedule, or that got auto-disabled by our backpressure after repeated
failures, has a permanent hole; and a brand-new endpoint starts empty, with no
history.

**Workaround developers use:** A periodic reconciliation sweep over
`updated_at.gte` alongside the webhook receiver — accepting duplicate work in
exchange for gap coverage. Also common: a startup catch-up sweep after deploys,
and persisting every received event id to detect holes.

**What they'd do instead:** A backfill or replay request bounded by time range and
event type, so a receiver can recover a known outage window in one operation
instead of resyncing whole resources.

**How to spot it:** A webhook handler and a scheduled sweep of the same resource
coexisting in one codebase, often with a comment like "safety net" or "in case we
miss a webhook". **Never flag this as redundancy and never recommend removing it.**
It is the correct mitigation for this gap, and it is the difference between an
integration that survives an outage and one that silently loses data.

---

### SPF-GAP-009 — No delete events, and deletions leave no trace in a delta sweep

**What's missing:** Nothing in the webhook catalog ends in `:deleted`. Removal is
also invisible to polling. Most v2 deletes are soft — deleting a product or promo
marks it inactive — but the list endpoints hard-filter to active rows, and
`GET /v2/products/{id}` on a deleted product returns 404. So the row does not
appear in an `updated_at.gte` delta as a tombstone; it simply stops appearing. A
few operations do remove rows outright (a customer merge destroys the losing
customer record, for example), and those leave no trace at all.

A downstream mirror therefore cannot distinguish "unchanged" from "deleted" by any
means the API offers.

**Workaround developers use:** Periodically fetch the complete id set for a
resource and diff it against the local mirror to infer deletions — which requires
the full pagination sweep the delta filter was supposed to avoid, so the deletion
requirement alone forces a full-resync cadence. Others probe individual ids for
404s, or accept permanent drift and let stale records accumulate.

**What they'd do instead:** `*:deleted` webhook events, or a `deleted_at` field
plus an `include_deleted=true` list option, so deletions ride the existing delta
sweep as tombstones.

**How to spot it:** A full-id-set fetch whose only consumer is a set difference
against local state — `const remote = new Set(allIds); for (const localId of mine) if (!remote.has(localId)) markDeleted(localId)`.
Also a handler that treats a 404 from a detail fetch as "deleted", and
reconciliation code with comments about records that "disappear". Any of these
means the developer built our missing tombstone mechanism themselves.

---

### SPF-GAP-010 — No idempotency keys on writes

**What's missing:** v2 accepts no idempotency key on any write. There is no
`Idempotency-Key` header, no request-level dedupe, and no way to retrieve the
result of a previously accepted request. Every POST is processed as a new request.

This matters most on money-moving and record-creating endpoints — charging a
subscription payment, retrying a payment, issuing a refund, creating a product or
promo. A client whose request times out or whose connection drops after the server
began processing has no safe way to find out whether the operation happened. It
can retry and risk a duplicate, or not retry and risk a silent miss.

**Workaround developers use:** Defensive read-before-write (list recent payments
or search for a matching record before retrying), application-side locks or dedupe
tables keyed on their own request id, or a blanket no-retry policy on writes that
turns every transient network error into a manual support ticket.

**What they'd do instead:** Send `Idempotency-Key: <uuid>` and retry freely,
getting the original result back on a repeat rather than a second charge.

**How to spot it:** A retry wrapper (`axios-retry`, `p-retry`, a hand-rolled
exponential-backoff helper) that is explicitly restricted to GET, or a comment
near a POST saying retries are unsafe. The inverse is also worth flagging: a
generic retry wrapper applied to all methods including POST on a payment or refund
endpoint is a live double-charge risk created by this gap — report it as our gap
and warn them plainly, without implying they should have known.

---

### SPF-GAP-011 — No quota-free conditional requests or caching

**What's missing:** There is no designed caching contract. We send no
`Cache-Control` and no `Last-Modified`, we do not honor `If-Modified-Since`, and
there is no "has anything changed since X" endpoint that costs less than the data
itself.

The web framework's default weak `ETag` may produce a 304 for a byte-identical
response body, but that is incidental and helps almost nothing: the query, the
`COUNT(DISTINCT ...)`, and the serialization all ran to produce the body being
hashed, and both the per-minute rate limit and the monthly quota are incremented
before the handler executes. A 304 saves bandwidth. It does not save a call, and
it does not save server work. Any response carrying a computed or time-dependent
field breaks the hash anyway.

The result: a poll over unchanged data costs exactly as much quota as a poll that
returns everything new.

**Workaround developers use:** Polling less often than the business actually
wants, which trades data freshness for quota. Or caching locally with a TTL and
hoping the TTL is shorter than the rate of change. Or building their own
change-detection layer by hashing responses after paying for them in full.

**What they'd do instead:** A conditional request that returns 304 without
consuming quota, or a lightweight change endpoint returning ids and timestamps
only (this pairs directly with SPF-GAP-002).

**How to spot it:** Any polling loop that refetches the same query on a fixed
interval with no conditional headers — and especially a comment or config value
explaining that the interval was raised to stay under the quota. Cadence chosen to
protect quota rather than to meet the freshness requirement is this gap. Also look
for response hashing or "did anything change" comparisons in application code
performed after a full fetch.

---

### SPF-GAP-012 — Rate limit and monthly quota are scoped per account, not per key

**What's missing:** Both limits key on the account: the burst limit (100 requests
per 60-second fixed window) and the monthly request quota. Every API token issued
for the same merchant account draws from the same buckets. There is no per-key
limit, no per-key quota allocation, and no per-key usage attribution in the
response headers — `X-RateLimit-Remaining` and `X-Monthly-RateLimit-Remaining`
describe the account, not the caller.

So two integrations serving the same merchant contend with each other. A batch job
that bursts can 429 an unrelated real-time listener, and neither integration can
see who consumed the quota. For a platform serving many merchants, one noisy
integration starves every other integration on that account, and the developer who
gets paged has no way to prove it was not them.

Also note the fixed window: it resets on a wall-clock boundary rather than
sliding, so two adjacent bursts either side of the boundary can both pass while a
smoother pattern gets rejected.

**Workaround developers use:** A shared client-side token bucket or global
concurrency limiter sized well below the real limit to leave headroom for whoever
else is on the account; jittering job start times to dodge other integrations; and
treating 429s as normal operating conditions with broad backoff.

**What they'd do instead:** Per-key limits and per-key usage reporting, so one
integration's traffic cannot degrade another's and each caller can see and manage
its own consumption.

**How to spot it:** A global rate limiter or semaphore around every Spiffy call
(often a `Bottleneck`, `p-limit`, or hand-rolled token bucket configured
noticeably below 100/min), backoff logic reading `X-RateLimit-Reset` or
`Retry-After`, and scheduling comments about avoiding overlap with another job.
Deliberately under-using the available limit is a rational response to a bucket
they do not control, not over-caution.

---

### SPF-GAP-013 — No bulk export reachable with an API key

**What's missing:** There is no general bulk, batch, or CSV export available to
API consumers. Pulling an entire resource means paging it, at a maximum of 100
records per request, each request also paying a `COUNT(DISTINCT ...)`.

The one exception, and the only bulk read an API key can actually reach, is
`GET /v2/affiliates/payouts/{payout_id}/download`, which streams `text/csv`
without pagination. It covers exactly one thing — a single payout's rows — and
generalizes to nothing else.

A reports system with CSV export does exist, but it sits **outside the v2 API**:
it is mounted at `/reports` rather than `/v2/reports`, and it authenticates with
a dashboard session rather than `public_api`. An API key cannot call it. Do not
recommend it as a workaround — it will return an auth error, not data.

**Workaround developers use:** Sequential paging of the full resource, usually
overnight, often with a concurrency limiter to avoid starving their other jobs
(see SPF-GAP-012).

**What they'd do instead:** A bulk export endpoint reachable with an API key,
returning a full dataset in one or a handful of calls — the way the dashboard's
own report export already works internally.

**How to spot it:** A warehouse load, nightly ETL, BI sync, or "initial import"
routine that pages a resource end to end with no date filter, or with a date
filter spanning all history. Often the largest single line in the call budget.

---

## When nothing here matches — the Unknown bucket

If a developer's problem does not map cleanly onto any ID above, **do not force it
onto the nearest one**. A poor match makes aggregation worse than no match at all:
it inflates a known gap's count with something it is not, and it buries the signal
we did not already have.

Instead, record it in Part 2 under **Unknown**, with:

- **What they were trying to do**, in their own words. Quote them. Their framing
  of the problem is the thing we cannot reconstruct later from a paraphrase.
- **The workaround they built**, concretely enough that an engineer could picture
  the code — with `file:line` evidence where the source was readable.
- **Its call cost**, using the same arithmetic as every other figure in the
  report, tagged `measured`, `stated`, or `assumed`.
- **Why it does not match an existing gap**, in a sentence. If it is adjacent to
  one, say which and how it differs.

These entries are the most valuable output of the whole exercise. The twelve gaps
above are the ones we already know about; a new one is the entire reason to read a
submitted report. Be generous with this bucket. An Unknown that turns out to be a
duplicate of SPF-GAP-005 costs a minute to reclassify. A real gap flattened into a
bad ID is lost.

Repeated Unknowns describing the same thing across reports are how a new
`SPF-GAP-NNN` gets minted.
