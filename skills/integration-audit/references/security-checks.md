# Spiffy-specific security checks

Generated 2026-08-13 against API v2.

## Scope — read this first

This list covers **exposure of Spiffy credentials and trust of Spiffy payloads**.
That is the whole scope, and the list below is complete.

General application security is **out of scope**: no XSS, CSRF, SQL injection,
dependency CVEs, TLS configuration, password handling, or infrastructure advice.
Those matter, but they aren't Spiffy-specific, the developer has better tools for
them, and an audit that wanders into them stops being useful and starts being
noise. If you notice something serious outside this scope, mention it in one line
and move on — do not turn the report into a security review.

Report these alongside the efficiency findings, not instead of them. Call volume
is the point of this audit.

---

## S1 — A Spiffy API key reachable from a client

**Why this is severe on Spiffy specifically:** API keys carry **all scopes**.
There is no per-key scoping, no read-only key, and no way to mint a key limited
to one resource. An exposed key is full account access — read every customer,
issue refunds, cancel subscriptions, delete products.

**Look for:** a key in frontend source or a built bundle; a key in a mobile app;
a `fetch`/`axios` call to `api.spiffy.co` from browser code; a key in an env var
with a client-exposed prefix (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`,
`EXPO_PUBLIC_`, `GATSBY_`); a key committed to the repo, in a `.env` that isn't
ignored, or in a config file, fixture, or test.

**Say:** all Spiffy API calls belong on a server they control. A browser or mobile
client should call their backend, which calls Spiffy. If a key has been exposed,
it needs rotating in the Spiffy dashboard — not just removing from the code, since
anything committed to git history or shipped in a bundle is already out.

**The efficiency tie-in, worth stating:** the rate limit and monthly quota are
scoped per account, not per key. A leaked key doesn't only expose data — anyone
using it consumes the same budget their integration depends on.

---

## S2 — A webhook endpoint that doesn't verify the signature

**Look for:** a route handling Spiffy webhooks with no HMAC check — no reference
to the `Spiffy-Signature` header, no `createHmac`, no signing secret.

**Why:** the endpoint is a public URL. Without verification, anyone who finds it
can post a forged `order:success` and cause whatever that event triggers —
fulfilment, access provisioning, account credit.

**Say:** verify before doing anything with the payload. The header is
`Spiffy-Signature: t=<unix>,v1=<hex>`; the signature is HMAC-SHA256 over
`` `${t}.${rawBody}` `` using the endpoint's signing secret. See
`webhook-events.md`.

---

## S3 — Signature verified against a re-serialized body

**Look for:** an HMAC computed over `JSON.stringify(req.body)`, or over anything
derived from the already-parsed body, rather than over the raw bytes received.

**Why it's insidious:** parse-then-stringify reproduces the original string most
of the time, so this passes in testing and in normal operation. It breaks on
non-ASCII characters, on floats whose textual form isn't preserved, and on
anything else where the round trip isn't byte-exact. Then signatures fail, the
handler rejects valid deliveries, and — because repeated failures throttle and
eventually auto-disable the endpoint — events start being lost permanently.

**Say:** capture the raw body before JSON parsing and sign that. In Express:
`express.json({ verify: (req, _res, buf) => { req.rawBody = buf } })`.

This is a known trap and a common reason developers abandon webhooks. If you find
it, it belongs in the report's feedback section too — it means our documentation
didn't get them there.

---

## S4 — No replay window on webhook deliveries

**Look for:** signature verification that never checks the `t` value.

**Why:** the server imposes no signature validity window, so this policy is
entirely the receiver's. A captured delivery stays replayable indefinitely.

**Say:** reject deliveries whose timestamp is outside a tolerance they choose —
five minutes is typical. Pair it with idempotency on the envelope `id`, which
they need anyway since delivery is at-least-once.

Lower severity than S1–S3. Note it; don't lead with it.

---

## S5 — Secrets in the wrong place

**Look for:** an API key, OAuth client secret, or webhook signing secret that is
hard-coded, committed, present in a test fixture, or shared with a third party.

**Also flag:** one key shared across every environment and system. Multiple named
keys are supported, and separate keys per system make rotation surgical instead
of an outage — with no per-key scoping (S1), separate keys are the only blast-radius
control available.

---

## S6 — Spiffy customer data in logs

**Look for:** logging entire API responses or webhook payloads. Order and customer
objects carry names, emails, addresses, and partial card details.

**Say:** log identifiers, not payloads. This is the one general-hygiene item worth
keeping, because Spiffy payloads are unusually rich in personal data and the
order-family webhook payload is larger than most developers expect — it arrives
pre-expanded with items, payments, subscriptions, paymentplans, checkout and
checkoutview.

---

## S7 — Mishandled OAuth refresh tokens

Only applies to integrations using OAuth rather than an account API key.

**Look for:** a stored refresh token that is reused, or retry logic that replays a
refresh request after a failure.

**Why:** refresh tokens are **single-use** and rotate on every refresh. Replaying
a consumed one fails, and an integration that doesn't persist the new token
locks itself out and needs re-authorization.

**Say:** persist the new refresh token atomically on every refresh, and don't
retry a refresh with a token that has already been sent.

---

## S8 — Resolving Spiffy records from user-supplied input

**Look for:** a request handler that takes an email, order id, or customer id
from the client — a query string, form field, or request body — and uses it to
look up Spiffy data, without first checking that the caller owns that record.

**Why:** this is the shape of an enumeration bug. If the identifier arrives from
the browser and is passed to Spiffy, anyone can iterate it and read orders and
customer records that aren't theirs. Spiffy payloads carry names, emails,
addresses and partial card details, so the blast radius is real.

**Say:** resolve identity from *your own* authenticated session, then use the
Spiffy id you stored against that user. Never let a client-supplied value select
which Spiffy record to return.

### The `search=` trap specifically

`search=` on customers matches across **`name_first`, `name_last` and `email`
together**, as a partial match — not an exact one. So `?search=<email>` can
return a customer whose *name* happens to contain that string, and can return
several customers.

Using it to answer "which customer is this?" is wrong twice over: it's a
correctness bug, because a match is not an identity, and a security bug, because
the first result may be someone else. If you must look up by email, use the exact
filter `filter[email]=` — and see the note on identity keys in
`webhook-events.md`, because email is a poor identifier even when matched
exactly.
