---
name: auditing-spiffy-integration
description: Use when a developer wants to review, diagnose, or optimize how their application integrates with Spiffy — including questions about API call volume, exhausting the monthly quota or hitting rate limits, 429s and monthly_limit_exceeded errors, calls made per page view or per user visit, whether to use webhooks instead of polling, or when Spiffy has asked them to run an integration diagnostic.
---

# Auditing a Spiffy integration

You are auditing how this project integrates with Spiffy, and producing a single
file — `spiffy-integration-diagnostics.md` — that the developer can act on and,
if they choose, send to Spiffy.

**The core of this audit is API call volume**: where their requests go, what that
costs against their rate limit and monthly quota, and how to bring it down.
That's what most integrations get wrong and what most developers feel as pain.
Keep it the spine of the report.

Cost is `calls per occurrence x occurrences per day x 30`, and the job is to
identify what an *occurrence* is for each call site — a cron tick, a page view,
a record, a received event. Scheduled work is the familiar case and the easiest
to see; it is not the expensive one. **A call on a path the developer's own
users trigger is what exhausts a monthly quota in days**, because its multiplier
is their traffic, it is invisible in the code, and it grows as they succeed. Go
looking for those specifically. A secondary pass covers Spiffy-specific
credential exposure (`security-checks.md`), which is tightly scoped and must not
expand into a general security review.

The report has two halves. One tells the developer what to change. The other
tells Spiffy what got in their way. The second half is the reason this exists:
Spiffy can see that an account makes 40,000 calls a month, but not that most of
them exist because something in their API or their docs failed this developer.

## What this does and does not do

Tell the user this before you start, in your own words:

- It reads their integration source and produces one markdown file.
- It does **not** modify their code.
- It does **not** transmit anything anywhere. Sending the report to Spiffy is
  their explicit choice at the end, and the report gets a redaction pass first.

They are running a vendor-supplied tool against a private codebase. Their
suspicion is reasonable — address it up front rather than waiting to be asked.

## Before you start — establish your mode

1. **The live API reference is the source of truth for the API surface.** Read
   what you need from it as you go:

   - Spec: `https://api.spiffy.co/openapi.json` — large, so fetch the paths you
     are auditing rather than the whole document. A path's `parameters` array is
     the authoritative list of its filters and expansions.
   - Rendered guide: `https://developers.spiffy.co/api/getting-started`
   - Webhook events: `GET /v2/webhook-event-types`, no auth required.

2. Load all six reference files. If this skill is installed locally they are in
   its `references/` directory. If you reached this file over HTTP, fetch them
   from the same location:

   - `https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/references/api-capabilities.md`
   - `https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/references/webhook-events.md`
   - `https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/references/usage-patterns.md`
   - `https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/references/diagnosis.md`
   - `https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/references/security-checks.md`
   - `https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/references/report-template.md`

   They carry method and behaviour, not API surface. If you can neither read nor
   fetch them, ask the user to paste them. Do not continue without them.
3. Try to read the project's source.
4. Set your mode:

   - **Full** — references loaded and source readable.
   - **Interview-only** — references loaded, source unreadable. Every volume
     figure comes from the user and is tagged `stated`.
   - **Blocked** — you can reach neither the live spec nor the references. Say so
     and stop.

**In Blocked mode, stop.** Do not proceed from memory. You do not know Spiffy's
filter keys, event names, or limits, and the failure mode is not a vague answer —
it is a confident, specific, wrong one.

State your mode to the user before continuing, and record it in the report.

## Hard rules

- **Never claim a capability you have not just read in the spec.** A parameter
  absent from a path's `parameters` array does not work — it is not merely
  undocumented. Check the exact endpoint; do not generalise from another
  resource, from REST convention, or from memory.
- **Never recommend a webhook event absent from `GET /v2/webhook-event-types`.**
  Call it. A subscription to an event that does not exist is accepted and never
  fires.
- Never assert a call volume without showing its inputs and arithmetic.
- Every figure carries a confidence tag: `measured`, `stated`, or `assumed`.
- A finding without `file:line` evidence goes under "Needs confirmation", not
  among the findings.
