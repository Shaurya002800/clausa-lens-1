# Problem Statement

Distributed failures are difficult to reproduce.

A single request may cross several services, exceed a timeout, retry, consume or publish messages, mutate state, and trigger external side effects. Logs, metrics, and traces can show pieces of that history, but an engineer still has to reconstruct the conditions and state needed to make the same failure happen again.

## Core problem

Engineers need a safe way to turn one observed distributed incident into a controlled, reproducible execution.

The hard part is not merely finding an error line. It is preserving enough evidence to recreate the request's journey without copying an entire environment or repeating real-world side effects.

## Product response

CausaLens addresses this by:

1. normalizing execution evidence from an instrumented system,
2. detecting a known failure through a scenario-specific failure oracle,
3. reconstructing the request's trace, timeline, and propagation relationships,
4. packaging the minimum required inputs, fixtures, policies, and failure condition into a Replay Capsule,
5. validating that the replay environment is isolated,
6. reproducing the captured failure with a baseline replay,
7. changing exactly one approved condition,
8. comparing the baseline and what-if executions, and
9. highlighting the first meaningful divergence with supporting evidence.

## Why existing observability is not enough

Observability answers important questions about what was recorded. It does not, by itself:

- recreate the state and dependency behavior required for reproduction,
- guarantee that a reproduction attempt is isolated from production,
- verify that the same failure occurred again,
- execute controlled what-if changes, or
- compare intermediate execution differences rather than only final outcomes.

CausaLens complements observability by converting captured execution evidence into a replayable incident artifact.

## Target user outcome

Instead of spending hours manually rebuilding an incident in staging, an engineer can open the incident, inspect its trace, safely reproduce it, change one condition, and see where the execution first diverges.

The project does not claim that a replay diff mathematically proves a universal root cause. It provides concrete, intervention-supported evidence for the supported scenario.
