---
name: Schedule a draw and collect the result asynchronously
description: >-
  Build a large or future-dated draw with the granular flow — create, add entries in batches, seal against
  a distant drand round — then take the outcome from the HMAC-signed webhook instead of blocking on HTTP.
api: openapi/proofdraw-api-openapi.yml
base_url: https://proofdraw.com/api
operations:
  - POST /v1/draws
  - POST /v1/draws/{id}/entries
  - POST /v1/draws/{id}/seal
  - GET /v1/draws/{id}
  - POST /v1/draws/{id}/resolve
generated: '2026-08-11'
method: generated
source: https://proofdraw.com/api
---

# Schedule a draw and collect the result asynchronously

Use this instead of `/draws/instant` when the entry list exceeds one batch, when entries arrive over time,
or when the draw must fire at a specific future moment ("noon tomorrow"). Anything with an offset beyond
about 45 seconds **must** use this pattern — the synchronous wait is hard-capped at 60s.

## Steps

1. **Create the draw** — `POST /v1/draws`:

   ```json
   {
     "name": "Q3 audit sample",
     "direction": "loser",
     "drand_chain": "quicknet",
     "callback_url": "https://example.com/proofdraw/webhook"
   }
   ```

   Store the returned `id` (4-char Crockford-Base32, e.g. `K7M2`) — it is the public commitment id — and
   store `callback_secret` immediately. **The secret is returned on this response only and can never be
   read again.**

2. **Add entries in batches** — `POST /v1/draws/{id}/entries`, at most **5,000 entries per call**. Each
   call commits atomically in a transaction, so batches are independently durable. The response returns
   `added`, `entry_count`, and the `tickets[]` array — including any auto-generated ticket ids. **This is
   your only chance to map auto-generated tickets back to your own records**; there is no endpoint that
   reads a draw's entries back.

   Your tier also caps *total* entries per draw (100 on free), separately from the 5,000-per-call ceiling.
   Exceeding it returns `422 entry_limit_exceeded`.

3. **Seal against the future round** — `POST /v1/draws/{id}/seal`:

   ```json
   { "round_offset_seconds": 3600, "wait": false }
   ```

   Guidance from the docs:
   - `30s` — fastest practical default.
   - `60–300s` — when you want auditors or entrants to see the commit *before* the round fires.
   - `3600s+` — scheduled draws. The auto-resolver cron fires at round time.
   - **Maximum 7 days.** Past that, schedule the seal call itself instead of holding a sealed draw open.

   Sealing hashes the canonical list *with* the chain and round in its header, pushes it to the public
   mirror `github.com/proofdraw/draw-lists`, asserts the commit landed ≥10s before the round, and submits
   the hash to an OpenTimestamps calendar. The response carries `list_hash`, `list_url`, `verify_url`,
   `public_commit_url`, `commitment_text` and `tweet_intent_url` — publish `commitment_text` if you want
   observers to see the commitment before the round fires.

   On `500 seal_failed` the draw stays **open** (git push failed, or the commit landed too close to the
   round). Retry `/seal` with a **larger** `round_offset_seconds`.

4. **Take the result from the webhook.** ProofDraw POSTs `draw.sealed`, `draw.resolved` and
   `draw.cancelled` to your `callback_url`. Verify every delivery:

   - Header: `X-ProofDraw-Signature: sha256=<hex>`
   - Key: the `callback_secret` from step 1
   - Algorithm: HMAC-SHA256

   Reject any delivery whose signature does not match. Expect `draw.resolved` up to ~60 seconds after
   `drand_round_time` — the auto-resolver runs on a one-minute tick.

5. **Fallbacks if the webhook does not arrive:**
   - `GET /v1/draws/{id}` — poll; `state` moves `sealed` → `resolved`.
   - `POST /v1/draws/{id}/resolve` — force it from your own timer. Idempotent; returns the same winner.
     `409 not_yet_available` means the round genuinely has not published yet; `503 drand_unavailable`
     is a transient beacon outage and the cron will pick it up regardless.

## Hard rules

- **`GET /v1/draws` returns only the 100 most recent draws and truncates silently** — no cursor, no total,
  no truncation flag. Persist every draw id in your own system at creation time.
- **Once sealed, nothing can change**: no entries added, no edits, no deletion. `DELETE` returns
  `409 state_conflict`. Cancel only while `state == "open"`.
- **`callback_secret` cannot be rotated or re-read.** Losing it means losing the ability to verify that
  draw's deliveries.
- Webhook retry policy, delivery timeouts, ordering and replay protection are **not documented** — treat
  deliveries as at-least-once and de-duplicate on draw `id` + event name yourself.
