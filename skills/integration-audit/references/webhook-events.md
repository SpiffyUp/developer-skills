# Spiffy webhook events — catalog

Generated 2026-08-13 against API v2. Derived from server source.

**This catalog is closed-world.** The event types below are the complete set. If
a resource has no event listed here, mirroring it requires polling. Say so and
cite the gap ID — do not invent an event name, and do not infer one from REST
convention or from what another API offers. A recommendation to subscribe to an
event that does not exist is worse than no recommendation at all, because the
developer will build against it and it will never fire.

The live list is also served, unauthenticated, by `GET /v2/webhook-event-types`.
If this file and that endpoint disagree, the endpoint is right.

## Event types

42 event types, in 9 families. An endpoint subscribes to an explicit list of
types, or to `"*"` for all of them. `"*"` is the default when an endpoint is
created without an `events` array.

### order (5)

| Type | Fires when |
|---|---|
| `order:success` | An order completes successfully |
| `order:migrate` | An order is migrated |
| `order:adjustment:added` | An adjustment is added to an order |
| `order:note:added` | A note is added to an order |
| `order:complete` | An order is marked complete |

### payment (5)

| Type | Fires when |
|---|---|
| `payment:success` | A payment succeeds |
| `payment:failed` | A payment fails |
| `payment:refunded` | A payment is refunded |
| `payment:disputed` | A payment is disputed |
| `payment:requires_action` | A payment requires additional action |

### subscription (14)

| Type | Fires when |
|---|---|
| `subscription:started` | A subscription starts |
| `subscription:unpaid` | A subscription becomes unpaid |
| `subscription:canceling` | A subscription is set to cancel |
| `subscription:resumed` | A subscription is resumed |
| `subscription:canceled` | A subscription is canceled |
| `subscription:expired` | A subscription expires |
| `subscription:swapped_from` | A subscription is swapped from |
| `subscription:swapped_to` | A subscription is swapped to |
| `subscription:trial_end` | A subscription trial ends |
| `subscription:trial_end:upcoming` | A subscription trial is about to end |
| `subscription:recurring_payment:success` | A recurring payment succeeds |
| `subscription:recurring_payment:upcoming` | A recurring payment is upcoming |
| `subscription:recurring_payment:failed` | A recurring payment fails |
| `subscription:recurring_payment:refunded` | A recurring payment is refunded |

These are lifecycle transitions. There is no generic `subscription:updated`.

### card (4)

| Type | Fires when |
|---|---|
| `card:expiring` | A card is about to expire |
| `card:expired` | A card has expired |
| `card:updated` | A card is updated |
| `card:auto_updated` | A card is automatically updated |

### customer (2)

| Type | Fires when |
|---|---|
| `customer:marketing:opted_in` | A customer opts in to marketing |
| `customer:note:added` | A note is added to a customer |

Both are activity events. Neither is a create/update/delete signal, so neither
substitutes for mirroring customers.

### product / upsell (2)

| Type | Fires when |
|---|---|
| `product:purchased` | A product is purchased |
| `upsell:success` | An upsell succeeds |

Both describe a purchase, not a change to the product catalog.

### payment_plan (8)

| Type | Fires when |
|---|---|
| `payment_plan:success` | A payment plan is created successfully |
| `payment_plan:unpaid` | A payment plan becomes unpaid |
| `payment_plan:resumed` | A payment plan is resumed |
| `payment_plan:defaulted` | A payment plan transitions to defaulted (terminal write-off) |
| `payment_plan:payment:failed` | A payment plan payment fails |
| `payment_plan:payment:refunded` | A payment plan payment is refunded |
| `payment_plan:payment:upcoming` | A payment plan payment is upcoming |
| `payment_plan:payment:success` | A payment plan payment succeeds |

### promo (1)

| Type | Fires when |
|---|---|
| `promo:applied` | A promo code is applied |

### affiliate (1)

| Type | Fires when |
|---|---|
| `affiliate:registered` | An affiliate is registered |

There is also an internal `test` event type, emitted only by
`POST /v2/webhook-endpoints/{id}/test`. It carries a fixed stub body, not a real
object, and is not subscribable.

## Payload shape

Every delivery is a POST with this envelope:

```json
{
  "id": "evt_9f1c3a2e-5d47-4b0a-9c11-8e6f2b74d013",
  "object": "event",
  "type": "order:success",
  "api_version": "v2",
  "created": 1755043200,
  "account_id": 42,
  "data": { "object": { "...full v2 API object..." } }
}
```

- `id` is `evt_` plus a UUID, assigned when the event is queued. It is stable
  across every retry of that event, which makes it the natural idempotency key.
- `created` is a Unix timestamp in seconds, taken from when the event row was
  created, not from when this attempt was sent.
- `account_id` is the Spiffy account the event belongs to.

The key fact for anyone weighing a migration: **`data.object` is produced by the
same v2 serializers the REST API uses.** It is not a reduced or bespoke shape. A
handler that already parses a v2 API response can parse a webhook body with the
same code.