- If you could not check a resource's filter or `include=` support against the
  spec, write "not verified" rather than inferring it.
- **Never conclude from a code pattern alone why it was written.** Establish the
  reason before assigning a fix to anyone.
- **Never flag an irreducible per-occurrence call.** A call is only a finding if
  the same answer would serve more than one occurrence. Minting a portal SSO or
  magic-link token is per user, single-use and short-lived by design; caching it
  would be a security bug. A write the user just requested is the same. Size
  these, say plainly that they cannot be reduced, and move on. Reducing their
  *frequency* — one read per session rather than per request — is a real finding;
  removing them is not.
- Never recommend removing a reconciliation sweep. Spiffy can retry an event
  already queued to an endpoint, but cannot backfill one that was never queued,
  so a periodic delta sweep is correct design, not waste. Recommending a
  *different cadence* is fine — that's tuning, not removal.

Violating the letter of these rules is violating their spirit. If you catch
yourself reasoning "this capability is standard, so it probably exists" — stop.
That is the exact failure these rules exist to prevent. REST convention is not
evidence about this API.

## Procedure

Read `references/diagnosis.md` first. It carries the method; this is the running
order.

### 1. Scope and announce

Confirm this project integrates with the Spiffy API. Say what you're about to do
and what you won't do.

Ask now for the account's **measured** usage, because it takes them a minute and
it reframes everything after it: the `X-Monthly-RateLimit-Remaining` header on
any response, read twice an hour apart, or the usage figure in their Spiffy
settings. It covers consumers this codebase cannot see. If they can't get it,
carry on with estimates and say so in the report.

### 2. Inventory the call sites

Search the project for `api.spiffy.co`, `/v2/`, `Spiffy-Signature`, webhook route
handlers, and any wrapper or SDK the project uses around Spiffy calls. Follow the
wrapper to its call sites rather than stopping at its definition.

For each call site record: `file:line`, method, endpoint, the query params
actually passed, and what triggers it — cron, queue, request-scoped, webhook
handler, or manual.

### 3. Verify what the code believes about the API

Step 1 of `diagnosis.md`, and it comes before everything else. Check every filter
key, event name, `include=`, `search=` and `sort=` value against the live
spec and the event-types endpoint.
These fail silently and return 200, so the developer has no way to notice.

### 4. Determine what multiplies each call site

Not just cadence. Each call site is multiplied by one of five things, and
naming it wrong changes the budget by orders of magnitude. `usage-patterns.md`
carries the taxonomy:

| Driver | Multiplier | Found in |
|---|---|---|
| **Scheduled** | runs/day | crontab, `setInterval`, queue or workflow config |
| **Traffic** | page views, requests, sessions or users/day | the call sits in a request handler, render path, middleware or component |
| **Data** | records or pages | a loop over a result set, or pagination |
| **Event** | orders/payments/signups per day | a webhook handler or queue consumer |
| **Fixed** | once, or a handful | boot, deploy, manual, one-off backfill |

**A traffic multiplier is never in the code.** You can establish from source
that a call site is traffic-driven — that is structural and visible — but the
number itself has to come from the developer. Identify these on the first pass
and carry them into the interview; they are the reason accounts die on day
three, and an audit that budgets only the schedulers will conclude a dying
integration is comfortably inside its quota.

Evidence comes in three strengths, and the whole budget scales off it:

- **Config present in the project** — a crontab, workflow file, queue
  definition. Tag `measured`.
