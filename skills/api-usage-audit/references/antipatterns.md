# Antipattern catalog

Generated 2026-08-13 against API v2.

Detection recipes for the audit. Each entry says what to look for, what it costs,
what to do about it, and — importantly — **which part of the report it belongs
in**.

The Part 1 / Part 2 split is not cosmetic. Part 1 entries are things the
developer can fix with capabilities that exist today. Part 2 entries are things
they built around because we don't offer an alternative. Putting a Part 2 finding
in Part 1 means telling someone to fix something they can't, using an API that
doesn't exist. That is the worst output this skill can produce.

---

## Part 1 — fixable with what exists today

### AP-01 — Polling a resource that has full webhook coverage

**Detect:** A scheduled call to `GET /v2/orders`, `/v2/subscriptions`,
`/v2/payments` or `/v2/paymentplans` on any cadence, where the handler only reads
fields present in the webhook payload.

**Why it costs:** Every run pays full pagination for data that would have been
pushed. A 5-minute cadence is 8,640 runs/month before pagination multiplies it.

**Fix:** Subscribe to the relevant events in `webhook-events.md`. The
order-family payload already carries `items`, `payments`, `subscriptions`,
`paymentplans`, `checkout` and `checkoutview`, so the receiver usually needs no
follow-up call at all.

**Mandatory caveat:** Pair the migration with a low-frequency `updated_at.gte`
reconciliation sweep. A failed delivery can be retried, but an event that was
never queued to the endpoint cannot be backfilled (SPF-GAP-008).

**Report as:** Part 1.

---

### AP-02 — An unrecognized filter key silently returning everything

**Detect:** Any filter param whose key is not in the allowed-filter table for
that resource in `api-capabilities.md`. Check every filter key on every call
site. Common shapes: a plausible-but-wrong name (`state` for `status`), a field
that exists on the object but isn't filterable, or a filter copied from a
different resource.

**Why it costs:** Unknown filter params are silently ignored and the request
returns **200 with the full unfiltered list**. The caller believes it is fetching
a narrow slice and is actually paging the entire table. This is usually the
single most expensive bug in an integration, and it appears in neither the code
nor the logs as an error.

**Fix:** Correct the key against the table. If no equivalent filter exists for
what they need, it becomes a Part 2 finding instead.

**Report as:** Part 1, and rank it first regardless of estimated volume — the
estimate is probably an undercount.

---

### AP-03 — Client-side N+1 where `include=` is available

**Detect:** A loop over a list response issuing one request per item to fetch a
related resource, on a list that supports `include=` (customers, orders,
payments, paymentplans, subscriptions).

**Why it costs:** N extra calls per run, where N is the page size. On a
100-record page that's 100 calls instead of 0.

**Fix:** Add the relation to the list request with `include=`. Note that
expansion runs per-item server-side, so a large page with several includes is
slower per request — it is still far cheaper than N round trips, and quota is
counted per request.

**Report as:** Part 1. If the list does **not** support `include=`, this is
SPF-GAP-006 and belongs in Part 2.

---

### AP-04 — `per_page` left at the default

**Detect:** A paging loop with no `per_page` param, or one set below 100.

**Why it costs:** The default is 50 against a maximum of 100, so the loop makes
twice the calls it needs to.

**Fix:** Set `per_page=100`.

**Report as:** Part 1. Small on its own; often large in combination with a full
resync.

---

### AP-05 — Full resync where `updated_at.gte` is supported

**Detect:** A scheduled paging loop with no date filter, over a resource that
supports `updated_at` (customers, orders, subscriptions, paymentplans, products,
affiliates).

**Why it costs:** Every run pages the entire history rather than the delta.

**Fix:** Filter on `filter[updated_at.gte]=<last run>`, ISO 8601 UTC.

**Caveat to state:** Keep a periodic full resync anyway, at a much lower
frequency. Deltas miss hard deletes entirely (SPF-GAP-009). Recommend reducing
the frequency of the full sync, never removing it.

**Report as:** Part 1.

---

### AP-06 — withdrawn

Previously recommended the reports CSV export as a bulk-read escape hatch. That
was wrong: `/reports` is not part of the v2 API, is mounted outside `/v2`, and
authenticates with a dashboard session rather than `public_api`. An API key
cannot call it. See **AP-17** for the correct classification.

The ID is retained rather than reused so that reports generated against earlier
drafts remain readable.

---

### AP-07 — No backoff on 429, or ignoring `Retry-After`

**Detect:** A request helper that retries immediately on any non-OK response, or
that treats 429 like any other error.

**Why it costs:** Retrying into a rate limit consumes quota without ever
succeeding, and lengthens the outage.

**Fix:** Honour `Retry-After` on the per-minute 429. Note that the **monthly
quota 429 does not send `Retry-After`** — that one needs a long fallback and an
alert, since retrying won't help until the month rolls over.

**Report as:** Part 1.

---

### AP-08 — Re-fetching inside a webhook handler

**Detect:** A webhook handler that receives an event and immediately calls
`GET /v2/<resource>/{id}` for the same object.

