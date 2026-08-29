# CausaLens Contract Freeze

This document is the E0 authority for the hackathon contracts. Initial contract versions are `1.0`. Implementations may choose different frameworks, but they must preserve these names, semantics, lifecycle rules, and boundaries.

## Stack

- Core API: Go
- Demo services: Go
- Frontend: Next.js + TypeScript
- Persistence: PostgreSQL
- Local orchestration: Docker Compose
- Capture: canonical CausaLens events
- OpenTelemetry: optional capture adapter only
- Replay: isolated Go worker
- Payment: controlled simulator
- Ledger: replay-only PostgreSQL namespace

## ExecutionEvent v1.0

```json
{
  "schema_version": "1.0",
  "event_id": "evt-001",
  "execution_id": "exec-8271",
  "trace_id": "trace-8271",
  "parent_event_id": "evt-000",
  "replay_run_id": null,
  "component": {"name": "payment", "instance": "payment-1"},
  "operation": {"name": "authorize", "kind": "DEPENDENCY"},
  "event_type": "START",
  "attempt": 1,
  "logical_operation_id": "checkout-8271",
  "occurred_at": "2026-08-29T10:32:01.015Z",
  "sequence": 17,
  "duration_ms": null,
  "status": "RUNNING",
  "attributes": {"configured_latency_ms": 350}
}
```

`OperationKind`: `ENTRYPOINT | DEPENDENCY | STATE_CHANGE | SIDE_EFFECT | CONTROL`  
`EventType`: `START | COMPLETE | ERROR | TIMEOUT | RETRY | EFFECT | STATE_OBSERVATION`  
`Status`: `RUNNING | SUCCESS | FAILED | TIMEOUT | BLOCKED`

`event_id` is globally unique. `trace_id` identifies one request journey; `execution_id` identifies one concrete execution; `logical_operation_id` remains stable across retries; `attempt` starts at 1 and increments only for retries. `replay_run_id` is null for the captured original and references a `ReplayRun` for replay events. `sequence` is monotonic only within one component instance and execution. Events may arrive out of order, and graph reconstruction must not rely solely on wall-clock order. `parent_event_id` is direct execution parentage. `RETRY` observes retry initiation; a graph `RETRY` edge connects the new attempt to the prior attempt. `configured_latency_ms` is configuration; `duration_ms` is observed execution duration.

## Incident v1.0

```text
Incident
├── schema_version
├── incident_id
├── status                 DETECTED | READY | BLOCKED
├── failure_oracle_id
├── failure_oracle_version
├── trace_id
├── execution_id
├── detected_at
├── summary
├── evidence_event_ids[]
└── graph_id
```

An incident is a detected failure artifact. Replay lifecycle belongs to `ReplayRun`, not Incident. P0 condition: one logical checkout + timeout-driven retry + duplicate committed ledger effects.

## Execution Graph v1.0

Nodes contain `event_id`, `component`, `operation`, `event_type`, `attempt`, `occurred_at`, and `status`.

Edges are `PARENT_CHILD | TEMPORAL | DEPENDENCY | RETRY | STATE | SIDE_EFFECT`. Edges represent observed execution structure, not automatic causal claims. `RETRY` connects attempts; `DEPENDENCY` connects caller/dependency execution; `SIDE_EFFECT` connects an operation to a committed effect.

## ReplayCapsule v1.0

```text
ReplayCapsule
├── schema_version
├── capsule_id
├── source incident and trace
├── system_pack id/version
├── sanitized trigger
├── events[]
├── graph
├── state_fixtures[]
├── dependency_fixtures[]
├── timing_policy
├── replay_plan
├── failure_oracle
├── allowed_interventions[]
├── safety_policy
└── integrity { algorithm, digest }
```

Capsules are immutable after compilation. The digest covers all replay-relevant content. They contain no production credentials, destinations, uncontrolled URLs, or non-replay state. Every run references the same capsule hash. Changing fixtures, policies, replay plans, or oracle behavior creates a new capsule.

## ReplayRun v1.0

