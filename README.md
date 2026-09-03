# Spiffy developer skills

Agent skills for developers building on [Spiffy](https://developers.spiffy.co).

Run them in whatever AI tool you already use — Claude Code, Cursor, ChatGPT, or
anything else that can read your project. They're plain markdown. Nothing to
install, nothing to execute.

## Skills

| Skill | What it does |
|---|---|
| [`integration-audit`](./skills/integration-audit) | Where your API calls go, what they cost against your quota, and what you can cut — including the per-visit calls that burn a month's quota in days |

## Using one

Paste this into your AI tool from inside your project:

```
Read https://raw.githubusercontent.com/SpiffyUp/developer-skills/main/skills/integration-audit/SKILL.md
and follow it to audit how this project integrates with Spiffy.
```

Each skill's own README covers what it produces and when it's worth running.

## How they work

Skills carry decision rules and the traps that aren't obvious from the docs. For
the technical detail — endpoints, parameters, payloads — they point you at
[developers.spiffy.co](https://developers.spiffy.co).

Endpoints, parameters and event names are read live from
[the OpenAPI spec](https://api.spiffy.co/openapi.json) and from
`GET /v2/webhook-event-types`, so a skill doesn't carry a copy that can go stale
as the API moves.

What a skill does carry is behaviour a spec can't state — which mistakes fail
silently, which calls are irreducible, what a well-built integration looks like.
If one of those tells you something that doesn't match the current API, that's
our bug — tell us.

## Privacy

Skills read your code and write a file. They don't change your code and they
don't send anything anywhere. Sharing anything with us is your call.
