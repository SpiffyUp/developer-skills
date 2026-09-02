# Spiffy integration audit

Works out where your Spiffy API calls actually go, what they cost against your
rate limit and monthly quota, and what you can cut.

If your quota disappears in the first few days of the month, this is the thing
to run. That pattern almost always means a call sits somewhere your own users
trigger — a page load, a session, a component render — so its cost scales with
your traffic rather than your data, and nothing in your scheduler config shows
it.

Run it in whatever AI coding tool you already use — Claude Code, Cursor, ChatGPT,
or anything else that can read your project.

## Run it

Paste this into your AI tool, from inside your project:

```
Read https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/SKILL.md
and follow it to audit how this project integrates with Spiffy.
```

It reads your integration code, asks a few questions it can't answer from the
code, and writes `spiffy-integration-diagnostics.md`.

## What you get

**A call budget.** Where your requests go, ranked, with the arithmetic shown,
measured against your plan's monthly quota and the per-minute limit. This is the
core of it — most integrations cost far more than they need to, and usually it's
one or two call sites doing nearly all of it.

Each call site is costed by what actually multiplies it: a schedule, your
traffic, your record count, or your order volume. Traffic-driven calls are the
ones that exhaust a month's quota in days, and they're the easiest to miss —
the multiplier lives in your analytics, not in your code. Where demand is over
the ceiling you get a days-to-exhaustion figure rather than a percentage.

Calls that genuinely have to happen per user — minting a portal SSO token, a
write someone just asked for — are sized and left alone. They can't be cached
and the audit won't pretend otherwise.

**Changes worth making.** Concrete fixes using capabilities that already exist:
pagination you're leaving on the table, `include=` that collapses an N+1,
`updated_at.gte` instead of a full resync, polling that a webhook would replace,
and filter keys that are silently doing nothing.

That last one is worth running the audit for on its own. An unrecognized filter
key doesn't error — the request returns 200 with the whole unfiltered list. The
code looks right, the logs look right, and you're paging your entire table.

It also checks polling loops against the current event catalog. Customers,
products and promos emit create/update/delete events now, and integrations
written before that shipped are often still polling resources that would push to
them instead.

**A short security pass** covering Spiffy-specific exposure only: API keys
reaching client code, webhook signature verification, secret handling. Spiffy API
keys carry all scopes and can't be scoped down, so an exposed one is full account
access. This is not a general security review and doesn't pretend to be.

**Feedback for us.** Where our API or our docs got in your way — either something
we support that you couldn't get working, or something we don't support at all.
The audit asks *why* you built things the way you did, because that's the part we
can't see. We can tell an account makes 40,000 calls a month; we can't tell that
most of them exist because an event we don't emit forced you to poll.

## Privacy

The skill reads your code and writes a file. It doesn't modify your code, and it
doesn't send anything anywhere — no API calls, no uploads, no telemetry. Its only
outputs are a file on your disk and what it tells you.

The report is yours. It keeps file paths, endpoint shapes and volumes, because
that's what makes it useful to you.

The last section is a **shareable summary**, and it's the only part written to be
handed to us. It's built so you can send it without auditing it first: no file
paths, no code, no company names, no architecture, and record counts as rough
magnitudes rather than exact figures. What it does contain is what we were
missing and what you had to build instead.

It's safe to share as-is. What you do with it is entirely your call.

## A note on accuracy

Volume figures are estimated from your source and your answers, not measured from
real traffic. Every number is tagged `measured`, `stated`, or `assumed`.

The skill works from a fixed catalog of what the v2 API supports and is instructed
not to invent capabilities that aren't in it. If it tells you something that
doesn't match the current API, that's a bug on our side — let us know.

## Docs

- [API reference](https://developers.spiffy.co/api/getting-started)
- [Pagination](https://developers.spiffy.co/api/getting-started/pagination)
- [Filtering](https://developers.spiffy.co/api/getting-started/filtering)
- [Resource expansion](https://developers.spiffy.co/api/getting-started/resource-expansion)
- [Rate limiting](https://developers.spiffy.co/api/getting-started/rate-limiting)
- [Webhooks](https://developers.spiffy.co/api/webhooks)