```text
ReplayRun
├── schema_version
├── run_id
├── capsule_id
├── capsule_hash
├── run_type              BASELINE | WHAT_IF
├── intervention
├── trial_number
├── status                 CREATED | VALIDATING | RUNNING | COMPLETED | FAILED | BLOCKED
├── outcome                REPRODUCED | NOT_REPRODUCED | MITIGATED | UNCHANGED | INCONCLUSIVE
├── started_at
├── completed_at
├── observed_event_ids[]
├── effect_summary
├── failure_oracle_result
├── isolation_evidence
├── diff_id
└── error
```

Baseline has zero interventions. What-if has exactly one. What-if requires a safely completed baseline with outcome `REPRODUCED`. `BLOCKED` is never `NOT_REPRODUCED`; `FAILED` means runtime, fixture, or internal failure. `COMPLETED` requires an outcome. Comparison-derived first divergence belongs to `ReplayDiff`, not `ReplayRun`.

## Effect summary and oracle

Effect values are System Pack-classified committed side effects. P0 example:

```json
{"ledger_commit_count": 2}
```

Oracle ID: `duplicate_ledger_effect`.

The oracle matches when one `logical_operation_id` contains a timeout leading to a retry, a subsequent attempt (`attempt >= 2`), and at least two committed ledger effects. The golden demo expects `payment_attempt_count = 2` and `ledger_commit_count = 2`.

```text
OracleResult
├── matched
├── effect_summary
├── required_evidence_event_ids[]
├── explanation
└── version
```

## Intervention v1.0

```json
{"type":"PAYMENT_LATENCY","from":350,"to":50,"unit":"ms"}
```

The type must be allowed by the capsule and active System Pack. Baseline changes zero variables; what-if changes exactly one. The capsule remains immutable, the effective intervention is stored on `ReplayRun`, and Core validates that safety is not weakened.

## ReplayDiff v1.0

```text
ReplayDiff
├── schema_version
├── diff_id
├── baseline_run_id
├── comparison_run_id
├── intervention
├── matched_events[]
├── added_events[]
├── removed_events[]
├── changed_events[]
├── first_meaningful_divergence
├── baseline_effect_summary
├── comparison_effect_summary
├── baseline_oracle_result
├── comparison_oracle_result
└── limitations[]
```

Incidental timestamp jitter within declared tolerance is ignored. P0 divergence is Payment completing before the Checkout timeout in what-if but crossing it in baseline.

## Safety and reset

Default-deny policy: production credentials `BLOCK`; production destinations `BLOCK`; general internet `BLOCK`; payment provider `SIMULATED`; ledger `REPLAY-ONLY`; database writes `REPLAY NAMESPACE ONLY`.

Required attestation:

```text
SAFE REPLAY
Production access: BLOCKED
Network policy: PASS
Credentials: SANITIZED
Side effects: VIRTUALIZED/NAMESPACED
```

No System Pack may weaken these policies. Reset restores demo state, replay namespace, simulator configuration, fixtures, incidents, runs, interventions, and deterministic identifiers so the golden scenario can run again from a known state.

## SystemPack v1.0

```text
SystemPack
├── id
├── version
└── interface_version
```

Required operations: `normalize`, `detectIncident`, `extractFixtures`, `buildReplayPlan`, `validateCapsule`, `allowedInterventions`, `applyIntervention`, `compare`, `evaluateOutcome`, `labels`.

Core owns orchestration, contracts, lifecycle gates, and safety. Packs own scenario interpretation. Packs cannot weaken isolation, mutate capsules, bypass baseline reproduction, or hardcode experiment outcomes. `interface_version` is Core compatibility; `version` is the specific pack implementation.

## E0 exit criteria

- Stack is frozen.
- Every contract is version `1.0`.
- Every contract has a P0 example.
- P0 intervention and oracle are fixed.
- Safety is default-deny.
- Baseline reproduction is a gate.
- Reset behavior is explicit.
- `run_id`, `execution_id`, `trace_id`, `logical_operation_id`, and `event_id` semantics are unambiguous.
- No ambiguous `replay_id` remains.
- Run lifecycle and outcome are separate.
- Three developers can implement Core/Replay, Demo/System Pack, and Frontend independently.
