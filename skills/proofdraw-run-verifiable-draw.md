---
name: Run a verifiable draw
description: >-
  Pick a winner (or a loser) from a list in one call, with a public cryptographic receipt anyone can
  re-verify. Use ProofDraw's instant endpoint, handle the 60-second wait cap correctly, and never retry
  blind — there is no idempotency key.
api: openapi/proofdraw-api-openapi.yml
base_url: https://proofdraw.com/api
operations:
  - POST /v1/draws/instant
  - GET /v1/draws/{id}
  - GET /v1/me
  - POST /v1/draws/{id}/resolve
generated: '2026-08-11'
method: generated
source: https://proofdraw.com/api
---

# Run a verifiable draw

ProofDraw seals your entry list with a SHA-256 commitment bound to a **future** drand round, so nobody —
including ProofDraw — can know the winner at commit time. When the round publishes,
`winner_row = drand_value mod entry_count`.

## Before you start

- Auth is a static API key as a bearer token: `Authorization: Bearer pd_live_…`.
- **Check your budget first.** `GET /v1/me` returns `usage.draws_this_period` and `usage.draws_limit`
  plus the per-draw entry cap. On the free tier that is **5 draws lifetime — no monthly reset** and 100
  entries per draw. Overrunning returns `403 tier_limit_exceeded`, which is terminal, not retryable.
- **Never put PII in `ticket_id`.** Ticket ids are published in the permanent public list. Use opaque
  ids (hashes, customer numbers) and put private context in `metadata` (≤ 4KB, never sealed).

## Steps

1. **Preflight the quota** — `GET /v1/me`. If `draws_this_period >= draws_limit`, stop and report;
   do not attempt the draw.

2. **Create, seal, and (optionally) resolve in one call** — `POST /v1/draws/instant`:

   ```json
   {
     "name": "Spring giveaway",
     "direction": "winner",
     "drand_chain": "quicknet",
     "round_offset_seconds": 30,
     "wait": true,
     "callback_url": "https://example.com/proofdraw/webhook",
     "entries": [{ "ticket_id": "cust-001" }, { "metadata": { "email_hash": "…" } }]
   }
   ```

   - `round_offset_seconds` is how far in the future the committed round sits. **15s is the floor, 30s is
     the safe default.** The commit must land at least 10s before the round or the seal aborts.
   - Cap the batch at **5,000 entries per call**.
   - Supplying `callback_url` returns a `callback_secret` **on this response only**. Store it now — it is
     never retrievable again, and it is the HMAC key for every `X-ProofDraw-Signature: sha256=<hex>`
     delivery on this draw.

3. **Read the state, not the status code.**
   - `state: "resolved"` → `winner_ticket`, `winner_row`, `drand_round` and `verify_url` are populated.
     You are done.
   - `state: "sealed"` → the synchronous wait hit its **60-second hard cap** before the round published.
     **This is a success, not a failure.** The draw is committed.

4. **When you get `sealed`, do exactly one of these — never re-POST:**
   - wait for the `draw.resolved` webhook (the auto-resolver cron runs every minute); or
   - poll `GET /v1/draws/{id}` until `state == "resolved"`; or
   - call `POST /v1/draws/{id}/resolve` yourself after `drand_round_time` passes. This operation is
     **idempotent** — calling it on an already-resolved draw returns the same winner.

5. **Publish the receipt.** Share `verify_url` (`https://proofdraw.com/v/{id}`). Entrants verify it
   themselves in-browser; ProofDraw's backend is not involved in verification.

## Hard rules

- **There is no idempotency key.** A retried `POST /v1/draws/instant` creates a *second* permanent public
  draw and burns a second unit of quota. If a call times out at the network layer, recover by listing
  (`GET /v1/draws`) or by the webhook — never by repeating the create.
- **Sealing is irreversible.** Validate entries before you seal. `DELETE /v1/draws/{id}` returns
  `409 state_conflict` on any sealed draw, permanently.
- **Use `wait: true` only when `round_offset_seconds` ≤ ~45.** Beyond that the response will always come
  back `sealed`.

## Error handling

Errors are **not** RFC 9457. Every response is `{ "success": bool, "data": …, "message": string }`;
failures add a `code`:

| code | status | do |
|---|---|---|
| `validation_failed` | 422 | Read the per-field `errors` map. Fix and resend. Not retryable as-is. |
| `unauthenticated` | 401 | Re-issue a key via `POST /v1/auth/login`. |
| `tier_limit_exceeded` | 403 | Terminal on the free tier (lifetime cap). Report; do not retry. |
| `entry_limit_exceeded` | 422 | Shrink the list to the tier cap from `GET /v1/me`. |
| `state_conflict` | 409 | Read `state` first and branch. Never retry blind. |
| `not_yet_available` | 409 | The round has not published. Wait past `drand_round_time`, then retry. |
| `drand_unavailable` | 503 | Transient beacon outage. Retry in a few seconds. |
| `seal_failed` | 500 | Entries persisted, draw still `open`. Call `/seal` again with a larger offset. |
| `rate_limited` | undeclared | Watch `X-RateLimit-Remaining`; back off. No 429 is declared in the spec. |
