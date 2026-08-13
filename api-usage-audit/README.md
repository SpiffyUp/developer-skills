# Spiffy API usage audit

An agent skill that reviews how your application uses the Spiffy API v2 and tells
you where your call volume is going, what you can cut, and which limits you're
working around.

Run it in whatever AI coding tool you already use — Claude Code, Cursor, ChatGPT,
or anything else that can read your project.

## Run it

Paste this into your AI tool, from inside your project:

```
Read https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/api-usage-audit/SKILL.md
and follow it to audit how this project uses the Spiffy API.
```

It will read your integration code, ask you a handful of questions it can't
answer from the code alone, and write a file called
`spiffy-integration-diagnostics.md`.

## What you get

**A call budget.** Where your requests actually go, ranked, with the arithmetic
shown — measured against your plan's monthly quota and the per-minute limit.

**Part 1 — what you can fix.** Concrete changes using capabilities that already
exist: pagination you're leaving on the table, `include=` that collapses an N+1,
`updated_at.gte` instead of a full resync, filter keys that are silently doing
nothing, and polling that a webhook would replace.

**Part 2 — feedback for us.** The limitations you had to build around. These
aren't your mistakes, and the report doesn't treat them as such. Sending this
part back is how we find out which gaps actually cost people traffic.

## Privacy

The skill reads your code and writes one file. It doesn't modify your code and it
doesn't send anything anywhere.

Before showing you the report it runs a redaction pass — API keys, tokens,
customer data, internal hostnames and business logic come out; file paths,
endpoint shapes and volumes stay in, since those are what make the report useful.

Sending it to us is entirely your call. If you want our input on what it found,
email it to **support@spiffy.co**.

## A note on accuracy

Volume figures are estimated from your source code and your answers, not measured
from real traffic. Every number in the report is tagged `measured`, `stated`, or
`assumed` so you can tell which is which.

The skill works from a fixed catalog of what the v2 API supports, and it's
instructed not to invent capabilities that aren't in it. If it tells you
something that doesn't match the current API, that's a bug in our catalog — let
us know.

## Docs

- [API reference](https://developers.spiffy.co/api/getting-started)
- [Pagination](https://developers.spiffy.co/api/getting-started/pagination)
- [Filtering](https://developers.spiffy.co/api/getting-started/filtering)
- [Resource expansion](https://developers.spiffy.co/api/getting-started/resource-expansion)
- [Rate limiting](https://developers.spiffy.co/api/getting-started/rate-limiting)
- [Webhooks](https://developers.spiffy.co/api/webhooks)
