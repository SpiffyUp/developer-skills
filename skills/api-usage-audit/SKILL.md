---
name: auditing-spiffy-api-usage
description: Use when a developer wants to review, diagnose, or optimize how their application uses the Spiffy API — including questions about API call volume, hitting rate limits or the monthly quota, whether to use webhooks instead of polling, or when Spiffy has asked them to run an integration diagnostic.
---

# Auditing Spiffy API usage

You are auditing how this project uses the Spiffy API v2, and producing a single
file — `spiffy-integration-diagnostics.md` — that the developer can act on and,
if they choose, send to Spiffy.

## What this does and does not do

Tell the user this before you start, in your own words:

- It reads their integration source and produces one markdown file.
- It does **not** modify their code.
- It does **not** transmit anything anywhere. Sending the report to Spiffy is
  their explicit choice at the end, and the report gets a redaction pass first.

They are running a vendor-supplied tool against a private codebase. Their
suspicion is reasonable — address it up front rather than waiting to be asked.

## Before you start — establish your mode

1. Load the four reference files from this skill's `references/` directory:
   `api-capabilities.md`, `webhook-events.md`, `known-gaps.md`,
   `antipatterns.md`. If you cannot fetch URLs, ask the user to paste them.
2. Try to read the project's source.
3. Set your mode:

   - **Full** — references loaded and source readable.
   - **Interview-only** — references loaded, source unreadable. Every volume
     figure comes from the user and is tagged `stated`.
   - **Blocked** — references unavailable. Say so and stop.

**In Blocked mode, stop.** Do not proceed from memory. You do not know Spiffy's
filter keys, event names, or limits, and the failure mode is not a vague answer —
it is a confident, specific, wrong one.

State your mode to the user before continuing, and record it in the report.

## Hard rules

- Never claim a capability absent from `api-capabilities.md`. That catalog is
  closed-world.
- Never recommend a webhook event absent from `webhook-events.md`.
- Never assert a call volume without showing its inputs and arithmetic.
- Every figure carries a confidence tag: `measured`, `stated`, or `assumed`.
- A finding without `file:line` evidence goes under "Needs confirmation", not
  among the findings.
- If a resource's filter or `include=` support is not in the catalog, write "not
  covered by this catalog" rather than inferring it.
- Never recommend removing a reconciliation sweep. Spiffy can retry an event
  already queued to your endpoint, but it cannot backfill events that were never
  queued — anything that fired while an endpoint didn't exist, was disabled, or
  wasn't subscribed is unrecoverable. A periodic delta sweep is correct design,
  not waste.

Violating the letter of these rules is violating their spirit. If you catch
yourself reasoning "this capability is standard, so it probably exists" — stop.
That is the exact failure these rules exist to prevent. REST convention is not
evidence about this API.

## Procedure

### 1. Scope and announce

Confirm this project calls the Spiffy API v2. Say what you're about to do and
what you won't do. If the project only uses API v1, say that this audit covers v2
only and ask whether to continue anyway.

### 2. Inventory the call sites

Search the project for `api.spiffy.co`, `/v2/`, `Spiffy-Signature`, webhook route
handlers, and any wrapper or SDK the project uses around Spiffy calls. Follow the
wrapper to its call sites rather than stopping at its definition.

For each call site record: `file:line`, method, endpoint, the query params
actually passed, and what triggers it — cron, queue, request-scoped, webhook
handler, or manual.

Check every filter param you find against the allowed-filter table in
`api-capabilities.md`. **An unrecognized filter key is silently ignored and
returns the full unfiltered list with a 200.** This is usually the single most
expensive bug in an integration, and it is invisible in both the code and the
caller's logs. Look for it specifically.

### 3. Determine cadence

From the code wherever it exists: cron expressions, `setInterval`, queue
schedules, retry configuration, workflow definitions. Tag each call site
`measured` (cadence found in code) or needs-asking.

A cron expression written in a **comment**, with no scheduler config in the
project, is weaker evidence than a real schedule. Tag it `measured` but add a
"needs confirmation" entry naming the crontab or job runner that would settle it
— the whole budget scales off these numbers, so an unverified cadence is worth
one line.

### 4. Interview — only the gaps

Ask these in one batch, and skip any the code already answered:

