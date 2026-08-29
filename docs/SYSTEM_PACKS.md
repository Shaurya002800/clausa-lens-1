# System Packs

System Packs adapt the CausaLens replay workflow to explicitly supported failure domains without moving scenario-specific logic into the Core API.

A pack is a versioned adapter contract, not a promise that the MVP supports arbitrary systems.

## Version 1 interface

A System Pack defines:

| Capability | Responsibility |
| --- | --- |
| `normalize` | Convert supported source evidence into `ExecutionEvent` records. |
| `detectIncident` | Evaluate the versioned failure oracle and return supporting evidence. |
| `extractFixtures` | Select sanitized state and dependency fixtures required for replay. |
| `buildReplayPlan` | Declare the supported entrypoint, service sequence, timing policy, and reset strategy. |
| `validateCapsule` | Apply scenario-specific capsule validation in addition to core checks. |
| `allowedInterventions` | Publish the exact single-variable changes the pack accepts. |
| `applyIntervention` | Convert one approved intervention into replay configuration. |
| `compare` | Align scenario events and identify meaningful differences. |
| `evaluateOutcome` | Evaluate the failure oracle for captured and replayed executions. |
| `labels` | Supply stable judge-facing service, event, and effect names. |

Every pack has a stable identifier and semantic version. Capsules and replay results record both.

## MVP pack: checkout duplicate effect

The hackathon implements one pack for the Gateway–Checkout–Payment–Ledger scenario.

### Capture knowledge

- Checkout request and logical operation identifier.
- Payment attempt number and controlled latency.
- Checkout timeout and retry event.
- Ledger effect identifier and logical operation identifier.

### Fixtures

- Sanitized checkout request.
- Replay-only customer/order/payment identifiers.
- Payment simulator response and delay.
- Empty, resettable replay ledger state.

### Failure oracle

The oracle evaluates to `true` when one logical checkout operation produces two committed ledger effects after a timeout-driven retry.

### P0 intervention

Change payment simulator latency from above the checkout timeout to below it. No other policy, fixture, or service behavior changes.

### P1 intervention

Enable deduplication while retaining the latency fault. This is a separate run and begins only after the complete P0 workflow is reliable.

### Comparison rules

- Align events by service, operation, logical operation identifier, event type, and attempt.
- Treat timing differences within declared tolerance as equivalent.
- Preserve retry and repeated-effect events.
- Highlight the earliest threshold, status, retry, or effect divergence.

## Core and pack boundary

The Core API owns orchestration, persistence, lifecycle rules, integrity validation, isolation enforcement, and generic diff storage. The pack owns supported evidence translation, scenario fixtures, failure meaning, interventions, comparison semantics, and presentation labels.

The pack cannot weaken the core isolation policy, authorize a production destination, mutate a compiled capsule, or bypass the successful-baseline requirement.

## Future packs

Queue redelivery, retry storm, stale cache, notification fan-out, and dependency failure packs are future examples only. They are not hackathon deliverables and should not be started before the checkout pack passes the complete reset-to-demo workflow.
