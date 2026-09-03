# Usage patterns — what multiplies a call, and what to do about it

A call site's cost is `calls per occurrence x occurrences per day x 30`. The
first number is visible in the code. The second one — the **multiplier** — is
what an audit has to establish, and it is where the large numbers come from.

This file is the catalog of patterns worth recognising. It is a set of starting
points, not a checklist: a pattern here is not automatically a finding, and a
pattern absent from here is not automatically fine. `diagnosis.md` decides what
a pattern *means*.

## The five drivers

Every call site is multiplied by exactly one of these. Name it before costing
anything.

| Driver | The multiplier | Where the evidence is |
|---|---|---|
| **Scheduled** | Runs per day | A crontab, `setInterval`, a queue schedule, a workflow file |
| **Traffic** | Page views, requests, sessions, or active users per day | The call sits inside a request handler, page render, middleware or component |
| **Data** | Records, or pages of records | A loop over a result set, or pagination |
| **Event** | Business volume — orders, payments, signups per day | A webhook handler, or a queue consumer |
| **Fixed** | One, or a handful | Boot, deploy, manual invocation, a one-off backfill |

Two things follow from this table, and they are the reason it exists.

**Scheduled cost is bounded and visible. Traffic cost is neither.** A cron's
multiplier is written down in the repository and cannot exceed 1,440 a day. A
traffic multiplier lives in the developer's analytics, is invisible in the code,
and grows with their success. This is why an integration that was fine for a
year fails in a week, and why traffic-driven call sites exhaust a quota in days
where a chatty cron takes months.

**Data cost scales with the merchant, not the integration.** The same code costs
one account 200 calls and another 20,000. When an integration serves multiple
merchants, size it against the largest, not the average.

## The rule that decides whether a per-occurrence call is a problem

Some calls genuinely have to happen once per occurrence. Flagging those destroys
your credibility on everything else in the report, because the developer knows
they are irreducible and you evidently do not.

> **A per-occurrence call is only a finding if the same answer would serve more
> than one occurrence.**

Apply it literally.

**Reducible — the answer is shared, so the call is waste:**

- Reading a product catalog, a checkout configuration or a promo list on every
  request. The answer is identical for every visitor and changes rarely.
- Re-reading a customer record on every page of one session.
- Fetching aggregates for a dashboard on every render.

**Irreducible — the answer is unique to the occurrence, so the call is correct:**

- **Minting an SSO or magic-link portal token.** It is per user, single-use and
  short-lived by design. It cannot be cached, cannot be shared between users,
  and caching it would be a security bug rather than an optimisation. An
  integration that mints one token per portal visit is doing exactly the right
  thing.
- **A write the user just asked for** — a cancellation, a card update, a refund.
- **A one-time read of a specific record the user just navigated to**, where no
  earlier call in the session already returned it.

For an irreducible call, the finding — if there is one — is capacity, not
design. Size it (`sessions per day x 30`), state what it costs, and say plainly
that it cannot be reduced. If that number alone approaches the quota, the answer
is a plan change or a different architecture, not a code fix, and pretending
otherwise wastes the developer's time.

**The middle case: reduce the frequency, not the call.** Some per-user reads are
irreducible *per session* but are being made per *request*. One access-check or
profile read at session start, cached for the session's life, is a different
call site from the same read on every page view — often by a factor of ten or
more. That is a real finding, and the fix is scope, not removal.

---

## Traffic-driven patterns

The ones that burn a month of quota in days. All of them share a shape: a Spiffy
call on a path that runs when *someone else* decides it runs.

### T1 — A read inside a request handler, page render, or middleware

The call runs once per HTTP request to the developer's own application. At any
real traffic level this dominates every other call site combined, and it is
invisible in a crontab.

**Establish first:** is the answer shared across visitors (reducible) or unique
to this one (irreducible)? Apply the rule above before writing anything up.

**Fix, in order of leverage:**

1. **Move the data out of the request path.** Subscribe to the relevant events,
   mirror into local storage, and serve page loads from that. Steady-state cost
   becomes proportional to business volume rather than traffic — usually orders
   of magnitude less. See `webhook-events.md` for the mirroring architecture.
2. **Cache it.** Spiffy sets no `ETag`, `Last-Modified` or `Cache-Control`, and
   no endpoint short-circuits on a request validator, so a repeated identical
   `GET` is a full query and a full quota unit every time. There is nothing
   server-side to lean on — **the cache has to be the developer's own**, and
   saying "add caching" without saying that is not useful advice. Key it on the
   resource id, give it a TTL matched to how fresh the data actually needs to
   be, and invalidate it from the corresponding webhook.
