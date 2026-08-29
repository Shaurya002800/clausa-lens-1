# High-Level Design

## Purpose

This HLD maps the product workflow to deployable responsibilities. `docs/ARCHITECTURE.md` defines logical behavior; this document defines runtime boundaries, ownership, and integration contracts for the hackathon implementation.

## Runtime view

```text
                    +---------------------------+
                    |      Command Center       |
                    | incident / replay / diff  |
                    +-------------+-------------+
                                  |
                                  | HTTP API
                                  v
                    +---------------------------+
                    |         Core API          |
                    | orchestration + contracts |
                    +------+------+-------------+
                           |      |
                           |      +--------------------------+
                           v                                 v
                    +-------------+                  +----------------+
                    | PostgreSQL  |                  | Replay Worker  |
                    | events/runs |                  | + Validator    |
                    +-------------+                  +-------+--------+
                                                            |
                                            isolated replay network
                                                            |
                          +----------------+-----------------+-------------+
                          v                v                               v
                    +-----------+    +-----------+                  +-----------+
                    | Checkout  |--->| Payment   |----------------->| Ledger    |
                    | replay    |    | replay    |                  | replay    |
                    +-----------+    +-----+-----+                  +-----------+
                                             |
                                             v
                                      +-------------+
                                      | Dependency  |
                                      | Simulator   |
                                      +-------------+

 Instrumented demo: Gateway -> Checkout -> Payment -> Ledger
                         |
                         +---- normalized evidence ----> Core API
```

## Runtime responsibilities

### Command Center

- Lists detected incidents.
- Shows the execution graph and millisecond timeline.
- Displays Replay Capsule validation and isolation evidence.
- Starts baseline and approved what-if replays.
- Shows replay lifecycle without hiding blocked or failed states.
- Presents a side-by-side Replay Diff and first meaningful divergence.

### Core API

- Owns validation and orchestration rules.
- Persists incidents, capsules, run specifications, events, oracle results, and diffs.
- Prevents what-if replay until baseline reproduction succeeds.
- Resolves the correct System Pack version.
- Never sends production credentials or destinations to the replay worker.

### PostgreSQL

PostgreSQL is the authoritative hackathon store for normalized events, incidents, graph relationships, capsule metadata, replay runs, interventions, and diff results. Large captured payloads should be represented by sanitized fixtures or content-addressed references rather than duplicated indiscriminately.

### Replay validator and worker

- Verifies capsule schema and integrity.
- Resolves replay-safe fixtures.
- Applies the default-deny isolation policy before process start.
- Starts the supported replay services in a clean namespace.
- Applies zero interventions for baseline or exactly one for what-if.
- Captures replay events through the same `ExecutionEvent` contract.
- Emits an auditable run result and shuts the namespace down.

### Isolated services

The MVP replay environment contains only the services needed by the golden checkout scenario. Payment-provider behavior is simulated, and Ledger writes target replay-only storage. The runtime must not inherit production endpoints or credentials.

### System Pack

The duplicate-effect System Pack provides the demo-specific adapters, fixtures, failure oracle, intervention schema, comparison rules, and labels. The Core API depends on the pack interface, not checkout-specific logic.

## API resource flow

The implementation may adapt route syntax to the selected framework, but it must preserve these resources and transitions:

| Operation | Result |
| --- | --- |
| ingest normalized events | events are validated and attached to a trace |
| list/open incident | incident, graph, timeline, and oracle evidence are returned |
| compile capsule | immutable capsule and validation status are returned |
| start baseline replay | zero-intervention run is created |
| inspect replay | lifecycle, isolation evidence, trace, effects, and oracle result are returned |
| start what-if replay | one-intervention run is created only after baseline reproduction |
| inspect diff | aligned timelines, first divergence, effect delta, and explanation are returned |
| reset demo | demo state, replay state, and fixtures return to a known clean condition |

## Data ownership

- Capture adapters own raw-source translation only.
- The Core API owns canonical contracts and state transitions.
- The System Pack owns scenario-specific interpretation and approved changes.
- The replay worker owns one isolated execution at a time.
- The analyzer owns comparison output, not orchestration.
- The Command Center renders evidence; it does not infer hidden results client-side.

## Isolation boundary

Everything launched by the replay worker is inside the replay boundary. Access beyond it is denied unless a destination is explicitly replay-safe and allow-listed. Production databases, queues, caches, webhooks, payment APIs, email, SMS, and general outbound internet access remain outside the boundary.

## Reproduction rule

The baseline replay is considered reproduced when all of the following are true:

1. the replay completes without an isolation or integrity violation,
2. the configured failure oracle evaluates to `true`,
3. the expected duplicate effect count is observed,
4. required structural events and retry relationships are present, and
5. timing-sensitive values fall within the System Pack's declared tolerance rather than requiring byte-for-byte timestamp equality.

## Comparison boundaries

- **Captured original versus baseline replay:** verifies whether the isolated runtime reproduced the supported incident.
- **Baseline replay versus what-if replay:** identifies what changed after exactly one approved intervention.

The first comparison controls access to experimentation. The second produces the Replay Diff. The Command Center must not collapse these into a single comparison or describe a what-if result as reproduction evidence.

## Error boundaries

- Invalid or tampered capsule: block before runtime creation.
- Missing fixture or unsupported pack version: mark `BLOCKED` with a concrete reason.
- Attempted prohibited egress or production access: terminate the run and mark `BLOCKED`.
- Service or simulator crash: mark `FAILED`; do not classify it as non-reproduction.
- Completed run with a false oracle: mark `NOT_REPRODUCED` and stop experimentation.

## HLD-to-implementation gate

Before parallel coding begins, E0 must record the selected application stack and freeze versioned definitions for `ExecutionEvent`, `Incident`, graph edges, Replay Capsule, replay run, intervention, Replay Diff, failure oracle, and System Pack. Framework choice may change implementation mechanics but must not change these responsibilities or the product boundary.
