# Demo Scenario

This is the golden hackathon scenario. Service names, failure shape, oracle, and P0 intervention remain fixed until the complete demo is reliable.

## Distributed system

```text
User
  |
  v
Gateway
  |
  v
Checkout
  |
  v
Payment Simulator
  |
  v
Ledger
```

Every request carries one `trace_id` and one logical `checkout_id`. Payment attempts retain the same `checkout_id` but have different attempt numbers.

## Captured incident

### Controlled configuration

- Checkout timeout: `200 ms`.
- Maximum payment attempts: `2`.
- Retry begins after the first timeout.
- Faulted payment latency: `350 ms` per attempt.
- Deduplication: disabled.
- Ledger effect key: `checkout_id + payment_attempt`.

### Expected execution

```text
0 ms    Gateway receives POST /checkout
4 ms    Checkout starts logical operation CHK-8271
15 ms   Payment attempt 1 starts with 350 ms latency
204 ms  Checkout crosses its 200 ms timeout
205 ms  Checkout schedules retry
210 ms  Payment attempt 2 starts
365 ms  Payment attempt 1 completes; Ledger commits effect 1
560 ms  Payment attempt 2 completes; Ledger commits effect 2
561 ms  Duplicate-effect failure oracle evaluates true
```

Exact scheduler timestamps may vary within the System Pack's declared tolerance. The required structure is timeout, retry, two attempts, and two ledger effects for one logical checkout.

### Failure oracle

The incident is detected when one `checkout_id` produces two committed ledger effects after a timeout-driven retry.

The captured original must expose:

- the request and trace identifiers,
- both payment attempts,
- the timeout and retry relationship,
- two distinct effect events tied to the same logical checkout, and
- the oracle evidence that created the incident.

## Baseline replay

The Replay Capsule preserves the faulted latency, timeout, retry policy, replay-only state, simulator behavior, and duplicate-effect oracle.

Before replay, the Command Center shows:

```text
CAPSULE VALID                 yes
ISOLATED ENVIRONMENT          yes
PRODUCTION DATABASE ACCESS    blocked
UNCONTROLLED NETWORK ACCESS   blocked
PAYMENT PROVIDER              simulated
LEDGER                        replay-only
```

The baseline replay passes when it safely reproduces the timeout, retry, two payment attempts, two ledger effects, and true failure-oracle result.

## P0 what-if replay

Change exactly one variable:

```text
Payment latency: 350 ms -> 50 ms
```

Timeout, retry policy, deduplication, fixtures, code version, and every other replay condition remain unchanged.

Expected what-if execution:

```text
0 ms   Gateway receives POST /checkout
4 ms   Checkout starts logical operation CHK-8271
15 ms  Payment attempt 1 starts with 50 ms latency
65 ms  Payment attempt 1 completes; Ledger commits one effect
66 ms  Checkout completes before timeout
67 ms  Duplicate-effect failure oracle evaluates false
```

## Expected Replay Diff

```text
BASELINE                         WHAT-IF
Payment latency 350 ms           Payment latency 50 ms
        |                                |
Checkout timeout                 Payment completes
        |                                |
Retry attempt 2                  Checkout completes
        |
Two ledger effects               One ledger effect
        |
Oracle: true                     Oracle: false
```

Expected first meaningful divergence: Payment crosses the Checkout timeout threshold in baseline but completes before it in what-if.

## Judge-facing sequence

1. Trigger the faulted checkout and show `INC-8271` detected.
2. Open the incident trace and timeline.
3. Show the timeout, retry, and duplicate ledger effect.
4. Open the Replay Capsule and its validation evidence.
5. Start the baseline replay and show the isolation gate.
6. Show `FAILURE REPRODUCED` with two replay-only effects.
7. Change payment latency from `350 ms` to `50 ms`.
8. Start the what-if replay.
9. Show the aligned timelines and first meaningful divergence.
10. Show the effect count change from two to one and the oracle change from true to false.

## P1 extension

After P0 is reliable, run a separate intervention that enables deduplication while retaining `350 ms` latency. The retry should remain, while the ledger accepts only one logical effect. This supports an exposed-defect interpretation without mixing two variables in one run.

## Reset contract

One reset operation must:

- remove captured demo incidents and replay runs,
- clear replay-only ledger effects,
- reload deterministic fixtures,
- restore `350 ms` faulted latency,
- disable deduplication,
- reset attempt counters and identifiers, and
- report that the system is ready for a fresh demo.

## Success criterion

From a clean reset, a judge can watch CausaLens capture, trace, safely reproduce, modify, and diff the incident without reading raw logs or requiring a manual data correction.
