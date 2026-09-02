# Report template

Fill this in and save it as **`spiffy-integration-diagnostics.md`**. Use that
filename exactly — Spiffy recognises it.

Replace every `<placeholder>`. Delete any section that genuinely has no content
and say so in one line rather than padding it.

---

```markdown
# Spiffy integration diagnostics — <integration or company name>

**Generated:** <date>
**Audit mode:** <Full | Interview-only>
**Confidence:** <High | Medium | Low — pick the nearest; the sentence carries the
nuance> — <one line on why, naming the weakest input the numbers rest on>

Volume figures are estimates derived from source code and from answers given
during the audit, not from measured traffic. Each carries a tag: `measured`
(from configuration in the project, or from a live response header), `stated`
(from the developer), or `assumed` (a default).

## Snapshot

**What this integration does:** <one or two sentences, in the developer's words>
**Scale:** <N merchants/accounts, N orders, N customers, N subscriptions>
**Traffic:** <page views / requests / sessions per day, if any call site is
traffic-driven — and whether it's growing. Say "no traffic-driven call sites" if
there are none.>
**Plan tier:** <tier> — <monthly quota> calls/month
**Measured usage:** <n> calls this month from `X-Monthly-RateLimit-Remaining` or
the Spiffy settings page, or "not available — figures below are estimates">
**Stack:** <language, framework, scheduler>
**Webhooks configured today:** <yes/no — which events, or what stopped them>

## Where the calls go

| Endpoint | Driver | Trigger / multiplier | Est. calls/month | % of total | Evidence |
|---|---|---|---|---|---|
| `GET /v2/orders` | scheduled | cron `*/5 * * * *` | 8,640 `measured` | 24% | `sync.js:14` |
| `GET /v2/customers/:id` | traffic | 3 × 1,200 page views/day | 108,000 `stated` | 75% | `middleware/auth.js:22` |
| <endpoint> | <driver> | <multiplier> | <n> `<tag>` | <n>% | `<file:line>` |

**Total estimated:** <n> calls/month

<If a measured figure is available and differs materially from this total, say so
here and say which is larger. A measured total well above the estimate means
something outside this codebase is drawing on the same account quota — name it as
a finding rather than leaving the discrepancy unexplained.>

## Call budget

| | Estimated | Ceiling | Headroom |
|---|---|---|---|
| Monthly | <n> | <quota> | <n> (<n>%) |
| Peak burst | <n>/min | 100/min | <n> |

<What this means, in a sentence or two. If over quota, lead with it and say what
actually happens at runtime. Note that both limits are per account, not per API
key — other integrations, internal tools and MCP connector usage on the same
merchant share them.>

<If demand exceeds the monthly allowance, give the days-to-exhaustion figure:
`allowance ÷ (calls per occurrence × occurrences per day)`. It describes the
failure the developer actually experiences better than a percentage over.>

## What you can change

Ranked by impact. Usually that means call savings; if the integration is already
efficient, rank by risk and say so.

Everything here is something an existing Spiffy capability already solves. Things
that are blocked, or that don't exist, are in the feedback section instead.

### 1. <Finding title>

**Where:** `<file:line>`
**Costs:** ~<n> calls/month (<n>% of total)
**What's happening:** <plain description>
**Why it's built this way:** <the developer's reason, or "not established">
**Change:** <specific parameter or endpoint>
**Saves:** ~<n> calls/month, or "not quantifiable from source — <what would settle it>"
**Reference:** <link to the relevant developers.spiffy.co page>

### 2. <...>

## Security

Spiffy-specific credential exposure and payload trust only. This is not a general
security review.

### <Finding title>

**Where:** `<file:line>`
**What's exposed:** <what an attacker could reach>
**Fix:** <specific action, including rotation if a credential has leaked>

<If there are no findings, write one line saying so and delete the rest.>

## Feedback for Spiffy

Not implementation mistakes. These are the places where Spiffy's API or
documentation shaped what got built.

### Where we got stuck

Things Spiffy supports that this integration couldn't successfully use.

#### <Short title>

**What they were trying to do:** <in the developer's own words>
**What stopped them:** <the blocker — an error, a gap in the docs, an environment
the documented approach didn't cover, something that silently didn't work>
**What they built instead:** <the workaround>
**Costs:** ~<n> calls/month
**Where:** `<file:line>`

### What doesn't exist

Capabilities this integration needed and Spiffy doesn't offer.

#### <Short title>

**What they needed:** <the capability, described in the context of this integration>
**What they built instead:** <the workaround>
**Costs:** ~<n> calls/month
**What they'd do if it existed:** <the call pattern they'd prefer>
**Where:** `<file:line>`

## Needs confirmation

Findings without direct code evidence, or that depend on an assumption worth
checking:

- <finding> — <what would confirm it>

## Appendix — method and assumptions

**How volumes were estimated:** <the arithmetic, per major call site>
**Assumptions made:** <each one, and what it would change if wrong>
**Could not determine:** <anything the audit couldn't establish>
**Catalog version:** <generation date of the reference files used>

If any capability described in this report doesn't match the current API, the
reference catalog may be out of date — please tell Spiffy, since that's a bug in
the diagnostic rather than in the integration.

---

## Shareable summary

Everything above is yours and stays local. This last section is the only part
written to be handed to Spiffy — it describes what was missing on their side and
contains nothing about your code, so it's safe to share as-is. Nothing is sent
anywhere automatically.

```
SPIFFY INTEGRATION FEEDBACK

