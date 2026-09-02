# Mirroring Spiffy data with webhooks

**`GET /v2/webhook-event-types` is the source of truth for event names.** It
needs no authentication and returns the live catalog. Call it, and check every
event an integration subscribes to against what comes back. Do not work from a
remembered list, and never invent an event name — a subscription to an event
that does not exist is accepted, stored, and simply never fires.

Delivery semantics, signature verification, the retry ladder and endpoint
management are documented in the API reference at
`https://developers.spiffy.co/api/getting-started`. This file covers what that
reference does not: what each payload actually contains, and what a
well-designed mirror looks like.

## What each payload carries

**Order-shaped events carry a superset of a bare `GET`.** For `order:*`,
`payment:*`, `upsell:*` and `payment_plan:*`, plus `product:purchased` and
`promo:applied`, `data.object` is a v2 order pre-expanded with `items`,
`payments`, `subscriptions`, `paymentplans`, `checkout` and `checkoutview`.

A receiver cannot pass `?include=`, so the payload arrives complete. Two of
those expansions are effectively webhook-only, since neither appears in a
default `GET`: `checkout` (the published offer configuration) and `checkoutview`
(UTM data for the converting visit).

**The practical consequence: an order-shaped receiver generally needs zero
follow-up calls.** A handler that receives `order:success` and then fetches the
order, customer, items, payments, subscriptions, payment plans or checkout is
paying for data it was handed.

**The one exception is `attribution`.** First- and last-touch attribution is a
different dataset from `checkoutview`'s converting-visit UTMs, and it is not in
the payload. A receiver that needs first-touch data does have to call
`GET /v2/orders/{id}?include=attribution`. Do not report that as redundant.

**Everything else delivers its own object**, and this catches people out:

| Event | `data.object` |
|---|---|
| `subscription:*` | The subscription, plus an embedded `customer` |
| `card:*` | The card, with internal `sources` stripped, plus an embedded `customer` |
| `customer:*` | The customer |
| `product:created`, `product:updated`, `product:deleted` | The product |
| `promo:created`, `promo:updated`, `promo:deleted` | The promo |
| `affiliate:updated` | The affiliate |
| `affiliate:registered` | The customer |

Note the split within a family: `product:purchased` and `promo:applied` deliver
an **order**, because they describe a purchase, while the CRUD events for those
same resources deliver the resource itself. An integration mirroring a catalog
wants the CRUD events, and their payload is already the object being mirrored.

## The recommended shape

Measure an integration against this. It is what "using the API well" looks like
for anything that keeps a local copy — a warehouse, a fulfilment system, a
customer portal.

**Build the mirror from the payload, not from follow-up reads.** See above: the
data is usually already there.

**Dedupe on the envelope `id`.** It is stable across every retry, so it is the
natural idempotency key. Delivery is at-least-once — the same event *will*
arrive twice eventually, and an upsert that isn't idempotent will double-apply.

**Don't trust arrival order.** A `subscription:canceled` can land after a later
state change and overwrite it. Compare the object's own timestamps before
writing rather than assuming the newest delivery is the newest truth.

**Acknowledge fast, then own the retry.** Respond 2xx first and do the write
asynchronously — but once you have returned 2xx the event is considered
delivered and will never be resent. Any failure after that point is yours to
retry, from your own queue.

**Keep a delta sweep as the safety net.** Events are only queued for endpoints
active when they fire, so anything that happened while an endpoint was down,
disabled or unsubscribed is unrecoverable. A low-frequency
`filter[updated_at.gte]=<last successful run>` sweep closes that hole. Use the
last *successful* run as the lower bound, not a fixed window, so an outage
widens the next sweep automatically.

`promos` is the exception — it takes no filters, so a promo sweep is a full
re-read of the active set. Keep it, but run it rarely.

**And a periodic full resync, at much lower frequency**, because deltas cannot
surface records that disappeared.

## Identity keys — store ids, never email

Store the Spiffy `id` for every object you mirror and use it as the join key
against your own records. Capture it at the first point of association and keep
it.

**Email is not an identifier.** It is mutable, customers change it, several
people can share one, and merging reassigns it. An integration keyed on email
will silently mis-link records the first time any of those happen, and
mis-linking one customer to another's orders is a data breach rather than a bug.

Matching by email is also weaker than it looks: `search=` matches partially and
across several fields at once, so it can return a different customer entirely —
see `security-checks.md` S8. If you must resolve by email, use the exact filter
`filter[email]=`, treat the result as a candidate rather than an identity, and
store the returned `id` so you never have to do it again.

**Ids can disappear, and nothing tells you.** Merging two customers deletes the
secondary record, and there is no merge event. A stored `customer_id` can start
returning 404 with no signal. Handle a missing id as "re-resolve and re-link",
not as "the customer is gone".

## What not to cache

Values Spiffy computes rather than stores — promo usage tallies, customer stats,
aggregate counts — are derived at read time and go stale immediately. Cache the
objects, not the arithmetic over them.

## Coverage gaps — what still needs polling

Check the live catalog before treating any of these as current; they are the
gaps as last verified, and they move.

- **No CRUD events for checkouts.** Customers, products and promos have a full
  create/update/delete set, and affiliates have `registered` and `updated`.
  Checkouts have none, so mirroring them means polling `GET /v2/checkouts` on
  its `updated_at` filter.
- **No `subscription:updated`.** Only the lifecycle transitions. A field edit
  that does not cross one of them produces no event.
- **No price-change or discount events.** Changing a price, or applying or
  removing a subscription discount, fires nothing.
- **No report-run-completed event.** Completion has to be polled.
- **Delete events exist only for customers, products and promos.** Orders,
  subscriptions, payment plans, checkouts and affiliates have no delete signal,
  and a hard delete never appears in an `updated_at` delta either — only a
  periodic full reconciliation will catch one.
- **No backfill.** Events never queued to an endpoint cannot be recovered, which
  is why the delta sweep above is correct design rather than waste.

**The CRUD events are recent.** An integration that polls customers, products or
promos was very likely built when polling was the only option — this is the most
common reason an otherwise well-built integration is over quota today. Treat it
as a fix, but check whether the polling predates the events and say so rather
than writing it up as an oversight.
