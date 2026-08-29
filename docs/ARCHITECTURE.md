# Architecture

## Architectural goal

The architecture exists to make one supported distributed incident safely replayable. Trace reconstruction, capsule compilation, isolation, reproduction, intervention, and diffing form one vertical workflow; none is a standalone platform for the hackathon MVP.

## Principles

1. Normalize external telemetry into a CausaLens-owned event contract.
2. Reconstruct execution and propagation relationships without labeling every edge as causal.
3. Detect incidents through explicit, testable failure oracles.
4. Treat replay inputs as immutable, versioned artifacts.
5. Deny production and uncontrolled external access by default during replay.
6. Require baseline reproduction before allowing a what-if replay.
7. Change exactly one approved variable per what-if replay.
8. Compare intermediate events and effects, not only final success or failure.
9. Keep scenario-specific capture, simulation, and evaluation logic inside a System Pack.
10. Optimize for one reliable golden demo before extensibility.

## Core contracts

### ExecutionEvent

Every captured or replayed action is normalized to one event shape:

| Field | Requirement | Purpose |
| --- | --- | --- |
| `schema_version` | required | Selects the event contract version. |
| `event_id` | required | Uniquely identifies the event within an execution. |
| `trace_id` | required | Groups one distributed request journey. |
| `parent_event_id` | optional | Links direct execution parentage. |
| `service` | required | Names the emitting service. |
| `operation` | required | Names the meaningful operation. |
| `event_type` | required | Classifies start, end, dependency, retry, message, state, effect, or error. |
| `timestamp_ms` | required | Orders the event within the controlled clock model. |
| `duration_ms` | optional | Records a completed operation's duration. |
| `attempt` | required | Distinguishes retries; defaults to `1`. |
| `status` | required | Records success, error, timeout, retrying, or blocked. |
| `links` | optional | Records message, dependency, retry, or effect relationships. |
| `input_ref` | optional | References sanitized captured input. |
| `output_ref` | optional | References sanitized output or effect evidence. |
| `attributes` | optional | Stores allow-listed, sanitized scenario metadata. |

Raw logs or OpenTelemetry spans may feed the normalizer, but they are not the canonical internal model.

### Incident

An incident identifies a captured execution whose failure oracle evaluated to `true`. It records:

- `incident_id`, `trace_id`, and detection time,
- ordered event identifiers,
- graph and timeline references,
- failure-oracle identifier, version, result, and evidence,
- System Pack identifier and version,
- capture source and sanitization status, and
- a human-readable summary derived from structured evidence.

### Execution graph

Graph nodes reference `ExecutionEvent` records. Edge types are limited to:

- `parent`,
- `temporal`,
- `message`,
- `dependency`,
- `retry`, and
- `effect`.

These edges describe observed execution relationships. Causal interpretation is reserved for post-replay evidence.

## Logical components

### 1. Capture adapters and event normalizer

Capture adapters accept evidence from the instrumented demo services. The normalizer validates and converts it into `ExecutionEvent` records while rejecting malformed, unsanitized, or unsupported evidence.

### 2. Incident store and failure oracle

The incident store persists normalized events and request metadata. A versioned failure oracle detects the golden duplicate-effect condition and creates an `Incident` record with the supporting evidence.

### 3. Trace and timeline builder

The builder converts an incident's events into an execution graph and a stable ordered timeline. It preserves retries, dependencies, and repeated effects instead of collapsing them.

### 4. Replay Capsule compiler

The compiler selects the minimum controlled inputs, state fixtures, dependency behavior, timing policy, replay instructions, safety metadata, and failure oracle required for reproduction. It emits a versioned immutable Replay Capsule.

### 5. Replay validator and isolated runtime

The validator checks schema, integrity, adapter compatibility, fixture completeness, sanitization, and isolation policy. The runtime then executes the capsule against replay-only services and records a new trace.

### 6. What-if controller

After baseline reproduction passes, the controller accepts exactly one intervention allowed by both the capsule and System Pack. It does not mutate the capsule; it records a new run specification referencing the original capsule.

### 7. Replay Diff analyzer

The analyzer compares captured original versus baseline for reproducibility, then baseline versus what-if for divergence. It reports matched events, added or removed events, changed attributes, effect counts, oracle results, and the first meaningful divergence.

### 8. Core API and Command Center

The Core API orchestrates the workflow and exposes incident, capsule, replay, and diff resources. The Command Center presents the judge-visible incident trace, isolation proof, replay status, what-if control, timeline diff, and evidence-backed explanation.

### 9. System Pack

The System Pack supplies scenario-specific normalization, fixture extraction, dependency simulation, failure-oracle evaluation, allowed interventions, comparison rules, and visualization labels. The MVP implements one checkout duplicate-effect pack.

## Primary data flow

```text
Instrumented demo services
          |
          v
Capture adapters -> Event normalizer
          |
          v
Incident store -> Failure oracle
          |
          v
Execution graph + timeline
          |
          v
Replay Capsule compiler
          |
          v
Schema + integrity + isolation validation
          |
          v
Baseline isolated replay
          |
          +-- failure not reproduced --> stop and report mismatch
          |
          v
One approved intervention
          |
          v
What-if isolated replay
          |
          v
Replay Diff + first divergence
          |
          v
Command Center explanation
```

## Replay lifecycle

```text
CREATED
  -> VALIDATING
  -> READY
  -> RUNNING
  -> REPRODUCED | NOT_REPRODUCED | FAILED | BLOCKED
```

- `REPRODUCED` means the configured failure oracle matched and the isolation gate passed.
- `NOT_REPRODUCED` means execution completed but the oracle did not match.
- `FAILED` means the replay could not complete because of an internal or fixture error.
- `BLOCKED` means schema, integrity, safety, or adapter validation rejected the run.

Only a baseline run in `REPRODUCED` state may authorize a what-if replay.

## MVP deployment boundary

The MVP contains one Command Center, one Core API, PostgreSQL behind the Core API, one isolated replay runtime, controlled dependency simulators, and the instrumented Gateway–Checkout–Payment–Ledger demo. It does not require Kafka, Redis, Kubernetes, a graph database, or a production telemetry backend.
