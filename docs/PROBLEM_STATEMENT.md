# Problem Statement

Distributed incidents are difficult to debug because observability data is fragmented across services, queues, databases, retries, caches, and external dependencies.

Existing logs and traces can reveal where events occurred, but they do not automatically prove which observed condition caused the failure, which condition merely amplified it, or which proposed fix actually prevents recurrence.

## Core problem
Engineers need a safe way to turn incident evidence into a reproducible experiment.

CausaLens addresses this by:
1. collecting execution evidence,
2. reconstructing the propagation graph,
3. extracting the minimum state and conditions required for replay,
4. reproducing the incident in isolation,
5. changing one approved variable at a time,
6. comparing intermediate and final outcomes.

## Why this matters
Without causal replay, incident response commonly depends on:
- manual log correlation,
- speculative root-cause hypotheses,
- incomplete staging reproductions,
- risky production experimentation,
- fixes validated only by final outcomes.

CausaLens turns debugging into controlled experimentation.
