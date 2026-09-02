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

Where a skill does carry a list of API specifics, it's because that list is what
an AI will otherwise invent. Those lists are generated from our source and
stamped with a date. If one disagrees with the current API, that's our bug —
tell us.

## Privacy

Skills read your code and write a file. They don't change your code and they
don't send anything anywhere. Sharing anything with us is your call.