3. **Re-scope it.** Per session instead of per request; at login instead of per
   page.

### T2 — Rendering Spiffy-owned UI from the API

An integration that fetches subscriptions, payment plans and orders in order to
build its own customer-portal, subscription-management or affiliate screens is
paying quota to rebuild something Spiffy already renders.

**Fix:** `<spiffy-element>` embeds the portal, a single subscription, a single
payment plan, the affiliate portal, or the affiliate registration form directly
in the page. It renders client-side and costs **zero API calls**, at any traffic
level. Signing the customer straight in still needs a per-session SSO token —
which is irreducible per T-rule, and is one call instead of the several the
hand-rolled screen was making per view.

This is only a fix where the developer wants what the embed renders. If they
need their own layout, their own data, or the values inside their own product
surface, it is not a substitute — say so and move on rather than pushing it.

### T3 — Per-component fetch on mount

Several components on one page each fetch independently, so a single page view
costs four or five requests instead of one. Common in component frameworks where
each widget owns its own data fetching.

**Fix:** hoist to one request per page and pass values down, or collapse the set
into a single expanded read with `include=`. Then apply T1 to what remains.

### T4 — Client-side polling in a browser tab

A `setInterval` in front-end code that calls Spiffy, directly or through the
developer's backend. The multiplier is **concurrent open tabs**, not users — a
dashboard left open all day on a 30-second interval is 2,880 requests from one
person, and background tabs keep firing.

**Fix:** drive the update from a webhook into the developer's own backend and
push to the client from there, or drop the interval to the slowest cadence the
feature can tolerate and stop it when the tab is hidden. If it exists because
there is no push mechanism on the developer's side, that is worth recording as
context rather than treating as carelessness.

### T5 — Resolving identity by lookup on every request

Calling a list endpoint with an email filter, or `search=`, on each request to
find the Spiffy customer for the current user.

**Fix:** resolve once, store the returned Spiffy `id`, and use it thereafter.
This is a correctness fix before it is a cost fix — `search=` matches partially
and across several fields at once, so it can return a different customer
entirely. See `security-checks.md` S8 and the identity-keys section of
`webhook-events.md`. Report it even when the volume is negligible.

---

## Scheduled patterns

The classic shape. Bounded, visible in the repository, and usually the easiest
to fix.

### S1 — Polling a resource that now emits events

**Check the polled resource against `webhook-events.md` before writing anything
else up.** Customers, products and promos have a full `created`/`updated`/
`deleted` set, and affiliates have `registered` and `updated`. These are recent
additions, and an integration that polls them was very likely built when polling
was the only option.

**Fix:** subscribe, mirror from the payload, and keep a low-frequency
`filter[updated_at.gte]` sweep as the backfill safety net — events are only
queued to endpoints that were active when they fired, so a sweep is correct
design and must never be recommended away.

**Do not write this up as a mistake.** Establish whether the polling predates
the events, and if it does, say so in the finding.

### S2 — A full resync where a delta filter exists

**Fix:** `filter[updated_at.gte]=<last successful run>`, using the last
*successful* run rather than a fixed window so an outage widens the next sweep
automatically. Confirm the resource supports it by checking that path's
`parameters` in the spec — most do, and `promos` supports no filters at all.

### S3 — A cadence tighter than the data can change, or than the need

A five-minute poll feeding a report someone reads each morning costs 288 runs a
day for one useful one. The fix is a number, and it comes from the developer,
not from you: ask what consumes it and how stale it is allowed to be.

### S4 — A wide rolling window standing in for a delta

`created_at.gte = now - 7 days`, every run. Correct, and pays for the same
records dozens of times. Fix as S2.

### S5 — Tight polling of an asynchronous report run

A report run is asynchronous and takes tens of seconds. A one-second poll loop
spends dozens of units waiting.

**Fix:** exponential backoff. And check the destination — a completed run
carries grand totals inline, so if those answer the question, the results pages
need never be fetched at all.

---

## Data-driven patterns

### D1 — `per_page` left at the default

The default is 50 and the maximum is 100. Every paginated read at the default
costs exactly twice what it needs to.