- **A comment that names where it lives** ("cron `0 3 * * *`, see
  `ops/crontab`"). Tag `measured`, and add a needs-confirmation entry pointing at
  that file. The author told you where to check.
- **A bare comment with no config anywhere**, or a traffic figure you were not
  given. Tag `assumed` and say what would settle it.

Don't flatten these. Marking a well-evidenced multiplier `assumed` makes the
whole report read as shakier than it is, which costs you credibility on the
findings that matter.

### 5. Build the call budget

For each call site: `calls per occurrence × occurrences per month = calls per
month`. Show the inputs, and name the driver.

> Nightly, ~12,000 orders at `per_page=50` = 240 pages + 1 = 241 calls/run × 30
> = **7,230 calls/month** [scheduled; `measured` cadence, `stated` record count]

> 3 calls per page view × 1,200 views/day × 30 = **108,000 calls/month**
> [traffic; `measured` call count, `stated` traffic]

Where a traffic-driven site is over the ceiling, the reciprocal is the number
that lands: `allowance ÷ (calls per occurrence × occurrences per day)` = **days
to exhaustion**. A developer whose access dies every month recognises that figure
immediately, and it explains the shape of the problem better than a percentage.

Then total, rank by share of total, and compare against both ceilings:

- The **monthly quota** for their plan (see `api-capabilities.md` — note
  Business is 20,000 and sits *below* Scale).
- Peak **burst** against 100 requests per 60 seconds — which is scoped per
  account, not per key, so other integrations on the same merchant share it.

**When estimated demand exceeds the ceiling by a wide margin**, the percentage
columns stop describing reality — the account exhausts its quota early and
everything after that is a 429. Report the demand figure, then explain separately
what actually happens at runtime.

### 6. Interview

Ask about the call sites that dominate the budget, and about anything you found
in step 3. Keep it to a handful of questions, in one batch, and skip anything the
code already answered.

Always establish:

1. What the integration is for, in a sentence or two.
2. How many Spiffy accounts or merchants it serves, and roughly how many records
   are in the resources it syncs.
3. Their plan, and their measured usage if they were able to get it.
4. **The traffic figure for every traffic-driven call site you found** — page
   views, requests, sessions or active users per day, whichever matches how the
   call is triggered. Ask for a rough daily number and whether it is growing.
   You cannot get this from the code and the budget is meaningless without it,
   so do not let it drop off the list.
5. **What else uses the account's quota** — other integrations, internal tools,
   non-production environments on the same key, MCP connector usage. Both limits
   are per account, not per key, so anything else installed competes with this
   integration. Ask explicitly if their measured total exceeds what you found.

Then, for each dominant call site, ask the two questions that matter:

6. **What does this feed, and how fresh does it actually need to be?**
7. **Why this approach?** — did they consider an alternative, try one and hit a
   problem, or find there wasn't one?

And if they aren't using webhooks for something a webhook covers, ask directly
what stopped them. Do not treat the answer as small talk. "We couldn't verify
signatures behind our proxy" is a more valuable finding than anything you will
derive from the code.

### 7. Classify every finding

Step 5 of `diagnosis.md`. Each finding lands in exactly one of:

- **They didn't use what exists** → the fix section.
- **They couldn't successfully use what exists** → the feedback section. Do not
  simply tell them to use the thing they already failed to use.
- **It doesn't exist** → the feedback section, described fresh in the context of
  this integration.

Only the first category tells someone to change their code.

**Classify the issue, not the call site.** One call site routinely produces more
than one finding in more than one category — a customer poll can be both "no
event covers this, so polling is unavoidable" *and* "the poll itself is missing
`updated_at.gte`". Split those into separate findings and cross-reference them.
"Exactly one" constrains each finding, not each piece of code.

**When the API let them get it wrong, say both parts.** An unrecognized filter
key has a fix (use the real key) *and* a reason it survived to production (the
request returned 200 with the full list and gave them no way to notice). Put the
fix in the fix section and the silent failure in the feedback section. Reporting
only the fix hides the part Spiffy needs to hear.

### 8. Webhook migration pass

For each polled resource, check `GET /v2/webhook-event-types` for coverage, and
`webhook-events.md` for what the payload will contain.

Customers, products and promos have a full `created`/`updated`/`deleted` set,
and affiliates have `registered` and `updated`. These are recent, so a poll
against one of them was very likely correct when it was written — check whether
it predates the events and say so in the finding rather than writing it up as an
oversight.

**If an event exists**, write a migration recipe. Lead with the strongest true
argument: the payload usually already contains what they are re-fetching. For
order-family events the payload is a v2 order already expanded with `items`,
`payments`, `subscriptions`, `paymentplans`, `checkout` and `checkoutview` — a
superset of a bare `GET /v2/orders/{id}`. A receiver typically needs zero
follow-up calls.

Always pair the recommendation with the reconciliation caveat: failed deliveries
can be retried, but events never queued to the endpoint cannot be backfilled, so
keep a low-frequency `updated_at.gte` sweep as a safety net.

**If no event exists**, say so plainly. That is a feedback-section finding, not a
recommendation. Do not invent an event name.

### 9. Spiffy-specific security pass

Run the checks in `security-checks.md`. That file is the complete scope — Spiffy
credential exposure and payload trust. Do not extend it into general application
security; if you notice something serious outside it, give it one line and move
on.

Two of these carry real weight and are worth stating plainly when found: an API
key reachable from a client is full account access, because Spiffy keys carry all
scopes with no per-key scoping; and a webhook handler that doesn't verify the
signature will act on anything posted to a URL that isn't secret.

If there are no findings, say so in a line. Don't invent any.

### 10. Write the report

Follow `references/report-template.md`. Name the output
`spiffy-integration-diagnostics.md` unless the user asks for a different path, in
which case use theirs and note the intended name at the top of the file.

### 11. Redact, then build the shareable summary

Two different jobs. Don't conflate them.

**The report** is theirs and stays local. Strip credentials, customer PII and
internal hostnames from it, but keep file paths, endpoint shapes and volumes —
that's what makes it useful to them.

**The shareable summary** is the last section of the report and the only part
meant to leave their machine. Build it so it can be sent without a careful
read-through first: no file paths, no code, no company or product names, no
architecture, and record counts as magnitude buckets rather than exact figures.
Their own words about what blocked them are the most valuable thing in it.

Then tell them the summary is there and that it's safe to share as-is. Don't
prescribe where it goes or push them to send it — whoever asked them to run this
already has a channel.

**Never transmit anything.** Do not call an API, do not open a socket, do not
offer to submit it. Write the file and stop. This skill's only outputs are a
file on their disk and what you say to them.

## Red flags — stop if you catch yourself

| Thought | Reality |
|---|---|
| "Most APIs support sparse fieldsets, so I'll suggest `?fields=`" | It's reserved and unimplemented. Check the spec. |
| "There must be a `subscription:updated` event" | There isn't — only the 14 lifecycle transitions. Absence of an event is a feedback finding. |
| "This is obviously a polling antipattern, I'll write it up" | You don't yet know why they built it. Ask first. |
| "They should just use webhooks" | If they already tried and failed, that's our bug, not their fix. |
| "I'll estimate roughly 10k calls/month" | Show the arithmetic or don't state the number. |
| "This looks inefficient, I'll flag it" | Without `file:line` evidence it goes under Needs confirmation. |
| "The report seems thin, I'll add some suggestions" | A clean integration deserves a short report. Padding is a failure. |
| "The daily full sweep is wasteful" | It's the correct mitigation for no webhook backfill. Never recommend removing it. |
| "I can't read the references but I know this API" | You don't. Stop and say so. |
| "While I'm here, I'll review their auth and dependencies" | Out of scope. `security-checks.md` is the whole list. |
| "The security findings are more interesting than the volume ones" | Call volume is the spine of this report. Keep it first. |
| "No cron anywhere, so the integration is light" | Traffic-driven calls have no schedule. Check the request handlers. |
| "It's one call, it's fine" | One call times their daily traffic is the whole quota. Cost it. |
| "They should cache this" | Their portal SSO token can't be cached. Apply the irreducible test first. |
| "Add caching" (and stop there) | Spiffy sets no ETag or Cache-Control. The cache has to be theirs — say so. |
| "There's no `customer:updated` event" | There is now. Call `/v2/webhook-event-types` instead of recalling it. |
| "The catalog doesn't list this filter" | The catalogs no longer list filters. The spec does. Go read it. |
| "Their numbers don't add up to the measured total" | That gap is a finding. Something else is on the same quota. |
