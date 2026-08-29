# CausaLens E0 Contract Freeze

## Authority

This document is the normative E0 v1.0 contract for the hackathon implementation. Architecture, replay, safety, demo, and planning documents explain these contracts but do not define competing field sets. If another document disagrees with this file, this file wins and the other document must be corrected before implementation continues.

Contract names, field names, enum values, lifecycle rules, API resources, ordering rules, safety gates, repository boundaries, and workstream ownership are frozen. A breaking change requires agreement from all three members, a contract-version change, and an impact review.

## Notation

- Fields are required unless suffixed with `?`.
- `?` means the field is nullable or absent only under the stated rule.
- Timestamps use RFC 3339 with nanosecond-capable precision in UTC.
- Durations and timeline offsets use non-negative integer milliseconds.
- Identifier strings are opaque and globally unique for their resource type.
- JSON objects reject unknown top-level fields in P0 contract validation.
- All enums are closed; implementations must reject unknown values.
- Contract version is the string `1.0`.
- Representative SHA-256 literals in examples demonstrate the required 64-character format; executable fixtures calculate and verify their actual content digests.

## Frozen stack

| Concern | Frozen selection |
| --- | --- |
| Core API and replay worker | Go 1.26.7 |
| Demo services and System Pack | Go 1.26.7 |
| Frontend | Next.js 16.2.11 Active LTS |
| Frontend language | TypeScript 6.0.x |
| Node.js | 20.20.2 |
| Package manager | npm 10.8.2 with committed `package-lock.json` |
| Persistence | PostgreSQL 15.18 |
| Local orchestration | Docker 29.3.1 and Docker Compose 5.1.1 |
| Capture | Canonical CausaLens events |
| OpenTelemetry | Optional input adapter only |
| Payment dependency | Controlled simulator |
| Replay ledger | Replay-only PostgreSQL schema |

Go is an implementation prerequisite and is not currently installed on the review machine. Install the frozen Go version before E1 begins. Package lockfiles and Go module checksums must be committed when scaffolding is created.

