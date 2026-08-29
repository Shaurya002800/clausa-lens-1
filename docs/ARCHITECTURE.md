# Architecture

## Architectural principles
1. Capture evidence without coupling the core to one business domain.
2. Convert raw telemetry into an execution/propagation graph.
3. Treat replay inputs as an immutable versioned artifact.
4. Never allow a replay to invoke uncontrolled real-world side effects.
5. Permit only explicit, approved counterfactual interventions.
6. Compare intermediate execution, not only final success/failure.
7. Keep domain-specific logic inside System Packs.

## Logical components

### Capture Layer
Ingests requests, messages, spans, dependency interactions, state references, timing metadata, and failure evidence.

### Evidence Store
Persists normalized event evidence required by graph construction and capsule compilation.

### Propagation Graph Builder
Links execution evidence into causal/temporal relationships across services and asynchronous boundaries.

### Replay Capsule Compiler
Selects the minimum safe evidence, relevant state, policies, dependency behavior, and failure conditions required for reproducibility.

### Replay Engine
Runs the capsule in an isolated namespaced environment and records a fresh execution trace.

### Intervention Engine
Applies exactly one approved variable change per experiment, such as latency, retry policy, timeout, deduplication, idempotency behavior, ordering, or dependency response.

### Differential / Causal Analyzer
Compares baseline and counterfactual executions at both the outcome and intermediate-event level.

### API / Visualization Layer
Presents propagation graphs, replay runs, interventions, diffs, and causal explanations.

### System Packs
Provide domain adapters for capture, state extraction, dependency simulation, interventions, invariants, and outcome evaluation.

## Data flow

```text
Instrumented Services / Demo System
              |
              v
        Capture Layer
              |
              v
        Evidence Store
              |
              v
   Propagation Graph Builder
              |
              v
   Replay Capsule Compiler
              |
              v
          Replay Engine
              |
        +-----+------+
        |            |
        v            v
   Baseline      Intervention
    Replay          Runs
        |            |
        +-----+------+
              |
              v
 Differential / Causal Analyzer
              |
              v
      API + Visualization
```
