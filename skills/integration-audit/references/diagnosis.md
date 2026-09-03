# Diagnosis method

How to work out what an integration is doing, what it costs, and — the part that
matters — **why it was built that way**.

## The principle

A costly call pattern is a symptom. It is never, on its own, a finding.

The same nightly full resync can be any of three completely different things: a
developer who didn't know `updated_at.gte` exists, a developer who tried it and
hit something that didn't work, or a developer who checked and found the resource
genuinely has no delta filter. Those need three different responses — one is a
code change, one is our documentation failing, one is our API failing.

You cannot tell them apart by looking at the code. **You have to ask.**

So: find what costs, establish what it was for, ask why that approach, and only
then decide who owns the fix. Never skip to the verdict because the pattern looks
familiar.

## Step 1 — Verify what the code believes about the API

Do this before anything else, because these are wrong in ways the developer
cannot see. They are not style problems; they are correctness bugs that also cost
money.

**Check every filter key against the spec**, for that exact resource path — a
key absent from that path's `parameters` array does not work. An unrecognized key is dropped
silently and the request returns 200 with the full unfiltered list. Nothing in
the response, the status code, or the developer's logs indicates the filter did
nothing. A key that looks obviously right — `state` for `status`, or a field that
exists on the object but isn't filterable — will sail through review.

**Check every webhook event name** against `GET /v2/webhook-event-types`. A
subscription to an event that doesn't exist simply never fires.

**Check every `include=`, `search=`, and `sort=` value** the same way. All are
silently ignored where unsupported.

Anything failing these checks is a finding regardless of its estimated volume,
because the estimate is a floor — you are measuring what the code *thinks* it
fetches, not what it actually fetches.

## Step 2 — Find where the cost is

### Measure before you estimate

Every response carries `X-Monthly-RateLimit-Remaining`. Two readings an hour
apart give the account's real consumption rate — no arithmetic, no assumptions,
and it includes traffic this codebase cannot see: other integrations, dashboard
activity, MCP tool calls, non-production environments on the same key. The
merchant's Spiffy settings show the same figure with a projection.

Ask for it early. An estimate from source is a *floor* — it describes what this
code intends to send, not what the account spends. When the measured total is
much larger than the sum of the call sites you found, the gap is the finding,
and further optimisation of this repository may be aimed at the wrong target
entirely.

### Name the multiplier for each call site

Cost is `calls per occurrence x occurrences per day x 30`. An occurrence is
whatever makes the call happen again, and there are five kinds — scheduled,
traffic, data, event, fixed. `usage-patterns.md` has the taxonomy and the
patterns that go with each.

Get this right before costing anything, because it is where the order of
magnitude lives. A scheduled call site's multiplier is written in the repository
and is bounded. A traffic-driven one lives in the developer's analytics, is
invisible in the code, and rises with their success — which is why an
integration that was comfortable for a year fails in a week, and why the
accounts that exhaust a quota in days almost always have one.

If you cannot find a multiplier in the code, that is a question for the
interview, not a number to invent.

### Then rank

Build the call budget, rank by share of total, and work down from the top.

Concentrate on what dominates. An integration's traffic is usually one or two
call sites and a long tail of irrelevance; a pure-efficiency finding that saves
0.1% is noise even when it's correct. Say so rather than padding the report.

**That threshold applies to efficiency only.** A correctness bug — a filter that
silently does nothing, a delta that misses rows, a sync that is quietly wrong —
is a finding regardless of what it costs, because the developer is acting on bad
data and doesn't know it. When the two rules disagree, correctness wins. Rank by
volume, but say plainly that the severity is correctness rather than cost.

### Some per-occurrence calls cannot be reduced

Before writing up any call that runs once per user, session or request, apply
this test:

> **A per-occurrence call is only a finding if the same answer would serve more
> than one occurrence.**

Minting a portal SSO token is per user, single-use and short-lived by design —
it cannot be cached, cannot be shared, and caching it would be a security bug. A
write the user just requested is the same. For calls like these the finding, if
any, is capacity rather than design: size it, say plainly that it is
irreducible, and move on. Recommending a fix for something the developer knows
is unavoidable costs you the credibility you need for the findings that matter.

The middle case is the valuable one. A read that is irreducible *per session*
but is happening *per request* is a real finding, and the fix is to change its
scope rather than remove it.

## Step 3 — Establish intent

For each call site that matters, answer: **what is this trying to achieve?**

Get as far as you can from the code — what it fetches, what it keeps, what it
writes, what triggers it. A loop that discards everything but two fields wants a
change feed. A nightly job writing to a warehouse wants a mirror. A per-request
fetch inside a page handler wants freshness.

