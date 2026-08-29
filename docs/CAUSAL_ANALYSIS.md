# Causal Analysis

CausaLens uses controlled replay differentials rather than claiming causal inference solely from observational telemetry.

## Experiment rule
Change one approved variable per replay experiment.

Possible interventions include:
- dependency latency
- retry count
- retry interval
- timeout
- deduplication
- idempotency handling
- message ordering
- controlled failure injection
- dependency response

## Comparison levels

### Outcome comparison
Did the final failure disappear, persist, or change form?

### Intermediate comparison
Which nodes, edges, retries, state transitions, or dependency interactions diverged?

## Intended classification

### Initiating trigger
The condition that starts the problematic execution path.

### Amplification behavior
A mechanism that makes the incident larger or more severe, such as retries or fan-out.

### Latent defect
A pre-existing defect exposed by the trigger rather than created by it.

### Temporary mitigation
A change that suppresses the symptom without resolving the underlying defect.

## Important limitation
For the hackathon, CausaLens should present experimental evidence and confidence-oriented explanations rather than claiming mathematically proven causality in arbitrary distributed systems.
