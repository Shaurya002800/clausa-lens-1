# Demo Scenario

## Target demo narrative
A distributed request triggers a reproducible failure that is initially ambiguous from logs alone.

Recommended failure shape:
1. API receives a request.
2. Service A invokes or publishes work to Service B.
3. Dependency latency exceeds a threshold.
4. Retry behavior causes duplicate processing or amplification.
5. Final state becomes incorrect.
6. CausaLens captures the incident.
7. A Replay Capsule reproduces it.
8. A single intervention changes retry/deduplication/latency behavior.
9. The counterfactual run changes the propagation path.
10. The UI highlights the divergent events and explains the causal role.

## Judge-facing sequence
- Show the failure.
- Show the propagation graph.
- Open the generated Replay Capsule.
- Run baseline replay.
- Prove isolation.
- Apply one intervention.
- Run counterfactual replay.
- Show graph/event diff.
- Present trigger vs amplifier vs mitigation classification.

## Success criterion
A judge should be able to understand the causal experiment without reading raw logs.
