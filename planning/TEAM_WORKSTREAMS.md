# Team Workstreams

## Workstream A — Capture / Evidence / Graph
Owns:
- demo instrumentation
- normalized event model implementation
- evidence ingestion/store
- propagation graph builder
- graph query APIs

Primary interface:
normalized events -> incident graph

## Workstream B — Capsule / Replay / Intervention
Owns:
- Replay Capsule compiler
- isolated replay runtime
- dependency stubs
- replay orchestration
- intervention execution
- replay result capture

Primary interface:
incident graph + state -> Replay Capsule -> replay result

## Workstream C — API / Visualization / Integration
Owns:
- control-plane API
- incident list/detail
- graph visualization
- capsule inspection
- replay controls
- baseline/counterfactual diff UI
- demo integration

Primary interface:
core APIs/results -> judge-visible workflow

## Shared responsibilities
- schema review
- integration tests
- demo fixtures
- failure narrative
- pitch preparation

## Coordination rule
Shared contracts are frozen during E0. Breaking changes require explicit agreement rather than unilateral edits.
