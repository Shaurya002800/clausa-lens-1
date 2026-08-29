# CausaLens

CausaLens is a causal replay experimentation platform for distributed systems.

It captures execution evidence from an incident, reconstructs its propagation graph, compiles the minimum safe inputs, state, policies, and failure conditions into an immutable Replay Capsule, verifies reproducibility inside an isolated replay environment, and then performs controlled single-variable interventions to determine why the incident occurred.

## Core idea

Traditional observability platforms are primarily designed to answer **what happened**. CausaLens is designed to experimentally investigate **why it happened**.

The core workflow is:

```text
Distributed System
        |
        v
Execution Evidence
        |
        v
Propagation Graph
        |
        v
Replay Capsule
        |
        v
Isolated Replay
        |
        +--> Baseline Replay
        +--> Counterfactual Intervention A
        +--> Counterfactual Intervention B
        +--> Counterfactual Intervention C
        |
        v
Differential Analysis
        |
        v
Causal Explanation
```

CausaLens aims to distinguish:

- initiating trigger
- amplification behavior
- latent defect
- temporary mitigation

instead of relying only on correlation across logs and traces.

## Hackathon scope

The DevJams'26 build targets a credible end-to-end vertical slice:

1. Generate a known distributed failure.
2. Capture enough execution evidence to reproduce it.
3. Build a propagation graph.
4. Compile a Replay Capsule.
5. Replay the failure in an isolated environment.
6. Modify exactly one approved variable.
7. Compare baseline and counterfactual executions.
8. Visualize the causal difference.

The nominal hackathon duration is 48 hours, with roughly 36 hours treated as effective implementation time after reviews, judging checkpoints, presentations, integration, and interruptions.

## Architecture

The current high-level architecture consists of:

- Capture Layer
- Evidence Store
- Propagation Graph Builder
- Replay Capsule Compiler
- Replay Engine
- Intervention Engine
- Differential / Causal Analyzer
- API and Visualization Layer
- Pluggable System Packs

See [docs/HLD.md](docs/HLD.md) and [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Domain-neutral system packs

The core is designed to stay domain-neutral. System Packs adapt capture, replay, state extraction, and intervention logic for specific distributed failure classes.

Initial examples include:

- duplicate payments
- job/message redelivery
- retry storms
- stale-cache failures
- notification fan-out
- dependency latency failures
- deduplication/idempotency failures

## Current status

Completed so far:

- problem framing
- product definition
- hackathon positioning
- domain-neutral architecture
- replay safety model
- Replay Capsule concept
- counterfactual intervention model
- HLD direction
- E0-E3 execution planning
- parallel team workstream design

Implementation and integration are the next milestones.

## Documentation

- [Project Context](docs/PROJECT_CONTEXT.md)
- [Problem Statement](docs/PROBLEM_STATEMENT.md)
- [Architecture](docs/ARCHITECTURE.md)
- [High-Level Design](docs/HLD.md)
- [Replay Capsule](docs/REPLAY_CAPSULE.md)
- [Replay Safety](docs/REPLAY_SAFETY.md)
- [Causal Analysis](docs/CAUSAL_ANALYSIS.md)
- [System Packs](docs/SYSTEM_PACKS.md)
- [Demo Scenario](docs/DEMO_SCENARIO.md)
- [Hackathon Execution](docs/HACKATHON_EXECUTION.md)
- [E0-E3 Plan](planning/E0-E3.md)
- [Team Workstreams](planning/TEAM_WORKSTREAMS.md)
- [Implementation Roadmap](planning/IMPLEMENTATION_ROADMAP.md)
- [Scope](planning/SCOPE.md)
