# Spiffy developer skills

Agent skills for developers building on the [Spiffy API](https://developers.spiffy.co).

Run them in whatever AI coding tool you already use — Claude Code, Cursor,
ChatGPT, or anything else that can read your project. They're plain markdown, so
there's nothing to install and nothing to execute.

## Skills

### [`api-usage-audit`](./api-usage-audit) — where your API calls actually go

Reviews how your application uses the Spiffy API v2, estimates your monthly call
budget against your plan's quota, and tells you what to cut. Produces a file
called `spiffy-integration-diagnostics.md`.

Paste this into your AI tool, from inside your project:

```
Read https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/api-usage-audit/SKILL.md
and follow it to audit how this project uses the Spiffy API.
```

It's most useful if you're hitting rate limits, watching your quota climb, or
wondering whether webhooks would replace a polling job.

## How these work

Each skill carries a `references/` catalog of what the v2 API actually supports —
filter keys per resource, the `include=` matrix, the webhook event list, rate
limits and quotas. The catalogs are **closed-world**: the skill instructs the
reading agent to treat anything absent from them as nonexistent.

That constraint is the whole point. Asked to find API inefficiencies without one,
a capable model will confidently recommend query params and webhook events that
don't exist, in a format plausible enough to survive review. We measured this
before building — it invented nine event names in a single report.

The catalogs are generated from our server source rather than from our published
docs, and each carries a generation date. If a skill tells you something that
doesn't match the current API, that's a bug on our side — please tell us.

## Privacy

These skills read your code and write a file. They don't modify your code and
they don't transmit anything anywhere. Sending a report to us is your choice, and
the skill runs a redaction pass first — credentials, customer data, internal
hostnames and business logic come out; file paths, endpoint shapes and call
volumes stay in.

## Feedback

The diagnostic's report has a section addressed to us, listing the API
limitations your integration had to work around. That half is the reason we
built this: we can see that an account makes 40,000 calls a month, but not that
most of them exist because we don't emit an event you needed.

Send reports, corrections, or anything a skill got wrong to
**support@spiffy.co**.

## Docs

- [API reference](https://developers.spiffy.co/api/getting-started)
- [Authentication](https://developers.spiffy.co/api/getting-started/authentication)
- [Pagination](https://developers.spiffy.co/api/getting-started/pagination)
- [Filtering](https://developers.spiffy.co/api/getting-started/filtering)
- [Resource expansion](https://developers.spiffy.co/api/getting-started/resource-expansion)
- [Rate limiting](https://developers.spiffy.co/api/getting-started/rate-limiting)
- [Webhooks](https://developers.spiffy.co/api/webhooks)
