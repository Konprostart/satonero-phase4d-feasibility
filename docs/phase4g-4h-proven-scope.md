# Phase 4G / 4H — proven scope

Status: **proven on disposable regtest at daemon CLI level, pre-settlement.**
Both phases are implemented as GitHub Actions workflows in this repository and
have one green run each. Neither phase involves the Android app, real funds,
settlement or a release of any invoice.

These documents record what the green runs actually asserted. They do not
extend the scope.

## Common setup (both phases)

- Satonero daemon pinned to `2534f8b` (the workflow's "Verify pinned SHAs"
  step fails otherwise).
- `MostroP2P/mostro-regtest` pinned to `c6adbdaa`.
- `nak` v0.22.2 pinned by version and SHA-256.
- Disposable keys for the daemon, maker and taker, generated inside the run and
  never written to the repository.
- bitcoind + three LND nodes (mostro, alice, bob) + in-memory nostr relay.
- Daemon boots against its own LND, syncs, maintenance off.
- Artifacts carry a sanitized manifest and sanitized events only; a six-check
  secret gate must pass before artifacts are uploaded.

## Phase 4G — TakeSell pre-invoice

Green run: `42270459515` (2026-11-06), workflow `phase4g-takesell-regtest`.

Asserted by the run:

| Check | Evidence |
| --- | --- |
| exactly one maker NewOrder, ACK decrypted and verified | `maker_neworder14`, `maker_ack14` |
| exactly one taker TakeSell, response correlated to the order id | `taker_take_sell14` |
| maker receives `waiting-buyer-invoice` | `maker_waiting_buyer_invoice14` |
| kind 38383 for the order is published `s=pending`, then `s=in-progress` | `order38383.initial.json`, `order38383.updated.json` |
| order state transition logged by the daemon | `updated Nostr new Status: waiting-buyer-invoice` |
| no payment, invoice, channel, bond or Cashu activity | manifest `limits`: `payments=0`, `channels=0`, `invoices=0`, `bonds=0`, `cashu=0` |

Manifest schema: `phase4g-safe-evidence-v1`.
Verdict line: `SUCCESS: one NewOrder, one TakeSell, both pre-invoice responses and in-progress order event correlated; artifact validated`.

**Proven:** the v2 (kind 14 / NIP-44) order and take handshake reaches
`waiting-buyer-invoice`, and the public order book event reflects it.

**Not proven by 4G:** anything after the seller is asked for a hold invoice.

## Phase 4H — TakeSell + hold invoice

Green run: `42489003791` (2026-11-08), workflow `phase4h-takesell-invoice-regtest`.

Everything 4G proves, plus:

| Check | Evidence |
| --- | --- |
| exactly one taker `add-invoice` carrying a real payout invoice | `taker_add_invoice_request14` |
| seller hold invoice created and returned (`waiting-seller-to-pay`) | `maker_pay_invoice14`, `taker_waiting_seller_to_pay14` |
| order state becomes `waiting-payment` and the public event stays `in-progress` | manifest, `order38383.waitingpayment.meta.json`, `DIAG class=expected_in_progress_sell_order` |
| no funds moved: no payment, no channel, no bond | manifest `limits`: `payments=0`, `channels=0`, `invoices=1`, `bonds=0`, `cashu=0` |
| payout and hold invoice recorded only as sha256 | manifest `invoice_sha256` |

Manifest schema: `phase4h-safe-evidence-v1`.
Verdict line: `SUCCESS: one NewOrder, one TakeSell, one AddInvoice, hold-invoice PayInvoice, WaitingSellerToPay and in-progress order state correlated; no funds moved; artifact validated`.

**Proven:** the daemon creates the seller hold invoice from the taker's payout
invoice and prompts the seller to pay, with zero payments and zero channels.

**Not proven by 4H:** that the hold invoice is ever paid or accepted, that the
`Active` DMs are emitted, or that the trade can be cancelled/finalised. Those
are Phase 5A.

## What neither phase proves

- No Android app or mobile wallet was involved; every action is driven by `nak`
  or the harness scripts.
- No real Lightning channel, inbound liquidity, routing or fee behaviour.
- No settlement, no release of the seller's hold invoice, no reputation update.
- No test of the daemon against a third-party relay, and no deployment.
- The runs use `2534f8b`, i.e. the pre-`stagea_dm_diag` daemon. Phase 5A is the
  first workflow that builds the branch with the SDK diagnostic.