Then ask the developer for what the code can't tell you. Aim the question at the
decision, not the mechanics:

> You're pulling all orders every five minutes. What does that feed, and how
> fresh does it actually need to be?

## Step 4 — Ask why this approach

This is the step that produces the findings worth having, and it is the one
easiest to skip once you think you know the answer.

For each significant call site, work out which of these is true — from the code
where it's unambiguous, from the developer where it isn't:

- **They didn't know there was another way.** No sign they considered one.
- **They tried and it didn't work.** Look for commented-out code, an abandoned
  webhook route, a workaround with a note attached, an unusual detail that only
  makes sense as a response to something. Ask directly: "did you look at X?"
- **They knew and chose this deliberately.** There's usually a constraint you
  can't see — a compliance rule, a platform they're hosted on, an internal
  system that can't accept inbound requests.
- **There was no other way.** Confirm against the capability catalogs before you
  accept this, and confirm it *is* absent rather than merely undocumented.

When a developer has abandoned an approach, find out what stopped them. That
answer is consistently the most valuable output of the entire audit, and it is
never visible in the code — the code shows what they built, not what they gave up
on.

## Step 5 — Classify by who owns the fix

Every finding lands in exactly one of three places.

### They didn't use what exists

A capability in the catalogs achieves their intent and they aren't using it.

Goes in the report's fix section, with the specific parameter or endpoint and a
link to the relevant page on developers.spiffy.co. This is the only category
where you tell someone to change their code.

### They couldn't successfully use what exists

The capability exists, and they tried, or would have, but something stopped them
— docs that didn't cover their situation, an error they couldn't interpret, an
environment the documented approach didn't account for, an implementation that
silently didn't work.

Goes in the feedback section, not the fix section. **Do not just tell them to go
use the thing they already failed to use.** Record what they were trying to do,
what they hit, and what they built instead. If you can see how to get them past
it, say so — but the blocker itself is the finding.

This category is easy to miss and cheap for Spiffy to fix, which makes it the
highest-value thing you can surface.

### It doesn't exist

Nothing in the catalogs achieves their intent.

Goes in the feedback section. Describe the limitation in the context of *this*
integration: what they needed, what they built instead, what it costs them, and
what they'd have done if it existed. Write it fresh — do not reach for a generic
label.

Be direct about it. A developer who wrote a 500-call pagination loop, a full
resync, or a client-side join because we gave them nothing better did the right
thing with what they had. Don't imply otherwise, don't soften it, and never
recommend a fix that doesn't exist.

## Signals worth looking at

**`usage-patterns.md` is the catalog**, organised by what multiplies each call.
Work through it once the multipliers are named. It is a set of starting points,
not a checklist: a pattern there tells you where to *look*, and steps 3 to 5 tell
you what it means. Something not in it is not thereby fine, and something in it
is not thereby a finding.

Two shapes deserve attention beyond their cost, because neither announces
itself:

- **A Spiffy call on a path the developer's own users trigger** — a request
  handler, a page render, middleware, a component mount, a browser interval.
  These are the ones that exhaust a quota in days, and they look small in the
  code because the multiplier is nowhere near them.
- **Anything with a comment explaining why it's odd.** Read the comment, then
  ask. It is usually a compressed account of something that did not work, which
  is exactly what step 4 is trying to recover.

## Measure against the recommended shape

`webhook-events.md` describes how a well-built integration keeps a local copy in
sync: mirror from the payload, dedupe on the envelope id, don't trust ordering,
acknowledge fast and own the retry after, keep a delta sweep for the backfill
gap, and store Spiffy ids rather than email as join keys.

Use it as the reference architecture. Where an integration diverges, work out
which of the three categories the divergence falls into rather than assuming they
got it wrong — plenty of divergence is a reasonable response to a constraint you
haven't found yet.

Divergence that is *always* worth reporting, whatever the reason:

- **Joining on email**, because the failure mode is mis-linking one customer to
  another's data.
- **Non-idempotent webhook writes**, because at-least-once delivery guarantees
  they will eventually double-apply.
- **No reconciliation path at all**, because there is no backfill and the data
  loss is permanent and silent.

## Rules

- **Never conclude from the pattern alone.** The same code is a mistake, a
  workaround, or the only option, and only the reason tells you which.
- **Never recommend a capability absent from the catalogs.** If their intent has
  no supported route, that is the finding.
- **Don't stop at the first plausible explanation.** A cheap match ends the
  inquiry that would have produced the useful answer.
- **Don't scold.** Every category except the first is about our gaps, not theirs.
- **Say when there's nothing to find.** A well-built integration produces a short
  report. Manufacturing findings to fill it is a failure.
