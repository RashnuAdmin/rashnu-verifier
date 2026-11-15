# rashnu-verifier

Independently verify a [Rashnu](https://rashnu.io) draw's fairness - standalone,
dependency-light, and with no reliance on the Rashnu platform or website. Run it
yourself, from any machine, using only public data.

Rashnu's whole premise is that you shouldn't have to trust it: every draw's result is
derived deterministically from (1) the full ticket list, published and hash-committed
before the draw's cutoff, and (2) a Bitcoin block that doesn't exist yet at that
cutoff. This package is the actual implementation of that recomputation - the same
code Rashnu itself runs to produce the "official" result, not a reimplementation that
could quietly drift from it.

## Install

```
npx rashnu-verifier <command> [...flags]
```

No install step needed - `npx` fetches and runs it. Or install it globally/locally
with `npm install -g rashnu-verifier` / `npm install rashnu-verifier`.

## Commands

### `compute-winner` - recompute the Bitcoin-derived winning ticket index

```
rashnu-verifier compute-winner \
  --cutoff <unix-seconds> \
  --rng-n <n> \
  --confirmations <k> \
  --tickets <total-tickets-sold>
```

All four flags come from values a draw publishes before its result can exist (its
cutoff timestamp, its RNG parameters, and its final confirmed ticket count) - see the
draw's own verify page for the exact numbers to pass. Fetches live Bitcoin data from
[mempool.space](https://mempool.space)'s public API (the only network dependency this
package has).

Outputs either:

```json
{ "status": "pending", "reason": "..." }
```

while the qualifying Bitcoin block doesn't have enough confirmations yet, or:

```json
{
  "status": "resolved",
  "qualifyingBlockHeight": 975472,
  "qualifyingBlockHash": "...",
  "qualifyingBlockTimestamp": 1794428965,
  "winningTxId": "...",
  "winningTxIndexInBlock": 3,
  "winningTicketIndex": 0
}
```

`winningTicketIndex` is the index (0-based, into the published ticket list) of the
winning ticket.

### `hash-tickets` - recompute a published ticket list's commitment hash

```
rashnu-verifier hash-tickets --file <path-to-ticket-list.json>
```

`--file` is a JSON array of `{ ticketIndex, buyerWalletAddress, ticketPublicId,
confirmedAt }` objects - the exact shape a draw's `ticket-list.json` endpoint
publishes. Outputs `{ "ticketCount": N, "sha256": "..." }`. Compare this hash against
the one the platform committed on-chain (in its `fundDraw` transaction, before the
qualifying Bitcoin block above could exist) - a match means the published list is
exactly what was committed to, not edited afterward.

## Using it as a library

```ts
import { determineWinningTicket, MempoolSpaceSource, canonicalizeTicketList, hashTicketList } from "rashnu-verifier";

const result = await determineWinningTicket(new MempoolSpaceSource(), {
  cutoffUnixSeconds: 1794427275,
  rngN: 3,
  confirmationsRequired: 6,
  totalTicketsSold: 3,
});
```

`BitcoinBlockSource` is an interface, not a fixed implementation - supply your own if
you'd rather read Bitcoin data from a different source (a local node, a different
explorer) than mempool.space.

## The algorithm, in full

This is the trust-critical core of the entire platform. It must stay exactly this
precise, with no implicit behavior, because it is independently re-implemented here
rather than trusted from the platform.

**Committed inputs**, fixed before the qualifying block can exist:

- `T` - the draw's cutoff timestamp (UTC).
- `N` - fixed integer, default **3**, **1-indexed over the block's full transaction
  list including the coinbase transaction as position 1** (so N=3 is the block's 2nd
  non-coinbase transaction - an adjustable constant, not hidden).
- `K` - required confirmations before a candidate block is treated as final, default
  **6** (~60 minutes).
- The full ordered ticket list, canonicalized and hashed at the moment of cutoff -
  before anyone can know what the qualifying block will contain.

**Steps:**

1. **Qualifying height.** `H = H_cutoff + 1`, where `H_cutoff` is the highest block
   whose header timestamp is `<= T`. Height-based, not "whichever block I happened to
   be watching live" - any future observer re-derives the identical `H` purely from
   permanent on-chain timestamps.
2. **Confirmation wait.** Don't evaluate block `H` until it has `>= K` confirmations.
   If `H` is reorged out before that, recompute `H_cutoff`/`H` from the new chain
   state and wait again.
3. **Thin-block fallback.** Once `H` has `K` confirmations, if `txCount(H) < N`,
   advance to `H + 1` and repeat step 2/3. (Sanity cap: abort/alert after 20 hops - an
   operational safety valve, not a cryptographic requirement. In practice this almost
   never triggers; average Bitcoin blocks carry thousands of transactions.)
4. **Select the transaction.** The Nth transaction (1-indexed, coinbase = position 1)
   in block `H`.
5. **Derive the winning index.** Parse that transaction's txid (standard big-endian
   hex display, as shown by mempool.space / blockstream.info / `bitcoin-cli
   getrawtransaction`) as an unsigned integer. `winning_index = txid_int mod
   total_tickets_sold`. (Modulo bias at this scale is undetectable: txid space is 256
   bits against at most low-millions of tickets - no rejection sampling needed.)
6. **Ticket ordering.** By `(created_at ASC, id ASC)`, where `id` is a monotonic
   `bigserial` (not a UUID, which carries no ordering information) - ties are
   structurally impossible.
7. **Canonical hash.** SHA-256 over a fixed-column CSV
   (`ticket_index,buyer_wallet_address,ticket_public_id,confirmed_at`), rows sorted by
   `ticket_index` ascending, each row (including the last) terminated by a single
   `\n`, UTF-8 bytes - not JSON, which has key-order/number-format ambiguity. A
   pretty-printed JSON copy is published for humans; the hash committed on-chain is
   always over this CSV. Implemented in `src/ticket-list.ts` (`canonicalizeTicketList`
   + `hashTicketList`).

## Found a discrepancy?

That's the point of this package existing - [open an
issue](https://github.com/RashnuAdmin/rashnu-verifier/issues/new) with your own
recomputed result and where it diverges from the platform's.