### Order-family payloads carry a superset of a bare GET

For `order:*`, `payment:*`, `product:*`, `upsell:*`, `payment_plan:*` and
`promo:*`, `data.object` is a **v2 order**, loaded with these expansions:

```
items, payments, subscriptions, paymentplans, checkout, checkoutview
```

That list is deliberately a superset of what `GET /v2/orders/{id}` returns.
`GET` has no default includes: a bare `GET` returns the base order and expands
only what the caller asks for via `?include=`. A webhook receiver cannot pass
query params, so the payload has to arrive complete.

Two of those expansions matter more than the rest, because in practice they are
webhook-only:

- **`checkout`** — the published offer configuration the order was bought
  against.
- **`checkoutview`** — UTM and attribution data for the visit that converted.

Both are ordinary opt-in expanders on `GET`, but neither appears in a default
`GET` response. They ship in every order-family webhook.

The practical consequence: **an order-family webhook receiver generally needs
zero follow-up API calls.** If you find a handler that receives `order:success`
and then calls `GET /v2/orders/{id}`, or fetches the customer, items, payments,
subscriptions, payment plans, checkout or attribution separately, those calls
are redundant — the data is already in the body it was handed.

### What the other families deliver

Do not generalize the order payload to the rest. Each family has its own primary
object:

| Event family | `data.object` |
|---|---|
| `order:*`, `payment:*`, `product:*`, `upsell:*`, `payment_plan:*`, `promo:*` | v2 order with the six expansions above |
| `subscription:*` | The subscription object, plus an embedded `customer` |
| `card:*` | The card, with internal `sources` stripped, plus an embedded `customer` |
| `customer:*`, `affiliate:*` | The customer object |

The embedded `customer` on subscription and card payloads is the base customer
serialization (id, account_id, email, name, company, tax id, timestamps). It is
not the richer `?include=customer` expansion, so it carries no custom fields and
no cards. A subscription reached through a swap (`subscription:swapped_from`)
takes a direct load and therefore ships the base row without the order-scoped
extras (`detail`, `usage`, `retry_schedule`). A missing buyer serializes to
`null` on the subscription and card paths.

One edge case worth handling defensively: if the queued event is missing the
reference ID for its primary object, the delivery falls back to sending the raw
reference IDs as `data.object` rather than failing. Receivers should tolerate a
body that is thinner than the table above promises.

## Signature verification

Every delivery carries a `Spiffy-Signature` header:

```
Spiffy-Signature: t=1755043200,v1=3f8a...c4
```

- `t` is the Unix timestamp in seconds at which the signature was generated.
- `v1` is a hex-encoded HMAC-SHA256.

The signed string is the timestamp, a literal `.`, then the JSON body:

```
signature = HMAC_SHA256(secret, `${t}.${JSON.stringify(payload)}`)
```

The secret is the endpoint's own `signing_secret` (prefix `whsec_`), returned
once when the endpoint is created and again when the secret is rotated. Each
endpoint has its own secret.

Verify against the exact bytes you received. Re-serializing the parsed JSON is
not guaranteed to reproduce the same string. Compare with a constant-time
comparison, and reject stale `t` values on your own policy — the server does not
impose a signature validity window.

Requests are also sent with `User-Agent: Spiffy-Webhooks/1.0` and
`Content-Type: application/json`. Neither is authenticated; only the signature
is.

## Delivery guarantees

Delivery is **at-least-once**. Receivers must be idempotent. Dedupe on the
envelope `id`, which stays constant across retries of the same event.

Ordering is not guaranteed. A dedicated processor claims pending events in
batches with `SKIP LOCKED`, so concurrent workers and retries can interleave.
Do not assume a `subscription:canceling` arrives before its
`subscription:canceled`. Use the object's own state, not arrival order.

A delivery counts as successful only on a **2xx within a 10 second timeout**.
Redirects are not followed, so a 301 or 302 is a failure. Respond 2xx fast and
do the work asynchronously.

### Retry ladder

Up to 7 attempts, with these delays before each attempt:

| Attempt | Delay since previous | Elapsed since first attempt |
|---|---|---|
| 1 | 0 (immediate) | 0 |
| 2 | 1 minute | ~1m |
| 3 | 5 minutes | ~6m |
| 4 | 30 minutes | ~36m |
| 5 | 2 hours | ~2h 36m |
| 6 | 8 hours | ~10h 36m |
| 7 | 24 hours | ~34h 36m |

After the 7th failed attempt the event is marked `failed` and is never retried
automatically. The total window is roughly 35 hours.

### Per-endpoint backpressure and auto-disable

`failure_count` is tracked per endpoint and shared across all its events. Any
successful delivery resets it to 0. As it climbs, deliveries are deliberately
slowed:

| Consecutive failures | Effect |
|---|---|
| 11–25 | 5 second delay before each delivery |
| 26–49 | 30 second delay before each delivery |
| 50 | Endpoint auto-disabled; no further deliveries |

