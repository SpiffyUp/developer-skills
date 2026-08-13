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
**Confidence:** <High | Medium | Low> — <one line on why>

Volume figures are estimates derived from source code and from answers given
during the audit, not from measured traffic. Each carries a tag: `measured`
(from code), `stated` (from the developer), or `assumed` (a default).

## Snapshot

**What this integration does:** <one or two sentences, in the developer's words>
**Scale:** <N merchants/accounts, N orders, N customers, N subscriptions>
**Plan tier:** <tier> — <monthly quota> calls/month
**Stack:** <language, framework, scheduler>
**Webhooks configured today:** <yes/no — which events, or why not>

## Where the calls go

| Endpoint | Trigger | Est. calls/month | % of total | Evidence |
|---|---|---|---|---|
| `GET /v2/orders` | cron `*/5 * * * *` | 8,640 `measured` | 71% | `sync.js:14` |
| <endpoint> | <trigger> | <n> `<tag>` | <n>% | `<file:line>` |

**Total estimated:** <n> calls/month

## Call budget

| | Estimated | Ceiling | Headroom |
|---|---|---|---|
| Monthly | <n> | <quota> | <n> (<n>%) |
| Peak burst | <n>/min | 100/min | <n> |

<One or two sentences on what this means. If over quota, say so plainly and lead
with it. Note that the per-minute limit is scoped per account, not per API key —
other integrations on the same merchant share this budget.>

## Part 1 — What you can fix

Ranked by impact. Usually that means estimated call savings — but if the
integration is already efficient, rank by risk instead and say so in one line.

**One exception to the ranking:** a silently-ignored filter key (AP-02) goes
first regardless of its estimated volume. Its estimate is always a floor, never a
ceiling, and it is usually also a data-correctness bug rather than only a cost
one. When it outranks a finding with a far larger number, say why in the entry so
a skimming reader isn't confused.

**If a saving cannot be computed, write "not quantifiable from source" and say
what would settle it.** Do not invent a number to fill the field. The `<n>`
placeholder below is a slot, not an obligation.

**If there is nothing to fix, write that and move on.** A well-built integration
producing a three-line Part 1 is a correct result, not a thin one. Do not
manufacture a finding to fill this section, and do not pad a marginal one to make
the ranking look substantial. An empty Part 1 with a clear budget above it is
more useful to the reader than five findings that save nothing.

### 1. <Finding title>

**Where:** `<file:line>`
**Costs:** ~<n> calls/month (<n>% of total)
**What's happening:** <plain description>
**Fix:** <specific change, with the exact param or endpoint>
**Saves:** ~<n> calls/month
**Reference:** <link to the relevant Spiffy docs page>

### 2. <...>

## Part 2 — Feedback for Spiffy

These are limitations the integration had to work around. They are not
implementation mistakes.

**Attribute shared cost once.** Several gaps often justify the same piece of
work — a reconciliation sweep can be caused by the backfill gap, the delete-event
gap and the bulk-export gap at the same time. Assign the call cost to the single
gap that best explains it, and cross-reference from the others with "see
SPF-GAP-NNN" rather than repeating the figure. Counting the same calls three
times inflates the totals Spiffy aggregates across reports, which is the one
thing Part 2 exists to get right.

### <SPF-GAP-NNN> — <gap title>

**Costs this integration:** ~<n> calls/month
**Workaround in use:** <what they built instead>
**What they'd do if it existed:** <the call pattern they'd prefer>
**Where:** `<file:line>`

### Unknown — <short title>

**What they were trying to do:** <in the developer's own words>
**Workaround built:** <description>
**Costs:** ~<n> calls/month
**Why no existing gap fits:** <one line>

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
```

---

## Notes for whoever runs this

- If Part 2 is empty, say so. An integration with no gap findings is a real and
  good outcome; inventing one to fill the section destroys the value of
  aggregating these reports.
- If the integration is in good shape, the report should be short. Length is not
  a proxy for thoroughness.
- Keep the gap IDs exactly as written in `known-gaps.md`. Aggregation across
  submissions depends on them.