Version references: [Go release history](https://go.dev/doc/devel/release), [Next.js releases](https://nextjs.org/blog), and [TypeScript releases](https://devblogs.microsoft.com/typescript/).

## Frozen repository layout and ownership

```text
cmd/
├── core-api/                 Member 2
├── replay-worker/            Member 2
├── demo-gateway/             Member 1
├── demo-checkout/            Member 1
├── demo-payment/             Member 1
└── demo-ledger/              Member 1
internal/
├── contracts/                Member 2; changes require all-member review
├── core/                     Member 2
├── capture/                  Member 1
├── graph/                    Member 2
├── capsule/                  Member 2
├── replay/                   Member 2
├── differential/             Member 2
└── systempack/checkout/      Member 1
web/                          Member 3
db/migrations/                Member 2
deploy/compose.yaml           Member 2
test/fixtures/golden/         Member 1; shared review
test/integration/             Member 3; shared execution
docs/                         Member 2; all-member review
planning/                     Member 2; all-member review
```

- **Member 1:** demo services, capture adapters, checkout System Pack, oracle fixtures, and golden scenario.
- **Member 2:** canonical contracts, Core API, persistence, graph, capsule, replay worker, isolation, differential analyzer, and orchestration.
- **Member 3:** Command Center, API integration, judge-visible workflow, and end-to-end integration tests.
- **Shared:** contract review, reset verification, demo rehearsal, security review, and scope control.

## Common references

```text
ComponentRef
├── name                    string
└── instance                string

OperationRef
├── name                    string
└── kind                    OperationKind

SystemPackRef
├── id                      string
├── version                 semver string
└── interface_version       "1.0"

FailureOracleRef
├── id                      string
└── version                 semver string
```

`OperationKind`: `ENTRYPOINT | INTERNAL | DEPENDENCY | STATE_CHANGE | SIDE_EFFECT | CONTROL`

## ExecutionEvent v1.0

```text
ExecutionEvent
├── schema_version          "1.0"
├── event_id                string
├── execution_id            string
├── trace_id                string
├── parent_event_id?        string
├── replay_run_id?          string
├── component               ComponentRef
├── operation               OperationRef
├── event_type              EventType
├── attempt                 integer >= 1
├── logical_operation_id    string
├── occurred_at             RFC3339 UTC timestamp
├── sequence                integer >= 0
├── duration_ms?            integer >= 0
├── status                  EventStatus
└── attributes              object of System Pack allow-listed JSON values
```

`EventType`: `START | COMPLETE | ERROR | TIMEOUT | RETRY | EFFECT | STATE_OBSERVATION`

`EventStatus`: `RUNNING | SUCCESS | FAILED | TIMEOUT | BLOCKED`

Semantics:

- `event_id` is globally unique.
- `execution_id` identifies one captured or replayed execution.
- `trace_id` identifies the logical request journey and remains stable across original and replay executions.
- `logical_operation_id` remains stable across retries and replays.
- `attempt` starts at `1` and increases only for a retry of the same logical operation.
- `replay_run_id` is absent for the captured original and required for replay events.
- `parent_event_id` is absent only for the root event; it must reference an event in the same execution otherwise.
- `sequence` is monotonic within one component instance and execution, not globally.
- `occurred_at` is observed time. It does not independently prove causal order.
- `configured_latency_ms` is configuration in `attributes`; `duration_ms` is observed duration.

P0 `attributes` allow-list:

| Key | Type | Used by |
| --- | --- | --- |
| `configured_latency_ms` | integer >= 0 | payment start |
| `checkout_timeout_ms` | integer >= 1 | checkout start and timeout |
| `effect_id` | string | ledger effect |
| `effect_committed` | boolean | ledger effect |
| `dependency_name` | string | dependency start/complete |

No captured secret, credential, destination URL, customer identity, or payment instrument may appear in `attributes`.

P0 example:

```json
{
  "schema_version": "1.0",
  "event_id": "evt-payment-1-start",
  "execution_id": "exec-original-8271",
  "trace_id": "trace-8271",
  "parent_event_id": "evt-checkout-start",
  "component": {"name": "payment", "instance": "payment-1"},
  "operation": {"name": "authorize", "kind": "DEPENDENCY"},
  "event_type": "START",
  "attempt": 1,
  "logical_operation_id": "checkout-8271",
  "occurred_at": "2026-08-29T10:32:01.015Z",
  "sequence": 1,
  "status": "RUNNING",
  "attributes": {"configured_latency_ms": 350}
}
```

Validation rejects duplicate event IDs, invalid enums, attempt values below one, non-allow-listed attributes, replay events without `replay_run_id`, captured-original events with `replay_run_id`, and non-root events whose parent cannot be resolved after the capture window closes.

## Deterministic graph and timeline ordering v1.0

Events may arrive out of order. P0 reconstruction uses this deterministic policy:

1. Create hard ordering constraints from `PARENT_CHILD`, `DEPENDENCY`, `RETRY`, `STATE`, and `SIDE_EFFECT` edges.
2. Topologically sort the constrained graph.
3. For concurrently eligible nodes, sort by `occurred_at`.
4. Treat timestamps within `5 ms` as tied.
5. Break ties by `component.name`, `component.instance`, `sequence`, then `event_id`, all ascending.
6. Emit the resulting zero-based `timeline_index` as derived graph output; do not add it to the captured `ExecutionEvent`.
7. A cycle in hard constraints blocks the incident or makes replay alignment `INCONCLUSIVE`; it must never be silently reordered.

Docker Compose P0 services share the host clock. The `5 ms` tolerance handles scheduler jitter, not arbitrary clock skew. Production-grade distributed-clock reconciliation is out of scope.

## Incident v1.0

```text
Incident
├── schema_version          "1.0"
├── incident_id             string
├── status                  IncidentStatus
├── block_reason?           ValidationIssue
├── failure_oracle          FailureOracleRef
├── system_pack             SystemPackRef
├── trace_id                string
├── execution_id            string
├── detected_at             RFC3339 UTC timestamp
├── summary                 string
├── evidence_event_ids[]    non-empty string array
├── graph_id?               string
└── sanitization_status     "PASS" | "FAIL"
```

`IncidentStatus`: `DETECTED | READY | BLOCKED`

- `DETECTED` means the oracle matched but graph/capture validation is not complete.
- `READY` requires a valid acyclic graph, available System Pack version, resolved evidence, and `sanitization_status = PASS`.
- `BLOCKED` requires `block_reason` and cannot be compiled into a capsule.

P0 example:

```json
{
  "schema_version": "1.0",
  "incident_id": "inc-8271",
  "status": "READY",
  "failure_oracle": {"id": "duplicate_ledger_effect", "version": "1.0.0"},
  "system_pack": {"id": "checkout_duplicate_effect", "version": "1.0.0", "interface_version": "1.0"},
  "trace_id": "trace-8271",
  "execution_id": "exec-original-8271",
  "detected_at": "2026-08-29T10:32:01.561Z",
  "summary": "Timeout-driven retry committed two ledger effects for checkout-8271.",
  "evidence_event_ids": ["evt-timeout", "evt-retry", "evt-ledger-1", "evt-ledger-2"],
  "graph_id": "graph-8271",
  "sanitization_status": "PASS"
}
```

## ExecutionGraph v1.0

```text
ExecutionGraph
├── schema_version          "1.0"
├── graph_id                string
├── incident_id             string
├── ordering_policy_version "1.0"
├── nodes[]                 GraphNode
└── edges[]                 GraphEdge

GraphNode
├── event_id                string
└── timeline_index          integer >= 0

GraphEdge
├── edge_id                 string
├── from_event_id           string
├── to_event_id             string
└── type                    GraphEdgeType
```

`GraphEdgeType`: `PARENT_CHILD | TEMPORAL | DEPENDENCY | RETRY | STATE | SIDE_EFFECT`

`TEMPORAL` records derived presentation order only and must not override a hard edge. Other edge types are hard ordering constraints. Edges represent observed structure, not automatic causal claims.

P0 example:

```json
{
  "schema_version": "1.0",
  "graph_id": "graph-8271",
  "incident_id": "inc-8271",
  "ordering_policy_version": "1.0",
  "nodes": [
    {"event_id": "evt-payment-1-start", "timeline_index": 2},
    {"event_id": "evt-retry", "timeline_index": 4},
    {"event_id": "evt-payment-2-start", "timeline_index": 5}
  ],
  "edges": [
    {"edge_id": "edge-retry", "from_event_id": "evt-payment-1-start", "to_event_id": "evt-payment-2-start", "type": "RETRY"}
  ]
}
```

Validation rejects missing node events, duplicate timeline indices, dangling edges, self-edges, and cycles in hard ordering constraints.

## ReplayCapsule v1.0

```text
ReplayCapsule
├── schema_version          "1.0"
├── capsule_id              string
├── created_at              RFC3339 UTC timestamp
├── source
│   ├── incident_id         string
│   ├── trace_id            string
│   ├── execution_id        string
│   ├── capture_environment "DEMO"
│   └── captured_at         RFC3339 UTC timestamp
├── system_pack             SystemPackRef
├── trigger
│   ├── request_or_message  sanitized JSON object
│   └── sanitized_headers   object
├── event_ids[]             non-empty string array
├── graph_id                string
├── state_fixtures[]        StateFixture
├── dependency_fixtures[]   DependencyFixture
├── timing_policy           TimingPolicy
├── replay_plan             ReplayPlan
├── failure_oracle          FailureOracleSpec
├── allowed_interventions[] InterventionSpec
├── safety                  SafetyPolicy
└── integrity
    ├── algorithm           "SHA-256"
    └── digest              lowercase hexadecimal string
```

P0 nested contracts:

```text
StateFixture
├── fixture_id              string
├── kind                    "POSTGRES_ROWSET"
├── content_ref             replay-only reference
├── content_digest          SHA-256 string
├── sanitization_status     "PASS"
└── reset_strategy          "TRUNCATE_AND_LOAD"

DependencyFixture
├── fixture_id              string
├── dependency              "payment_simulator"
├── request_match           JSON object
├── response                JSON object
├── latency_ms              integer >= 0
├── failure_mode            "NONE"
└── invocation_limit        integer >= 1

TimingPolicy
├── clock_tolerance_ms      5
└── timeout_ms              200

ReplayPlan
├── entrypoint              "gateway.checkout"
├── required_components[]   ["gateway", "checkout", "payment", "ledger"]
├── fixture_load_order[]    string array
└── reset_strategy          "GOLDEN_RESET_V1"

FailureOracleSpec
├── id                      "duplicate_ledger_effect"
├── version                 "1.0.0"
├── expected_match          true
└── expected_effect_summary {"payment_attempt_count": 2, "ledger_commit_count": 2}

SafetyPolicy
├── policy_version          "1.0"
├── sanitization_status     "PASS"
├── blocked_destinations[]  non-empty string array
├── allowed_destinations[]  replay-only string array
└── credential_profile      "replay-only"
```

P0 example:

```json
{
  "schema_version": "1.0",
  "capsule_id": "cap-8271",
  "created_at": "2026-08-29T10:33:00Z",
  "source": {
    "incident_id": "inc-8271",
    "trace_id": "trace-8271",
    "execution_id": "exec-original-8271",
    "capture_environment": "DEMO",
    "captured_at": "2026-08-29T10:32:01.561Z"
  },
  "system_pack": {"id": "checkout_duplicate_effect", "version": "1.0.0", "interface_version": "1.0"},
  "trigger": {
    "request_or_message": {
      "method": "POST",
      "path": "/checkout",
      "body": {"checkout_id": "checkout-8271", "amount_minor": 4999, "currency": "INR"}
    },
    "sanitized_headers": {"content-type": "application/json"}
  },
  "event_ids": ["evt-timeout", "evt-retry", "evt-ledger-1", "evt-ledger-2"],
  "graph_id": "graph-8271",
  "state_fixtures": [
    {
      "fixture_id": "state-ledger-empty",
      "kind": "POSTGRES_ROWSET",
      "content_ref": "fixture://golden/ledger-empty-v1",
      "content_digest": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "sanitization_status": "PASS",
      "reset_strategy": "TRUNCATE_AND_LOAD"
    }
  ],
  "dependency_fixtures": [
    {
      "fixture_id": "dependency-payment-350ms",
      "dependency": "payment_simulator",
      "request_match": {"logical_operation_id": "checkout-8271"},
      "response": {"status": "APPROVED"},
      "latency_ms": 350,
      "failure_mode": "NONE",
      "invocation_limit": 2
    }
  ],
  "timing_policy": {"clock_tolerance_ms": 5, "timeout_ms": 200},
  "replay_plan": {
    "entrypoint": "gateway.checkout",
    "required_components": ["gateway", "checkout", "payment", "ledger"],
    "fixture_load_order": ["state-ledger-empty", "dependency-payment-350ms"],
    "reset_strategy": "GOLDEN_RESET_V1"
  },
  "failure_oracle": {
    "id": "duplicate_ledger_effect",
    "version": "1.0.0",
    "expected_match": true,
    "expected_effect_summary": {"payment_attempt_count": 2, "ledger_commit_count": 2}
  },
  "allowed_interventions": [
    {"type": "PAYMENT_LATENCY", "value_type": "INTEGER", "unit": "ms", "minimum": 0, "maximum": 5000}
  ],
  "safety": {
    "policy_version": "1.0",
    "sanitization_status": "PASS",
    "blocked_destinations": ["production-databases", "public-internet", "real-payment-provider"],
    "allowed_destinations": ["payment-simulator", "replay-postgres"],
    "credential_profile": "replay-only"
  },
  "integrity": {
    "algorithm": "SHA-256",
    "digest": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
  }
}
```

Capsule compilation rejects incidents not in `READY`, missing fixtures, incompatible System Packs, failed sanitization, non-replay destinations, unsupported interventions, invalid replay plans, and integrity mismatches. The SHA-256 digest is calculated over deterministic JSON serialization of all capsule fields except `integrity.digest`; object keys are lexicographically ordered and arrays retain contract order. A compiled capsule is immutable. Changing fixtures, policy, plan, oracle, or pack version creates a new capsule ID and digest.

## Intervention v1.0

```text
Intervention
├── type                    "PAYMENT_LATENCY"
├── from                    350
├── to                      50
└── unit                    "ms"

InterventionSpec
├── type                    "PAYMENT_LATENCY"
├── value_type              "INTEGER"
├── unit                    "ms"
├── minimum                 0
└── maximum                 5000
```

P0 example:

```json
{"type": "PAYMENT_LATENCY", "from": 350, "to": 50, "unit": "ms"}
```

Baseline runs require no intervention. What-if runs require exactly one intervention allowed by the capsule and System Pack. `from` must equal the effective baseline value, `to` must satisfy the spec, and the intervention may not change safety policy.

## EffectSummary and OracleResult v1.0

```text
EffectSummary
├── payment_attempt_count   integer >= 0
└── ledger_commit_count     integer >= 0

EffectDelta
├── payment_attempt_count   integer
└── ledger_commit_count     integer

OracleResult
├── oracle                  FailureOracleRef
├── matched                 boolean
├── effect_summary          EffectSummary
├── required_evidence_event_ids[] string array
└── explanation             string
```

P0 example:

```json
{
  "oracle": {"id": "duplicate_ledger_effect", "version": "1.0.0"},
  "matched": true,
  "effect_summary": {"payment_attempt_count": 2, "ledger_commit_count": 2},
  "required_evidence_event_ids": ["evt-timeout", "evt-retry", "evt-ledger-1", "evt-ledger-2"],
  "explanation": "One logical checkout produced two committed ledger effects after a timeout-driven retry."
}
```

The oracle matches only when one `logical_operation_id` contains a timeout, a retry, a subsequent attempt with `attempt >= 2`, and at least two committed ledger effects.

## IsolationEvidence v1.0

```text
IsolationEvidence
├── policy_version          "1.0"
├── verdict                 "PASS" | "FAIL"
├── runtime_namespace       string
├── network_policy          "PASS" | "FAIL"
├── credential_profile      "replay-only"
├── datastore_destinations[] string array
├── simulator_interactions[] DependencyInteraction
├── denied_interactions[]   DependencyInteraction
└── teardown_result         "PASS" | "FAIL"

DependencyInteraction
├── dependency              string
├── destination             string
├── operation               string
└── result                  "SIMULATED" | "ALLOWED" | "DENIED"
```

P0 example:

```json
{
  "policy_version": "1.0",
  "verdict": "PASS",
  "runtime_namespace": "replay-run-base-8271",
  "network_policy": "PASS",
  "credential_profile": "replay-only",
  "datastore_destinations": ["postgres://replay/ledger_run_base_8271"],
  "simulator_interactions": [
    {
      "dependency": "payment_simulator",
      "destination": "http://payment-simulator:8080",
      "operation": "authorize",
      "result": "SIMULATED"
    }
  ],
  "denied_interactions": [],
  "teardown_result": "PASS"
}
```

Any production credential, production destination, uncontrolled internet access, denied interaction, or failed teardown produces `verdict = FAIL`. A run with failed isolation cannot have status `COMPLETED`.

## ReplayRun v1.0

```text
ReplayRun
├── schema_version          "1.0"
├── run_id                  string
├── execution_id            string
├── capsule_id              string
├── capsule_hash            SHA-256 string
├── run_type                "BASELINE" | "WHAT_IF"
├── baseline_run_id?        string
├── intervention?           Intervention
├── trial_number            integer >= 1
├── status                  ReplayRunStatus
├── outcome?                ReplayOutcome
├── started_at?             RFC3339 UTC timestamp
├── completed_at?           RFC3339 UTC timestamp
├── observed_event_ids[]    string array
├── effect_summary?         EffectSummary
├── failure_oracle_result?  OracleResult
├── isolation_evidence?     IsolationEvidence
└── error?                  RunError
```

`ReplayRunStatus`: `CREATED | VALIDATING | RUNNING | COMPLETED | FAILED | BLOCKED`

`ReplayOutcome`: `REPRODUCED | NOT_REPRODUCED | MITIGATED | UNCHANGED | INCONCLUSIVE`

Lifecycle:

```text
CREATED -> VALIDATING
VALIDATING -> RUNNING | BLOCKED | FAILED
RUNNING -> COMPLETED | BLOCKED | FAILED
```

- `BLOCKED` is used for contract, integrity, compatibility, or safety rejection before or during execution.
- `FAILED` is used for internal, fixture, simulator, or service failure.
- `COMPLETED` requires `outcome`, `completed_at`, effect summary, oracle result, and passing isolation evidence.
- Non-completed statuses require `outcome` to be absent.
- `FAILED` and `BLOCKED` require `error`; other statuses require it to be absent.
- `started_at` is set when the runtime begins and remains absent when validation blocks before runtime creation.
- `completed_at` is required for every terminal status: `COMPLETED`, `FAILED`, or `BLOCKED`.
- `isolation_evidence` is required for `COMPLETED` and for any isolation-related `BLOCKED` result.
- Baseline requires `baseline_run_id` and `intervention` to be absent.
- What-if requires a safely completed baseline reference and exactly one intervention.

Outcome applicability:

| Run type | Allowed completed outcomes |
| --- | --- |
| `BASELINE` | `REPRODUCED`, `NOT_REPRODUCED`, `INCONCLUSIVE` |
| `WHAT_IF` | `MITIGATED`, `UNCHANGED`, `INCONCLUSIVE` |

- `REPRODUCED`: baseline oracle matched and reproduction rules passed.
- `NOT_REPRODUCED`: baseline completed safely but oracle did not match.
- `MITIGATED`: what-if completed safely and the baseline failure oracle no longer matched.
- `UNCHANGED`: what-if completed safely and the same failure oracle still matched.
- `INCONCLUSIVE`: execution completed safely but graph alignment or required comparison evidence was insufficient.

P0 baseline example:

```json
{
  "schema_version": "1.0",
  "run_id": "run-base-8271",
  "execution_id": "exec-replay-base-8271",
  "capsule_id": "cap-8271",
  "capsule_hash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "run_type": "BASELINE",
  "trial_number": 1,
  "status": "COMPLETED",
  "outcome": "REPRODUCED",
  "started_at": "2026-08-29T10:34:00Z",
  "completed_at": "2026-08-29T10:34:00.561Z",
  "observed_event_ids": ["evt-replay-timeout", "evt-replay-retry", "evt-replay-ledger-1", "evt-replay-ledger-2"],
  "effect_summary": {"payment_attempt_count": 2, "ledger_commit_count": 2},
  "failure_oracle_result": {
    "oracle": {"id": "duplicate_ledger_effect", "version": "1.0.0"},
    "matched": true,
    "effect_summary": {"payment_attempt_count": 2, "ledger_commit_count": 2},
    "required_evidence_event_ids": ["evt-replay-timeout", "evt-replay-retry", "evt-replay-ledger-1", "evt-replay-ledger-2"],
    "explanation": "Baseline reproduced the timeout-driven duplicate ledger effect."
  },
  "isolation_evidence": {
    "policy_version": "1.0",
    "verdict": "PASS",
    "runtime_namespace": "replay-run-base-8271",
    "network_policy": "PASS",
    "credential_profile": "replay-only",
    "datastore_destinations": ["postgres://replay/ledger_run_base_8271"],
    "simulator_interactions": [
      {
        "dependency": "payment_simulator",
        "destination": "http://payment-simulator:8080",
        "operation": "authorize",
        "result": "SIMULATED"
      }
    ],
    "denied_interactions": [],
    "teardown_result": "PASS"
  }
}
```

P0 what-if example:

```json
{
  "schema_version": "1.0",
  "run_id": "run-whatif-8271",
  "execution_id": "exec-replay-whatif-8271",
  "capsule_id": "cap-8271",
  "capsule_hash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "run_type": "WHAT_IF",
  "baseline_run_id": "run-base-8271",
  "intervention": {"type": "PAYMENT_LATENCY", "from": 350, "to": 50, "unit": "ms"},
  "trial_number": 1,
  "status": "COMPLETED",
  "outcome": "MITIGATED",
  "started_at": "2026-08-29T10:35:00Z",
  "completed_at": "2026-08-29T10:35:00.067Z",
  "observed_event_ids": ["evt-whatif-payment-complete", "evt-whatif-ledger-1"],
  "effect_summary": {"payment_attempt_count": 1, "ledger_commit_count": 1},
  "failure_oracle_result": {
    "oracle": {"id": "duplicate_ledger_effect", "version": "1.0.0"},
    "matched": false,
    "effect_summary": {"payment_attempt_count": 1, "ledger_commit_count": 1},
    "required_evidence_event_ids": ["evt-whatif-payment-complete", "evt-whatif-ledger-1"],
    "explanation": "Payment completed before timeout; no retry or duplicate effect occurred."
  },
  "isolation_evidence": {
    "policy_version": "1.0",
    "verdict": "PASS",
    "runtime_namespace": "replay-run-whatif-8271",
    "network_policy": "PASS",
    "credential_profile": "replay-only",
    "datastore_destinations": ["postgres://replay/ledger_run_whatif_8271"],
    "simulator_interactions": [
      {
        "dependency": "payment_simulator",
        "destination": "http://payment-simulator:8080",
        "operation": "authorize",
        "result": "SIMULATED"
      }
    ],
    "denied_interactions": [],
    "teardown_result": "PASS"
  }
}
```

## RunError and ValidationIssue v1.0

```text
RunError
├── code                    ErrorCode
├── message                 string
├── retryable               boolean
└── details                 JSON object

ValidationIssue
├── code                    ErrorCode
├── path                    JSON-pointer-like string
└── message                 string
```

`ErrorCode`: `SCHEMA_INVALID | INTEGRITY_MISMATCH | PACK_UNAVAILABLE | FIXTURE_MISSING | SANITIZATION_FAILED | ISOLATION_VIOLATION | DESTINATION_BLOCKED | GRAPH_CYCLE | ORACLE_UNAVAILABLE | INTERVENTION_INVALID | INTERNAL_FAILURE`

The API returns stable error codes; UI copy may elaborate but must not replace them.

P0 examples:

```json
{
  "code": "INTEGRITY_MISMATCH",
  "message": "Replay Capsule digest does not match its canonical content.",
  "retryable": false,
  "details": {"capsule_id": "cap-8271"}
}
```

```json
{
  "code": "SCHEMA_INVALID",
  "path": "/attempt",
  "message": "attempt must be an integer greater than or equal to 1"
}
```

## ReplayDiff v1.0

```text
ReplayDiff
├── schema_version          "1.0"
├── diff_id                 string
├── baseline_run_id         string
├── comparison_run_id       string
├── alignment_version       "1.0"
├── intervention            Intervention
├── baseline_oracle_result  OracleResult
├── comparison_oracle_result OracleResult
├── matched_events[]        EventAlignment
├── added_event_ids[]       string array
├── removed_event_ids[]     string array
├── changed_events[]        EventChange
├── first_meaningful_divergence? FirstDivergence
├── baseline_effect_summary EffectSummary
├── comparison_effect_summary EffectSummary
├── effect_delta            EffectDelta
├── evidence_summary        string
└── limitations[]           string array
```

`effect_delta` is comparison minus baseline. A mitigated P0 run therefore has negative attempt and commit deltas.

```text
EventAlignment
├── baseline_event_id       string
└── comparison_event_id     string

EventChange
├── baseline_event_id       string
├── comparison_event_id     string
├── field                   string
├── baseline_value          JSON value
└── comparison_value        JSON value

FirstDivergence
├── baseline_event_id?      string
├── comparison_event_id?    string
├── rule                    string
├── baseline_value          JSON value
├── comparison_value        JSON value
├── baseline_timeline_index integer
└── comparison_timeline_index integer
```

P0 example:

```json
{
  "schema_version": "1.0",
  "diff_id": "diff-8271",
  "baseline_run_id": "run-base-8271",
  "comparison_run_id": "run-whatif-8271",
  "alignment_version": "1.0",
  "intervention": {"type": "PAYMENT_LATENCY", "from": 350, "to": 50, "unit": "ms"},
  "baseline_oracle_result": {
    "oracle": {"id": "duplicate_ledger_effect", "version": "1.0.0"},
    "matched": true,
    "effect_summary": {"payment_attempt_count": 2, "ledger_commit_count": 2},
    "required_evidence_event_ids": ["evt-replay-timeout", "evt-replay-retry", "evt-replay-ledger-1", "evt-replay-ledger-2"],
    "explanation": "Baseline reproduced the duplicate effect."
  },
  "comparison_oracle_result": {
    "oracle": {"id": "duplicate_ledger_effect", "version": "1.0.0"},
    "matched": false,
    "effect_summary": {"payment_attempt_count": 1, "ledger_commit_count": 1},
    "required_evidence_event_ids": ["evt-whatif-payment-complete", "evt-whatif-ledger-1"],
    "explanation": "What-if completed before timeout with one ledger effect."
  },
  "matched_events": [
    {"baseline_event_id": "evt-replay-payment-start", "comparison_event_id": "evt-whatif-payment-start"},
    {"baseline_event_id": "evt-replay-ledger-1", "comparison_event_id": "evt-whatif-ledger-1"}
  ],
  "added_event_ids": [],
  "removed_event_ids": ["evt-replay-timeout", "evt-replay-retry", "evt-replay-ledger-2"],
  "changed_events": [],
  "first_meaningful_divergence": {
    "baseline_event_id": "evt-replay-timeout",
    "comparison_event_id": "evt-whatif-payment-complete",
    "rule": "PAYMENT_COMPLETES_BEFORE_TIMEOUT",
    "baseline_value": "TIMEOUT",
    "comparison_value": "SUCCESS",
    "baseline_timeline_index": 3,
    "comparison_timeline_index": 3
  },
  "baseline_effect_summary": {"payment_attempt_count": 2, "ledger_commit_count": 2},
  "comparison_effect_summary": {"payment_attempt_count": 1, "ledger_commit_count": 1},
  "effect_delta": {"payment_attempt_count": -1, "ledger_commit_count": -1},
  "evidence_summary": "Lower payment latency completed before checkout timeout; retry and duplicate effect disappeared.",
  "limitations": ["Applies to checkout_duplicate_effect pack v1.0.0 fixtures."]
}
```

ReplayDiff requires two runs with status `COMPLETED`, a baseline with outcome `REPRODUCED`, a what-if comparison run whose `baseline_run_id` matches, the same capsule hash, and exactly one recorded intervention. `added_event_ids` exist only in comparison; `removed_event_ids` exist only in baseline. `changed_events` contain aligned pairs with a meaningful field change. Alignment uses the deterministic timeline policy and the semantic keys `component.name`, `operation.name`, `event_type`, `logical_operation_id`, and `attempt`. `first_meaningful_divergence` may be absent only when comparison outcome is `INCONCLUSIVE` or executions are structurally identical.

## Reset contract v1.0

```text
ResetRequest
└── scenario_id             "checkout_duplicate_effect"

ResetResult
├── schema_version          "1.0"
├── reset_id                string
├── status                  "COMPLETED" | "FAILED"
├── cleared_incident_count  integer >= 0
├── cleared_run_count       integer >= 0
├── cleared_ledger_count    integer >= 0
├── fixture_version         "1.0.0"
├── configured_latency_ms   350
├── deduplication_enabled   false
├── next_logical_operation_id "checkout-8271"
└── error?                  RunError
```

P0 example:

```json
{
  "schema_version": "1.0",
  "reset_id": "reset-1",
  "status": "COMPLETED",
  "cleared_incident_count": 1,
  "cleared_run_count": 2,
  "cleared_ledger_count": 3,
  "fixture_version": "1.0.0",
  "configured_latency_ms": 350,
  "deduplication_enabled": false,
  "next_logical_operation_id": "checkout-8271"
}
```

A failed reset blocks the golden demo. Reset must restore deterministic identifiers, fixtures, simulator configuration, empty replay state, and the original fault configuration.

## SystemPack v1.0

```text
SystemPackDescriptor
├── id                      string
├── version                 semver string
└── interface_version       "1.0"
```

P0 descriptor:

```json
{"id": "checkout_duplicate_effect", "version": "1.0.0", "interface_version": "1.0"}
```

Frozen Go interface:

```go
type SystemPack interface {
	Descriptor() SystemPackRef
	Normalize(ctx context.Context, raw RawEvidence) ([]ExecutionEvent, error)
	DetectIncident(ctx context.Context, events []ExecutionEvent) (OracleResult, error)
	ExtractFixtures(ctx context.Context, incident Incident, events []ExecutionEvent) (FixtureSet, error)
	BuildReplayPlan(ctx context.Context, incident Incident, fixtures FixtureSet) (ReplayPlan, error)
	ValidateCapsule(ctx context.Context, capsule ReplayCapsule) []ValidationIssue
	AllowedInterventions() []InterventionSpec
	ApplyIntervention(ctx context.Context, plan ReplayPlan, intervention Intervention) (ReplayPlan, error)
	Compare(ctx context.Context, diffID string, baseline ReplayExecution, comparison ReplayExecution) (ReplayDiff, error)
	EvaluateOutcome(ctx context.Context, execution ReplayExecution) (OracleResult, error)
	Labels() LabelSet
}
```

Supporting boundary types:

```text
RawEvidence
├── source                  string
├── content_type            string
├── received_at             RFC3339 UTC timestamp
└── payload                 byte array

FixtureSet
├── state_fixtures[]        StateFixture
└── dependency_fixtures[]   DependencyFixture

ReplayExecution
├── run                     ReplayRun
├── events[]                ExecutionEvent
└── graph                   ExecutionGraph

LabelSet
├── components              map string -> string
├── operations              map string -> string
├── event_types             map EventType -> string
├── effects                 map string -> string
└── interventions           map string -> string
```

`RawEvidence` is adapter-owned input and is never persisted as canonical evidence without normalization. `ApplyIntervention` returns a copied ReplayPlan; it must not mutate the capsule-derived plan. Core supplies `diffID` so the pack returns a complete ReplayDiff without inventing resource identity.

Every method returns deterministic output for the same validated input. Go errors represent execution failure; domain validation returns `ValidationIssue`. Packs cannot weaken safety, mutate compiled capsules, bypass baseline authorization, access production destinations, or hardcode replay outcomes.

## API resource contract v1.0

```text
AcceptedEventResponse
├── event_id                string
└── status                  "ACCEPTED"

IncidentListResponse
├── items[]                 Incident
└── next_cursor?            string

IncidentListQuery
├── status?                 IncidentStatus
├── cursor?                 string
└── limit?                  integer 1..100; default 20

IncidentDetailResponse
├── incident                Incident
├── graph                   ExecutionGraph
└── events[]                ExecutionEvent in timeline_index order

CreateRunRequest
├── run_type                "BASELINE" | "WHAT_IF"
├── baseline_run_id?        string
└── intervention?           Intervention

CreateDiffRequest
├── baseline_run_id         string
└── comparison_run_id       string

APIErrorResponse
└── error                   RunError
```

`CreateRunRequest` follows ReplayRun authorization rules: baseline omits both optional fields; what-if requires both. `IncidentDetailResponse.events` uses the frozen timeline order, not ingestion order.

| Method and path | Request | Success result |
| --- | --- | --- |
| `POST /v1/events` | `ExecutionEvent` | `202` AcceptedEventResponse |
| `GET /v1/incidents` | IncidentListQuery | `200` IncidentListResponse |
| `GET /v1/incidents/{incident_id}` | none | `200` IncidentDetailResponse |
| `POST /v1/incidents/{incident_id}/capsules` | none | `201` ReplayCapsule |
| `POST /v1/capsules/{capsule_id}/runs` | CreateRunRequest | `202` ReplayRun |
| `GET /v1/runs/{run_id}` | none | `200` ReplayRun |
| `POST /v1/diffs` | CreateDiffRequest | `201` ReplayDiff |
| `GET /v1/diffs/{diff_id}` | none | `200` ReplayDiff |
| `POST /v1/demo/reset` | ResetRequest | `200` ResetResult |

All responses use the contract objects in this file. Validation errors return `400`; missing resources `404`; invalid lifecycle transitions `409`; blocked safety requests `422`; internal failures `500`. Error bodies use `APIErrorResponse` with a stable `ErrorCode`.

P0 request example:

```json
{
  "run_type": "WHAT_IF",
  "baseline_run_id": "run-base-8271",
  "intervention": {"type": "PAYMENT_LATENCY", "from": 350, "to": 50, "unit": "ms"}
}
```

## E0 exit verification

- Stack and source references are frozen.
- Repository layout and member ownership are frozen.
- Every cross-workstream contract is version `1.0`.
- Every contract has exact field names, required/nullable behavior, validation rules, and a P0 example.
- Run status and outcome are separate everywhere.
- Baseline authorization requires status `COMPLETED`, outcome `REPRODUCED`, and passing isolation evidence.
- ReplayDiff, ReplayCapsule, and SystemPack each have one canonical definition.
- Cross-service timeline ordering and tie-breaking are deterministic for P0.
- Safety is default-deny and cannot be weakened by a System Pack.
- Reset behavior and deterministic identifiers are explicit.
- Primary API resources and workstream interfaces are frozen.
- No ambiguous `replay_id`, generic event `links`, or competing contract authority remains.