At 50 the endpoint's `status` flips to `disabled`, `disabled_at` is stamped, and
the account's owners and managers are emailed with a deep link to the webhook
settings. Nothing is queued for a disabled endpoint, and nothing is replayed
when it is re-enabled. Re-activating an endpoint through the API (`PATCH` with
`status: "active"`) resets `failure_count` and clears `disabled_at`, but the
events missed while it was disabled are gone.

## No replay or backfill

This is the constraint that decides whether a webhook migration is safe, so read
it before recommending one.

Events are queued **only for endpoints that are `active` at the moment the event
fires**. The dispatcher selects endpoints by `status = 'active'` and subscription
match, then writes one event row per matching endpoint. An endpoint created
tomorrow has no rows for anything that happened today.

Consequences, all of them permanent:

- **There is no backfill.** Registering an endpoint does not seed it with
  history. There is no "send me everything since timestamp X" call.
- **`retry` is not replay.** `POST /v2/webhook-endpoints/{id}/events/{event_id}/retry`
  re-queues an event row that already exists. It cannot create one, so it cannot
  recover an event that was never queued for that endpoint.
- **Downtime beyond the retry window is unrecoverable.** If a receiver is down
  for more than about 35 hours, or the endpoint auto-disables at 50 consecutive
  failures, those events are lost with no way to get them back.

### The rule that follows

**Any recommendation to replace polling with webhooks must pair the webhook with
a low-frequency reconciliation sweep** — a periodic list call filtered on
`updated_at.gte` (supported on orders and subscriptions), run hourly or daily,
to catch whatever the webhook path missed.

The reason is one sentence: recommending webhooks without a safety net gets the
reader burned during an outage, and the loss is silent and permanent.

The sweep is not a fallback to the old polling cadence. Going from a 5-minute
poll to webhooks plus a nightly sweep still removes almost all of the call
volume, which is the entire point.

## Coverage gaps — what you must still poll for

These absences are real. When a developer's polling loop covers one of them,
that belongs in Part 2 of the report as a platform gap, not in Part 1 as their
mistake. Do not suggest a fix that does not exist.

- **No `created` / `updated` / `deleted` events for customers, products, promos,
  checkouts or affiliates.** `customer:marketing:opted_in`, `customer:note:added`
  and `affiliate:registered` exist, but none of them is a CRUD signal, and
  `product:purchased` describes a purchase rather than a catalog change. Mirroring
  any of these resources requires polling.
- **No `subscription:updated`.** Only the 14 lifecycle transitions listed above.
  A field edit that does not cross one of those transitions produces no event.
- **No price-change or discount events.** Changing a price, or applying or
  removing a subscription discount, fires nothing. `promo:applied` fires at
  checkout on an order; it is not a promo-object change event.
- **No report-run-completed event.** A report run's completion has to be polled.
- **No delete events at all.** No family has one. A hard delete also never
  appears in an `updated_at` delta, because the row is gone, so an incremental
  sync alone cannot detect it. Catching deletions requires a periodic full
  reconciliation against the live list.
- **No backfill**, as above.

## Endpoint management

Developers can self-serve their webhook configuration through the API. All
routes require OAuth with the `webhooks` scope, and every endpoint is scoped to
the authenticated OAuth client.

| Route | Purpose |
|---|---|
| `POST /v2/webhook-endpoints` | Create. Returns the `signing_secret` — this is the only time it is exposed on create. |
| `GET /v2/webhook-endpoints` | List this client's endpoints. |
| `GET /v2/webhook-endpoints/{id}` | Fetch one. |
| `PATCH /v2/webhook-endpoints/{id}` | Update `url`, `events`, `description`, or `status` (`active` / `inactive`). |
| `DELETE /v2/webhook-endpoints/{id}` | Delete. |
| `POST /v2/webhook-endpoints/{id}/rotate-secret` | Mint a new signing secret and return it. |
| `POST /v2/webhook-endpoints/{id}/test` | Send a synchronous `test` event and return the response code, timing and body. |
| `GET /v2/webhook-endpoints/{id}/events` | Delivery history for the endpoint. |
| `GET /v2/webhook-endpoints/{id}/events/{event_id}` | One event with all its delivery attempts. |
| `POST /v2/webhook-endpoints/{id}/events/{event_id}/retry` | Re-queue an existing event row. Rejects an event already pending. |
| `GET /v2/webhook-event-types` | The live catalog. No auth required. |

Constraints worth knowing before recommending a design:

- **Maximum 5 endpoints per OAuth client.**
- **URLs must be HTTPS**, must be a hostname rather than a raw IP, and cannot be
  `localhost` or end in `.local` or `.internal`. The server also resolves the
  host at delivery time and refuses private or loopback addresses.
- `events` must be a non-empty array when supplied. Omitting it on create
  subscribes the endpoint to `"*"`.
- The delivery history is per endpoint, so it is a usable debugging surface: it
  records status code, response time, error message and a truncated response
  body for every attempt.