**Why it costs:** One avoidable call per event, scaling with order volume.

**Fix:** Read `data.object` from the payload. For order-family events it is
already expanded, and includes `checkout` and `checkoutview` — which a plain GET
does not return unless explicitly requested.

**Report as:** Part 1. Common in integrations that migrated from polling and kept
the old fetch path.

---

### AP-09 — Polling faster than the data can change

**Detect:** A cadence out of proportion to the resource — a nightly-settled
report polled every minute, payouts polled hourly, products polled every five
minutes.

**Why it costs:** Directly proportional waste.

**Fix:** Match the cadence to how fast the data actually changes, or move to
events where they exist. Ask the developer what latency the flow genuinely needs
— the answer is often much looser than the schedule implies.

**Report as:** Part 1.

---

## Part 2 — correct workarounds for missing capability

The developer did nothing wrong in any of these. State the cost and what they'd
do instead. **Do not recommend a fix that does not exist**, and do not phrase
these as mistakes.

### AP-10 — Polling a resource with no webhook coverage

**Detect:** Scheduled polling of customers, products, promos, checkouts or
affiliates.

**Why:** No `created`/`updated`/`deleted` events exist for these, so mirroring
them requires polling.

**Report as:** Part 2, SPF-GAP-007. If they're already using `updated_at.gte`
and `per_page=100`, they are doing this optimally — say so.

---

### AP-11 — Full resync to catch deletes

**Detect:** A periodic full sync whose stated purpose is finding removed records.

**Why:** No delete events exist, and hard deletes never appear in an `updated_at`
delta.

**Report as:** Part 2, SPF-GAP-009. Never recommend removing it.

---

### AP-12 — Over-fetching for lack of sparse fieldsets

**Detect:** Large list responses where the handler reads two or three fields.

**Why:** `?fields=` is reserved but unimplemented; responses are always the full
serialized object.

**Report as:** Part 2, SPF-GAP-002. Bandwidth and parse cost, not quota cost —
say so rather than overstating it.

---

### AP-13 — Client-side joins where `include=` is unavailable

**Detect:** Per-item fetches against products, affiliates, promos, reports or
events lists.

**Why:** Those lists don't support `include=`.

**Report as:** Part 2, SPF-GAP-006.

---

### AP-14 — Payments resync for lack of an `updated_at` filter

**Detect:** A full or `created_at`-based resync of `/v2/payments` whose purpose
is catching refunds or status changes.

**Why:** Payments filter on `created_at` only. A status change on an older
payment is undiscoverable by delta.

**Report as:** Part 2, SPF-GAP-003. Worth flagging even when volume is modest,
because the developer may not realise their delta is silently incomplete.

---

### AP-15 — Client-side dedupe standing in for idempotency keys

**Detect:** Application-level guards against duplicate writes — a local ledger of
attempted operations, a uniqueness check before a POST.

**Why:** No `Idempotency-Key` support on writes, so retry-after-timeout risks
double-creating.

**Report as:** Part 2, SPF-GAP-010.

---

### AP-16 — withdrawn

Previously described polling `GET /v2/reports/{id}/runs/{run_id}` for report-run
completion. There is no reports API in v2 at all, so no API consumer can be doing
this. Withdrawn for the same reason as AP-06.

If you encounter something that genuinely looks like report-run polling, it is
either v1, a dashboard session, or something we haven't catalogued — put it in
the Unknown bucket rather than forcing it here.

---

### AP-18 — Re-polling unchanged data at full quota cost

**Detect:** Any recurring poll where most runs return data identical to the
previous run — a catalog sweep, a settings mirror, a low-churn resource on a high
cadence.

**Why:** There is no conditional-request path that avoids consuming quota. A poll
that returns nothing new costs exactly the same quota unit as one that returns a
full page of changes, so there is no client-side way to make a no-op poll cheap.

**Report as:** Part 2, SPF-GAP-011. Usually a compounding factor on another
finding rather than a headline — if the developer is already using
`updated_at.gte` and a sensible cadence, this is the residual cost they cannot
remove. Note it as such rather than minting it as a separate problem.

---

### AP-17 — Paging an entire dataset because no bulk export is reachable

**Detect:** A warehouse load, nightly export, or analytics job that pages a whole
resource end to end.

**Why:** There is no bulk or CSV export available to API consumers. The reports
system exists, but it lives outside the v2 API and authenticates with a dashboard
session, so an API key cannot reach it. Paging is the only option available.

**Cost:** Hundreds of quota calls for one logical operation, each also paying a
`COUNT(DISTINCT ...)`.

**Report as:** Part 2, SPF-GAP-013. Do not recommend the reports export — it is
not callable with their credentials.

---

## When nothing matches

If a call site is expensive and matches no entry here, do not force it onto the
nearest label. Describe what it does, what it costs, and why none of these fit,
and put it in Part 2 under **Unknown**. Those entries are the most valuable thing
this audit produces, because they're the ones we couldn't predict.
