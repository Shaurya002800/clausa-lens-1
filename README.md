# CausaLens

**Distributed Incident Replay & Investigation**

> Make distributed incidents replayable.

CausaLens captures a request's journey across an instrumented distributed system, turns the incident into an immutable Replay Capsule, safely reproduces the failure in isolation, and shows how execution changes when one approved condition is modified.

Traditional observability helps engineers inspect what happened. CausaLens adds the controlled reproduction step: replay the failure, change one thing, and inspect the first meaningful divergence.

## Core workflow

```text
Capture
  -> Detect Incident
  -> Reconstruct Trace and Timeline
  -> Compile Replay Capsule
  -> Verify Isolation
  -> Reproduce Baseline Failure
  -> Change One Approved Condition
  -> Run What-if Replay
  -> Diff Executions
  -> Explain the Evidence
```

Replay is the product's center. What-if experimentation is the differentiator that becomes available only after baseline reproduction succeeds.

## Golden demo

The hackathon MVP supports one controlled checkout failure:

```text
Gateway -> Checkout -> Payment -> Ledger
                         |
                         +-- latency exceeds timeout
                                   |
                                   v
                                 retry
                                   |
                                   v
                         duplicate ledger effect
```

The demo captures the incident, reconstructs both payment attempts, packages the required evidence and fixtures, and reproduces the duplicate effect inside a replay-only environment.

The P0 what-if replay changes only payment latency:

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

The Replay Diff highlights the first meaningful divergence: Payment crosses the Checkout timeout threshold in baseline but completes before it in what-if.

## What makes it different

CausaLens is not another log viewer and does not claim to replace observability. Its key artifact and workflow are:

- **Replay Capsule:** minimum sanitized evidence, fixtures, policies, dependency behavior, replay instructions, and failure oracle required for one supported reproduction.
- **Safe baseline replay:** controlled execution with production data stores, credentials, and uncontrolled network access blocked.
- **What-if replay:** a separate run that changes exactly one approved variable without mutating the capsule.
- **Replay Diff:** aligned timelines, changed events, effect delta, oracle delta, and first meaningful divergence.
- **Evidence-backed explanation:** an interpretation tied to concrete replay results rather than an unsupported AI guess.

## High-level architecture

```text
Instrumented demo services
          |
          v
Capture adapters -> Event normalizer
          |
          v
Incident store + Failure oracle
          |
          v
Execution graph + Timeline
          |
          v
Replay Capsule compiler
          |
          v
Validator + Isolated replay runtime
          |
          +-- Baseline replay
          +-- One-variable what-if replay
          |
          v
Replay Diff -> Command Center
```

The MVP uses a Core API, PostgreSQL, one Command Center, one isolated replay worker, controlled dependency simulators, and one checkout duplicate-effect System Pack. OpenTelemetry may supply capture evidence, but CausaLens owns its canonical `ExecutionEvent` format.

## Hackathon scope

P0 requires:

1. one instrumented four-service demo,
2. normalized execution capture,
3. incident detection, graph, and timeline,
4. a validated immutable Replay Capsule,
5. safe baseline reproduction,
6. one latency what-if replay,
7. first-divergence and effect-count comparison, and
8. one repeatable judge-visible workflow with a clean reset.

The nominal event duration is 48 hours, with approximately 36 effective engineering hours. A second intervention, second System Pack, AI, production integrations, and advanced infrastructure remain outside P0.

See [planning/SCOPE.md](planning/SCOPE.md) for the authoritative priority boundary.

## Defensible claims

CausaLens provides controlled, reproducible replay for instrumented systems supported by its replay adapters. It compiles the minimum controlled evidence required by the supported scenario and generates intervention-supported evidence by comparing isolated executions.

It does **not** claim:

- arbitrary production-system replay,
- perfect byte-for-byte determinism,
- full production-environment capture,
- mathematically proven general causality,
- AI-discovered root cause, or
- production-scale observability replacement.

## Current status

The repository contains the refined product, architecture, replay, safety, demo, scope, execution documentation, and the final E0 v1.0 contract freeze in [docs/CONTRACTS.md](docs/CONTRACTS.md). `CONTRACTS.md` is the sole authority for implementation field names, enums, lifecycles, ordering, APIs, interfaces, repository boundaries, and ownership. Implementation has not started.

The next gate is E1 capture-to-incident-trace implementation. Any breaking change to the frozen contracts requires all-member agreement and a contract-version update.

## Documentation

### Product boundary

- [Project Context](docs/PROJECT_CONTEXT.md)
- [Problem Statement](docs/PROBLEM_STATEMENT.md)
- [Scope](planning/SCOPE.md)

### System design

- [Contract Freeze](docs/CONTRACTS.md)
- [Architecture](docs/ARCHITECTURE.md)
- [High-Level Design](docs/HLD.md)
- [Replay Capsule](docs/REPLAY_CAPSULE.md)
- [Replay Safety](docs/REPLAY_SAFETY.md)
- [Replay Differential Analysis](docs/REPLAY_DIFFERENTIAL_ANALYSIS.md)
- [System Packs](docs/SYSTEM_PACKS.md)

### Demo and execution

- [Demo Scenario](docs/DEMO_SCENARIO.md)
- [Hackathon Execution](docs/HACKATHON_EXECUTION.md)
- [E0–E3 Plan](planning/E0-E3.md)
- [Implementation Roadmap](planning/IMPLEMENTATION_ROADMAP.md)
- [Team Workstreams](planning/TEAM_WORKSTREAMS.md)
