# Phase 6A–6D — acceptance record

Phase 6 proves the Android application and a Satonero daemon complete a whole
fixed-price trade through a real Nostr relay on disposable regtest: the app
takes a sell order and reaches `active` (6A), marks the fiat as sent and the
order settles to `success` (6B), the buyer rates the seller and the reputation
comes back to the app (6C), and 6D re-runs the lifecycle against the corrected
rating-event key.

A Phase 6 result is **accepted** only when every criterion below is evidenced
by a green run of the corresponding workflow. Partial runs and runs with
unreviewed pin changes do not count. This record does **not** prove mainnet,
public relays, physical devices, independent operation or a release; see
[What this does not prove](#what-this-does-not-prove).

## Proven scope

| Phase | Workflow | What the app/daemon pair does |
| --- | --- | --- |
| 6A | `phase6a-mobile-takesell` | The real app boots against a real relay, sees the CLI maker's kind-38383 order, takes it, submits the buyer payout invoice and reaches `active`. Nothing is settled. |
| 6B | `phase6b-mobile-settlement` | The buyer taps `fiat-sent`, the seller releases, the daemon settles the seller hold invoice, pays the buyer invoice and the order reaches `success`. Liquidity is proven with `queryroutes` before release. |
| 6C | `phase6c-mobile-rating-reputation` | The buyer rates the seller through the production rating screen; the daemon validates it, publishes a signed kind-38384 aggregate and answers `rate-received`; the app renders the seller's updated reputation on a fresh pending order. Negative rating cases are exercised. |
| 6D | `phase6d-rating-conformance` | The same lifecycle on the corrected kind-38384 `d` tag: both rating directions publish under the **rated** user's trade pubkey, and the negative cases still produce no event. |

The app is driven exclusively through the production entry point
(`bootstrapAndRun`) and production UI; the only test-only code is
`integration_test/regtest_mobile_daemon_test.dart` in the mobile repository.
No production mobile or daemon behaviour is faked.

## Criteria

| ID | Criterion | Evidence in the run |
| --- | --- | --- |
| AC1 | **Pinned sources.** Daemon, `mostro-regtest`, mobile and `nak` at the workflow's pins; daemon and mobile checkout trees clean. | `Verify pinned SHAs and clean trees` |
| AC2 | **App discovery and take.** The app's order book shows the maker order (`REGTEST_GATE1_ORDER_VISIBLE ... items=1`), the app sends `take-sell`, receives `add-invoice` and submits the regtest buyer invoice. | Emulator log + `REGTEST_GATE2_TAKE_ACCEPTED` |
| AC3 | **Active.** Status sequence contains `waiting-buyer-invoice`, `waiting-payment` and ends `active`; session `role=buyer`. | `REGTEST_GATE2_STATUS_SEQUENCE`, `REGTEST_GATE2_ACTIVE` |
| AC4 | **Daemon order event.** The daemon published a signed kind-38383 revision for **this** order; `nak verify` accepts the signature; 6A requires `in-progress`, 6B/6C/6D require `success`. | `daemon published a signed kind-38383 ... event for the order` |
| AC5 | **App DMs.** Every app kind-14 event is addressed to the daemon and all signatures verify. 6A: 2, 6B: 3, 6C: 4, 6D: 4. | `app kind-14 events verified: count=N ...` |
| AC6 | **No premature settlement (6A).** Hold remains `ACCEPTED`, exactly one seller payment `IN_FLIGHT`, buyer payout `OPEN`, daemon channel count 1. | `no settlement: hold still ACCEPTED, ...` |
| AC7 | **Settlement (6B+).** Buyer reaches `fiat-sent`, then `success`; sequence contains both. | `REGTEST_GATE3_STATUS_SEQUENCE`, `REGTEST_GATE3_SUCCESS` |
| AC8 | **Payment states (6B+).** Hold invoice `SETTLED`; exactly one seller payment `SUCCEEDED`; buyer payout invoice `SETTLED`; exactly one daemon payout `SUCCEEDED` (1000 sat), zero `IN_FLIGHT`, zero `FAILED`. | `hold invoice SETTLED ...`, `buyer invoice SETTLED ...` |
| AC9 | **Rating accepted (6C+).** The buyer's rating opens, submits 5/5, and the daemon answers `rate-received` at `success`. | `REGTEST_GATE4_RATE_OPENED`, `..._RATING_SUBMITTED`, `..._RATE_RECEIVED` |
| AC10 | **Reputation published and shown (6C+).** At least one signed kind-38384 aggregate exists; the app renders a fresh pending order carrying the rater's aggregate (`total_reviews=1`, `total_rating=2.5`). | `kind-38384 event count on relay`, `REGTEST_GATE4_REPUTATION` |
| AC11 | **Both directions, correct key (6D).** The app's rating of the seller publishes `d` = the seller's trade pubkey; the seller's rating of the buyer publishes `d` = the buyer's trade pubkey. Both signatures verify. | `app rating accepted: signed kind-38384 d=<rated seller trade pubkey> ...`, `seller rating accepted: ... d=<rated buyer trade pubkey> ...` |
| AC12 | **Negative rating cases (6C/6D).** An invalid value (6) produces no event and is logged as invalid; an unknown order is answered `cant-do`/`not_found`; a non-participant is ignored; a duplicate produces no additional event. | `negative cases: invalid value ignored ..., unknown order cant-do/not_found, non-participant ignored, duplicate ignored` |
| AC13 | **Evidence and secret hygiene.** Diagnostics artifact uploaded with `if: always()`; the secret scan reports redactions; no invoice or key material is stored, only hashes. | Artifact upload + secret gate steps |
| AC14 | **Cleanup.** Disposable containers, volumes and runner temp are gone; the settled hold invoice has nothing to cancel. | `CLEANUP VERIFIED` |
| AC15 | **Scope.** Regtest only, disposable keys, no mainnet, no real funds, no release. | Workflow pins and `PHASE6X_GATE=pass CLEANUP=success` |

## Evidence log

Every listed run is a `workflow_dispatch` run at `attempt=1` with a single
`gate` job; all steps succeeded. `mostro-regtest` is pinned to `c6adbdaa` in
all four.

| Phase | Run | Harness head | Daemon | Mobile | Window (UTC) | Result |
| --- | --- | --- | --- | --- | --- | --- |
| 6A | 43223500816 | `41cc00e` | `378a5c6` | `18b2943` | 13:02:39 → 13:40:51 | **PASS** |
| 6B | 43255923049 | `cc02eea` | `378a5c6` | `62f2830` | 16:11:25 → 16:41:18 | **PASS** |
| 6C | 43283643290 | `657a8a9` | `378a5c6` | `19be723` | 18:48:24 → 19:22:52 | **PASS** |
| 6D | 43297096420 | `de79d05` | `ccce4cb` | `b980eb8` | 20:07:19 → 20:45:29 | **PASS** |

Artifacts: `phase6X-diagnostics` (38 464 B / 43 898 B / 64 753 B / 67 685 B)
and `phase6X-satonero-debug-apk` (~130 MB) for 6A–6D respectively.

### Gate output, final runs

6A (`43223500816`):

```text
REGTEST_GATE1_ORDER_VISIBLE order=87703c7f-... items=1
REGTEST_GATE2_STATUS_SEQUENCE=waiting-buyer-invoice,waiting-payment,active
REGTEST_GATE2_FINAL_STATUS=active
daemon kind-38383 event count on relay: 1
app kind-14 events verified: count=2 all addressed to the daemon, all signatures valid
no settlement: hold still ACCEPTED, seller payment IN_FLIGHT, buyer payout OPEN
PHASE6A_GATE=pass CLEANUP=success
```

6B (`43255923049`):

```text
REGTEST_GATE3_STATUS_SEQUENCE=waiting-buyer-invoice,waiting-payment,active,fiat-sent,settled-hold-invoice,success
REGTEST_GATE3_SUCCESS order=33866dff-... role=buyer status=success
daemon published a signed kind-38383 success event for the order
hold invoice SETTLED and the single seller payment SUCCEEDED
buyer invoice SETTLED, exactly one daemon payout SUCCEEDED, paid=1000 sat, zero IN_FLIGHT, zero FAILED
channels unchanged in shape: daemon=2 seller=1 buyer=1; no leftover payments
CLEANUP VERIFIED
```

6C (`43283643290`):

```text
REGTEST_GATE4_RATING_SUBMITTED order=9372d14a-... rating=5
REGTEST_GATE4_RATE_RECEIVED action=rateReceived status=success
REGTEST_GATE4_REPUTATION order=42590c7c-... amount=1234 total_reviews=1 total_rating=2.5
kind-38384 event count on relay: 2
negative cases: invalid value ignored (daemon logged the rejection), unknown order cant-do/not_found, non-participant ignored, duplicate ignored
CLEANUP VERIFIED
```

6D (`43297096420`):

```text
REGTEST_GATE4_RATE_RECEIVED action=rateReceived status=success
REGTEST_GATE4_REPUTATION order=b1456ece-... amount=1234 total_reviews=1 total_rating=2.5
38384 id=... d=be03e91e...a7efa4e reviews=1 rating=2 last=4     # buyer trade pubkey
38384 id=... d=0cbe329e...a7f118 reviews=1 rating=2.5 last=5    # seller trade pubkey
app rating accepted: signed kind-38384 d=<rated seller trade pubkey> 1 review / 2.5 (last 5, min 5, max 5)
seller rating accepted: signed kind-38384 d=<rated buyer trade pubkey> 1 review / 2.0 (last 4, min 4, max 4)
negative cases: invalid value ignored (daemon logged the rejection), unknown order cant-do/not_found, non-participant ignored, duplicate ignored
reputation order published and signed: d=b1456ece-... amt=1234 rating_tag=["rating",{"days":0,"since":1794441600,"total_rating":2.5,"total_reviews":1}]
app rendered the same reputation order the rater published
CLEANUP VERIFIED
```

### Rejected attempts

Only the final green run of each phase is evidence. Earlier attempts that a
workflow or driver defect aborted are listed so a rerun is not mistaken for a
redefinition of the gate:

| Run | Phase | Why it failed |
| --- | --- | --- |
| 43214264566 | 6A | The in-container relay read path returned no events while the relay had the event; the workflow now reads through a helper that falls back to a host-side client. |
| 43244609877 | 6B | The `trade.fiatSent` tap landed on the status snackbar that floats over the action row; the driver now waits for the snackbar to clear. |
| 43274795391 | 6C | Transient `curl` exit 22 while downloading the `geth-hdwallet` asset during the Docker build; no Phase 6C logic ran. |
| 43276864546 | 6C | A harness-only `grep -q`/`pipefail` bug (SIGPIPE) failed the last assertion; every functional assertion had passed. |
| 43296643422 | 6D | The daemon checkout used a hardcoded SHA literal while `DAEMON_SHA` changed; the pin assertion failed in under a minute. |

## What this does not prove

- **Not mainnet, not public relays.** Everything runs on a disposable regtest
  stack (bitcoind, three LND nodes, one self-hosted relay).
- **Not a physical device.** The app runs on an x86_64 Android emulator with
  KVM.
- **Not independent operation.** The stack is disposable; no persistent
  deployment, no backup/restore, no watchdog and no operator runbook is
  exercised here.
- **Not a release.** The APK in the artifacts is a debug build; nothing is
  published.
- **Not the relay read path itself.** From run 6A on, the gate prefers a
  host-side client because reads issued inside the relay container returned no
  events in some runs. The cause of that divergence is still unknown; it is a
  property of the test relay, not of the daemon.
- **Not a permanent pin.** Between 6A and 6D the mobile pins are commits on
  test branches (`18b2943`, `62f2830`, `19be723`, `b980eb8`), not on mobile
  `main`. The driver in `19be723` is the file proposed for `main`; while it
  stays on a branch, deleting that branch makes these runs harder to
  reproduce.
- **Not a signature policy.** Commits in these repositories are not signed;
  the quality bar reports that separately.