Integration type: <generic shape — e.g. "warehouse sync", "fulfilment listener",
                  "customer portal". No company or product names.>
Auth model:      <API key | OAuth app>
Merchants:       <1 | a handful | tens | hundreds+>
Scale:           <order of magnitude only — e.g. "tens of thousands of orders">
Plan tier:       <tier>
Est. usage:      <n> calls/month against a <quota> quota (<n>%)

WHAT WE COULDN'T GET WORKING
<For each: what they were trying to do, what stopped them, what they built
instead, and roughly what it costs. This is the most useful part — it is where
Spiffy supports something and the developer couldn't get there.>

WHAT DIDN'T EXIST
<For each: the capability needed, the workaround built, its rough cost, and what
they'd do instead if it existed.>

WHAT WOULD HAVE HELPED MOST
<One or two sentences, in the developer's own words.>
```
```

---

## Notes for whoever runs this

**Call volume is the spine.** The budget and the change list come first. Security
is a secondary pass and shouldn't crowd them out.

**The feedback section is the reason this exists**, so get the split right:

- *Where we got stuck* means the capability exists and something prevented its
  use. This is the most valuable and most missed category — and the cheapest for
  Spiffy to fix, because it's usually documentation rather than API surface.
- *What doesn't exist* means no capability achieves it. Describe it fresh, in the
  context of this integration. Don't reach for a generic label, and don't
  recommend a fix that isn't real.

**Attribute shared cost once.** Several limitations often justify the same piece
of work — a reconciliation sweep can be caused by more than one at a time. Assign
the cost to the one that best explains it and cross-reference from the others.
Counting the same calls repeatedly inflates the totals.

**But don't leave the root cause with no number.** When a blocker explains an
entire architecture — "we couldn't verify signatures, so we poll everything" —
state the traffic it accounts for as **explains: ~n calls/month**, distinct from
the **costs:** line used for attribution. Otherwise the most important finding in
the report is the only one without a figure attached, and it reads as the least
important.

**Savings compound, they don't add.** Fixing an N+1 and raising `per_page` on the
same loop don't sum — the second applies to what's left after the first. Say the
savings are sequential, and give a single post-fix total rather than inviting the
reader to add up the individual lines.

**If a section is empty, say so.** An integration with no feedback findings is a
real and good outcome. Inventing one to fill the section destroys the value of
comparing these reports.

**Length is not a proxy for thoroughness.** A well-built integration deserves a
short report.

## Constructing the shareable summary

The developer has to be able to send this without reading it line by line
first. That property comes from how it's built, not from a promise.

**It must not contain:** file paths, code, function or module names, endpoint
inventories, company or product names, customer data, credentials, internal
hostnames, or anything about their business logic or architecture.

**Exact record counts become magnitude buckets.** "~40,000 orders" can imply
revenue; "tens of thousands of orders" carries the same diagnostic weight
without the disclosure. Call volumes and quota percentages are fine as stated —
those describe Spiffy's API, not their business.

**Their own words are the most valuable content in it**, particularly on what
blocked them. Quote rather than paraphrase, but strip anything identifying.

**If there is nothing to report, say so and leave the block out.** An empty
summary is a good outcome, not a gap to fill.

**Security findings never go in it.** Those are the developer's own exposure, not
feedback about Spiffy, and a summary that names their vulnerabilities is not safe
to share. Keep them in the private report.

**Naming a Spiffy endpoint is fine; naming their code is not.** "The payments
list has no `updated_at` filter" describes Spiffy's API and belongs in the
summary. "`sync.js:61` calls it" describes their codebase and does not. The
prohibition is on their architecture, not on ours.

**Never submit it.** Produce it, tell the developer it's there, and stop.