1. What is this integration for, in a sentence or two?
2. How many Spiffy accounts or merchants does it serve?
3. Roughly how many records exist in the resources you sync (orders, customers,
   subscriptions)?
4. Which flows need fresh data within seconds rather than hours?
5. What is your Spiffy plan tier?
6. Do you have webhook endpoints configured today? If not, what stopped you?

Question 6 matters more than it looks. The answer is the most likely place an
unknown gap surfaces — "we tried but couldn't verify signatures behind our
proxy", "we need an event that doesn't exist". Treat the answer as data for
Part 2, not as small talk.

### 5. Build the call budget

For each call site: `calls per run × runs per month = calls per month`. Show the
inputs.

> Nightly, ~12,000 orders at `per_page=50` = 240 pages + 1 = 241 calls/run × 30
> = **7,230 calls/month** [`measured` cadence, `stated` record count]

Then total, rank by share of total, and compare against both ceilings:

- The **monthly quota** for their plan tier (see `api-capabilities.md`).
- Peak **burst** against 100 requests per 60 seconds — which is scoped per
  account, not per key, so other integrations on the same merchant share it.

The ranked table is the centerpiece of the report. "92% of your traffic is one
endpoint" is what makes the rest of the report worth reading.

**When estimated demand exceeds the ceiling by a wide margin**, the percentage
columns stop describing reality — the account exhausts its quota early and
everything after that is a 429. Report the demand figure, then explain separately
what actually happens at runtime: when the quota runs out, that the per-minute
limit is breached first, whether retries multiply the breach, and whether runs
overlap. A budget that reads "‑2,000,000% headroom" is arithmetically correct and
useless on its own.

### 6. Classify every finding

Three buckets:

- **Part 1 — their fix.** A capability that already exists would solve it.
- **Part 2 — our gap.** No capability exists. Cite the `SPF-GAP-NNN` ID from
  `known-gaps.md`.
- **Unknown.** Neither. Capture it in Part 2 under Unknown, in the developer's
  own words.

Getting this split right is the whole point. A Part 2 workaround is **not a
mistake** — the developer responded correctly to a missing capability. Never
phrase it as something they did wrong, and never recommend a fix that does not
exist. Be generous with the Unknown bucket rather than forcing a poor match onto
an existing gap ID.

### 7. Webhook migration pass

For each polled resource, check `webhook-events.md` for coverage.

**If an event exists**, write a migration recipe. Lead with the strongest true
argument: the payload usually already contains what they are re-fetching. For
order-family events the payload is a v2 order already expanded with `items`,
`payments`, `subscriptions`, `paymentplans`, `checkout` and `checkoutview` — a
superset of a bare `GET /v2/orders/{id}`. A receiver typically needs zero
follow-up calls.

Always pair the recommendation with the reconciliation caveat: failed deliveries
can be retried, but events that were never queued to the endpoint cannot be
backfilled (SPF-GAP-008), so keep a low-frequency `updated_at.gte` sweep as a
safety net. Omitting this gets them burned during an outage.

**If no event exists**, say so and cite the gap ID. Do not invent an event name.

### 8. Write the report

Follow `references/report-template.md` exactly. The output file must be named
`spiffy-integration-diagnostics.md`.

### 9. Redact

Before showing the report: remove API keys, tokens, customer PII, internal
hostnames, and proprietary business logic. Keep file paths, endpoint shapes, and
volumes — those carry the diagnostic value and are safe.

Then tell the user where the file is, and that they can send it to
`support@spiffy.co` if they want Spiffy's input on the gaps it surfaced. Frame it
as their choice. Do not send anything yourself.

## Red flags — stop if you catch yourself

| Thought | Reality |
|---|---|
| "Most APIs support sparse fieldsets, so I'll suggest `?fields=`" | It's reserved and unimplemented. Check the catalog. |
| "There must be a `customer:updated` event" | There isn't. Absence of an event is a Part 2 finding. |
| "I'll estimate roughly 10k calls/month" | Show the arithmetic or don't state the number. |
| "This looks inefficient, I'll flag it" | Without `file:line` evidence it goes under Needs confirmation. |
| "The report seems thin, I'll add some suggestions" | A clean integration deserves a short report. Padding is a failure. |
| "The daily full sweep is wasteful" | It's the correct mitigation for no webhook replay. Never recommend removing it. |
| "I can't read the references but I know this API" | You don't. Stop and say so. |
