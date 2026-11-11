# Phase 5A — acceptance criteria

Phase 5A proves the hold-invoice acceptance step of a fixed-price trade at
daemon CLI level on disposable regtest. It does **not** prove settlement, the
Android app, real Lightning liquidity or a deployment.

A Phase 5A result is **accepted** only when every criterion below is evidenced
by a green run of `phase5a-hold-invoice-active`. Partial runs and runs with
unreviewed pin changes do not count.

## Criteria

| ID | Criterion | Evidence in the run |
| --- | --- | --- |
| AC1 | **Pinned sources.** Daemon built from the commit under review, `mostro-regtest` from `c6adbdaa`, `nak` v0.22.2, and the daemon checkout is clean. | `Verify pinned SHAs and clean Satonero tree`. |
| AC2 | **Pre-invoice chain.** Exactly one maker `new-order` (ACK verified), one taker `take-sell`, one taker `add-invoice`, and the seller hold invoice is created. No retries. | `stagea_vars`, DM artifacts, `stagea_payment_state`. |
| AC3 | **Hold invoice accepted.** `HOLD_STATE=ACCEPTED`; seller has exactly one payment, total one, status `IN_FLIGHT`; buyer payout remains `OPEN`; buyer has zero payments. | `Phase 5A gate and sanitized manifest`. |
| AC4 | **Active DMs.** Exactly one `buyer-took-order` (to the maker) and exactly one `hold-invoice-payment-accepted` (to the taker), NIP-44 unwrapped, correct author/order id/`Active` status, observed inside the two independent 60 s windows. | `stagea_dm.out`, `stagea_dm_attempts.log`, `stagea_dm_timing.log`. |
| AC5 | **No settlement.** Nothing is settled or released: `settled=0`, `payouts=0`, buyer payments `0`, exactly one channel seller<->daemon, zero buyer channels, zero pending channels, no bond. | Manifest `limits` + gate. |
| AC6 | **Cancel path.** Hold invoice cancelled and verified `CANCELED`, zero seller `IN_FLIGHT` payments remain, payout still `OPEN`. Verified twice: by the gate and independently by the always-run cleanup. | Gate output + `CLEANUP VERIFIED`. |
| AC7 | **Evidence and secret hygiene.** `phase5a-diagnostics` and the sanitized manifest are uploaded with `if: always()`, the secret scan reports redactions, and no invoice/key material is present (only SHA-256). | Artifact upload steps. |
| AC8 | **Repeatability.** Two consecutive green runs on the same harness commit. Latency comes from the recorded attempt timings, not from an assumption. | Evidence log below. |
| AC9 | **Daemon CI.** The daemon commit being merged is green in `ci.yml`: `fmt`, `clippy`, `test`, `build`, `msrv`. | Daemon CI run. |
| AC10 | **Scope.** CLI/regtest, pre-settlement only. No Android app, no real payment, no mainnet, no release. | Workflow pins and manifest. |

## Evidence log

| Run | Harness commit | Started | Result | Notes |
| --- | --- | --- | --- | --- |
| 43018709176 | `99eff96` | 2026-11-11 12:15Z | FAIL | Wrong author key in the helper (`$DAEMON_ID`, an LND identity, instead of the daemon Nostr pubkey): 0 DMs after 120 s of polling. Root cause identified. |
| 43024050979 | `a657e9c` | 2026-11-11 12:52Z | **PASS** | 29/29 steps; both Active DMs on attempt 1; manifest ok; `CLEANUP VERIFIED`. |
| 43030138039 | `a657e9c` | 2026-11-11 13:33Z | **PASS** | Repeatability on the unchanged harness: 29/29 steps; both DMs attempt 1 at 1 s; `CLEANUP VERIFIED`. |
| 43035353939 | `5b9a066` | 2026-11-11 14:06Z | **PASS** | Probe removed (`4c09157`, zero `order38383_active` output in the log); daemon pin `fb30b78`, image built through the scoped `diag` target (`[diag 1/1] COPY ... stagea_dm_diag`), daemon started from the same image. 29/29 steps, `CLEANUP VERIFIED`. |

## Recorded latency

Run 43024050979:
- `buyer-took-order`: attempt 1 of 19, elapsed 1 s.
- `hold-invoice-payment-accepted`: attempt 1 of 19, elapsed 2 s.
- Daemon log: hold invoice accepted at 13:19:13.196Z; DMs published at
  13:19:13.396Z and 13:19:13.649Z.

Run 43030138039:
- `buyer-took-order`: attempt 1, elapsed 1 s.
- `hold-invoice-payment-accepted`: attempt 1, elapsed 1 s.
- `dm_poll_elapsed_s=60`: the helper ran out the second 60 s window before
  returning; the DMs themselves were found on the first poll.

Run 43035353939:
- `buyer-took-order`: attempt 1, `< 1 s`.
- `hold-invoice-payment-accepted`: attempt 1, `< 1 s`.
- `dm_poll_elapsed_s=0` (both found on the first poll).

## What Phase 5A does not cover

- No Android app; every action is driven by `nak` and the harness scripts.
- No settlement/finalisation, no reputation update, no dispute.
- No real Lightning channel size, inbound liquidity, routing or fees.
- No third-party relay and no deployment.
- The kind-38383 `s=active` probe was removed: the daemon does not publish that
  state (`create_status_tags()` treats `Active` as non-published), so the probe
  could never match. It was diagnostic-only and is not part of acceptance.
