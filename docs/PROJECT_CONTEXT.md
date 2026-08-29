# Project Context

## Project

CausaLens — Distributed Incident Replay & Investigation.

## North-star promise

**Make distributed incidents replayable.**

CausaLens captures a request's execution across an instrumented distributed system, reconstructs its execution path, compiles the minimum controlled evidence required for reproduction into a Replay Capsule, and replays the incident inside an isolated environment. After the failure is reproduced, an engineer can change one approved condition, replay again, and inspect the first meaningful divergence.

## Product workflow

```text
Capture execution evidence
        |
        v
Detect an incident
        |
        v
Reconstruct its trace and timeline
        |
        v
Compile a Replay Capsule
        |
        v
Verify replay isolation
        |
        v
Run a baseline replay
        |
        v
Reproduce the failure
        |
        v
Change one approved condition
        |
        v
Run a what-if replay
        |
        v
Find the first divergence
        |
        v
Produce an evidence-backed explanation
```

Replay is the product's center. Controlled experimentation is the differentiator that follows successful reproduction.

## Canonical terms

- **ExecutionEvent:** a normalized record of one meaningful action or state transition during a request.
- **Incident:** a captured execution for which a failure oracle detected an expected violation.
- **Execution trace:** the ordered request journey reconstructed from normalized events.
- **Execution/propagation graph:** the parent, temporal, message, dependency, and effect relationships between events. An edge is not automatically a causal claim.
- **Replay Capsule:** the immutable, versioned artifact containing the controlled inputs, fixtures, policies, and failure oracle required for replay.
- **Captured original:** the execution observed in the instrumented demo system.
- **Baseline replay:** an isolated replay with no intervention, used to verify reproduction.
- **What-if replay:** an isolated replay derived from the same capsule with exactly one approved intervention.
- **Replay Diff:** the structured comparison between executions, including their first meaningful divergence.
- **Evidence-backed explanation:** an interpretation derived from captured and replayed evidence; it is not a mathematical proof of causality.
- **System Pack:** a versioned adapter that defines how one supported failure domain is captured, replayed, evaluated, and safely modified.

## Approved claims

- CausaLens makes supported distributed incidents replayable.
- It provides controlled, reproducible replay for instrumented systems supported by its replay adapters.
- It compiles the minimum controlled evidence and state required by the supported replay scenario.
- It uses replay differences to produce intervention-supported evidence.
- It blocks uncontrolled production side effects during replay.

## Claims we do not make

- Replay of every arbitrary production system.
- Perfect byte-for-byte determinism across all distributed environments.
- Complete capture of an entire production environment.
- Mathematically proven general causal inference.
- AI-generated root cause as a source of truth.
- Replacement of a production-scale observability platform.

## Hackathon constraints

- DevJams'26 nominal duration: 48 hours.
- Effective engineering budget: approximately 36 hours.
- One reliable vertical slice is more valuable than broad platform coverage.
- The MVP uses one instrumented checkout system and one duplicate-effect failure scenario.
- Replay is default-deny and must not reach production data stores or uncontrolled external dependencies.
- Baseline reproduction must pass before any what-if replay is allowed.
- The core may expose a System Pack boundary, but only one pack is required for the MVP.
- AI, production integrations, and advanced causal discovery are outside the MVP.

## First success milestone

A checkout request experiences payment latency, times out, retries, and creates a duplicate ledger effect. CausaLens captures the incident, reconstructs its trace, compiles a valid Replay Capsule, reproduces the duplicate effect in isolation, changes one approved condition, and highlights the first meaningful execution divergence.
