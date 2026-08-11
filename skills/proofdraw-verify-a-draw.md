---
name: Independently verify a ProofDraw result
description: >-
  Check that a published draw was fair, using only public sources and no ProofDraw account — re-hash the
  sealed list, confirm the commitment predates the randomness, and re-derive the winning row yourself.
api: openapi/proofdraw-api-openapi.yml
base_url: https://proofdraw.com/api
operations:
  - GET /list/{hash}
  - GET /list/{hash}/ots
  - GET /health
generated: '2026-08-11'
method: generated
source: https://proofdraw.com/api
---

# Independently verify a ProofDraw result

This flow needs **no API key and no account**. That is the point of the product: if verification required
ProofDraw, it would not be verification. Every endpoint below is public.

Inputs you need from the operator (all present on any sealed draw response or on the public receipt page
`https://proofdraw.com/v/{id}`): `list_hash`, `drand_chain`, `drand_round`, `entry_count`,
`winner_row`, `winner_ticket`.

## Steps

1. **Fetch the sealed list** — `GET /list/{hash}` (public, no auth). Returns `text/plain` with
   `Cache-Control: public, max-age=31536000, immutable` — the URL *is* the content address, so the bytes
   can never change.

2. **Re-hash it.** Compute `SHA-256(response_bytes)` and compare to the `{hash}` component of the URL you
   just called. If they differ, the list has been substituted — stop, the draw is not verified.

3. **Read the header, do not trust the caller's framing.** The file begins with `#` header lines:

   ```
   # proofdraw v2
   # draw: K7M2
   # chain: quicknet
   # round: 8392001
   # round_time: 2026-05-07T18:42:38Z
   # entries: 3
   ```

   The hash covers the header, so the entry list and the drand round are bound together as one artifact.
   Confirm `chain` and `round` match what the operator published — an operator cannot reinterpret the same
   file against a different round.

4. **Fetch the randomness from drand directly, not from ProofDraw.** Pull round `round` on chain `chain`
   from a League of Entropy endpoint. `quicknet` publishes a round every 3 seconds (genesis 1692803367);
   `classic` every 30 seconds (genesis 1595431050).

5. **Re-derive the winner yourself**: `winner_row = BigInt(drand_value, 16) mod entry_count`, then read
   the ticket at that 0-based row of the list body (ticket ids follow the header, one per line, in
   submission order). It must equal the published `winner_ticket`.

6. **Check the commitment predates the randomness** — `GET /list/{hash}/ots` (public, no auth) returns the
   binary OpenTimestamps proof (`application/vnd.opentimestamps.ots`). Verify with the standard CLI:

   ```sh
   curl -o "{hash}.txt"     https://proofdraw.com/list/{hash}
   curl -o "{hash}.txt.ots" https://proofdraw.com/list/{hash}/ots
   ots verify {hash}.txt.ots
   ots upgrade {hash}.txt.ots   # pulls the Bitcoin anchor once it lands (~24h after seal)
   ```

   The calendar attestation is verifiable immediately; the Bitcoin block anchor arrives within ~24 hours.
   **A 404 here is a documented state, not a red flag** — OTS submission is best-effort at seal time and a
   background job retries it. The git commit plus the drand round binding are load-bearing on their own.

7. **Cross-check the third-party witness.** The same bytes are mirrored at
   `https://raw.githubusercontent.com/proofdraw/draw-lists/main/sha256/{hash}.txt`, and the sealing commit
   is visible at `public_commit_url`. GitHub's commit timestamp is an independent record that the list
   existed before the round time.

## What each check rules out

| Check | What it prevents |
|---|---|
| Re-hash matches the URL | Entry list swapped after the fact |
| Header names chain + round | Result re-interpreted against a different round |
| drand fetched from the beacon, not ProofDraw | Fabricated randomness |
| `mod entry_count` recomputed locally | Winner chosen rather than derived |
| OTS / git commit timestamp | Operator peeking at an already-published round before "committing" |

## Notes

- Errors on both public paths are **plain text, not the JSON envelope**: `400` if the hash fails
  `^[0-9a-f]{64}$`, `404` if no sealed list (or no proof yet) exists for that hash.
- An in-browser implementation of this exact flow is published at
  <https://proofdraw.com/verifier.html>, MIT-licensed at <https://github.com/proofdraw/verifier>. Read it
  rather than reimplementing blind — it is ~1 file of vanilla JS using Web Crypto.
- `GET /api/health` (public) tells you only whether the ProofDraw service is up. It is not required for
  verification: steps 1–7 work against the GitHub mirror and the drand beacon even if ProofDraw is offline.