Cheapest fix in the catalog and worth stating even when small, because it is one
parameter. It does not compound with the fixes below — apply it to whatever
pagination survives them.

### D2 — One request per item after a list request

A list, then a loop of single fetches. Cost is the page size, per page.

**Fix, in order of preference:** `include=` on the list endpoint if the relation
is expandable — one request for what the loop was doing in a hundred. Failing
that, `filter[id.in]` in chunks of 100. Check the endpoint's `parameters` in the
spec for the expansions it actually supports before recommending one, because an
unsupported `include=` value is silently dropped rather than rejected.

### D3 — Fetching everything to use a fraction

Paging a whole resource and then filtering, joining or deduping in application
code. Every page is a request; most of them are discarded.

**Fix:** push the predicate into the query. If no filter supports it, that is a
platform gap and belongs in the feedback section — do not invent a filter key,
and remember an unrecognised one returns 200 with the full unfiltered list.

### D4 — Aggregates assembled by hand

Paging orders or payments to compute revenue, counts, or a time series.

**Fix:** the analytics endpoint answers these in one request, including grouped
time series and period-over-period comparison. For large exact datasets, a
report run's export download is the cheapest bulk path in the API — the download
itself is unmetered, so a full pull costs about four requests at any size.
Confirm the metric or report type exists by calling `GET /v2/analytics/capabilities`
or `GET /v2/reports/types` before promising it.

---

## Event-driven patterns

### E1 — Re-fetching inside a webhook handler what the payload already carried

The most common single waste in an otherwise well-built integration. Order-family
payloads arrive already expanded — items, payments, subscriptions, payment plans,
checkout and checkout view — which is a superset of a bare `GET` of the same
order. A handler that receives one and then fetches the object has paid for data
it was handed for free.

**Fix:** read the payload. Cost drops to zero.

### E2 — Fan-out reads per received event

One event triggering several follow-up calls. Multiplied by business volume, so
it grows with the merchant's revenue.

**Fix:** E1 first, since it often removes the fan-out entirely. Then batch what
remains with `.in`, or serve it from the local mirror.

---

## Waste — costs that buy nothing

These are not architectural. They are requests that return nothing useful, and
they are worth checking on every audit regardless of what else is found.

### W1 — Retrying into a monthly 429

The two 429s are not the same and must be handled differently. A per-minute
`rate_limit_exceeded` carries `Retry-After` and will succeed later. A monthly
`monthly_limit_exceeded` carries no `Retry-After` and **will not succeed again
until the first of the month**.

Client code that treats every 429 as retryable will retry-loop for the rest of
the billing period. Because the monthly counter increments before the per-minute
limiter is reached, those retries keep consuming the allowance they are waiting
on — so the account stays exhausted, and the following month starts with the
loop still running.

**Fix:** branch on the error `code`. On `monthly_limit_exceeded`, stop, alert a
human, and do not resume until the reset timestamp.

### W2 — Ignoring `Retry-After`, or retrying without backoff

A fixed backoff shorter than the window burns quota on rejections that were
already going to fail. Every rejected request is still metered.

### W3 — Non-production environments on a production key

Staging, CI and local development sharing the merchant's key draw on the same
account quota. Worth asking about directly — it is invisible in source and is
occasionally the entire overage on its own.

### W4 — Filters that are silently doing nothing

An unrecognised filter key is dropped and the request returns 200 with the full
unfiltered list. The code looks right, the logs look right, and the integration
pages its entire table.

Both a correctness bug and a cost multiplier, and a finding regardless of
estimated volume — the estimate is a floor, because you are measuring what the
code believes it fetches. Same for an unrecognised `include=`, `search=` or
`sort=` value, and for a webhook subscription naming an event that does not
exist, which simply never fires.

### W5 — Everything else sharing the account's quota

Both limits are **per account, not per key**. Issuing another key adds no
allowance. Anything else installed on that merchant draws from the same pool,
including other integrations, and MCP connector tool calls, which have no
separate bucket.

An integration comfortably inside the limit alone can start failing the day a
second one is installed, and the per-minute limit is where that usually shows up
first. When the measured account total exceeds what this codebase can account
for, the difference is other consumers — find out what they are before
optimising further, because the fix may not be in this repository at all.

### W6 — Health checks and liveness probes against a real endpoint

A monitor hitting a data endpoint every minute is 43,200 requests a month —
more than the two smaller plan allowances outright and most of the largest one,
spent entirely on checking that the API is up.
