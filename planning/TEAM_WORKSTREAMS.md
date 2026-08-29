# Team Workstreams

The team is currently in a preparation and contract-freeze phase. Present responsibilities and later implementation ownership are tracked separately so that planning does not pretend coding has already been divided.

## Current preparation responsibilities

### Member 1 — Judge defense and grill-me preparation

Owns:

- adversarial technical and product questions,
- claim-defense practice,
- competitor and observability-tool comparisons,
- limitations and failure-case questions,
- concise answers about replay fidelity, safety, determinism, and causality, and
- feedback to the repository owner when documentation creates a weak or unsupported claim.

Output: a reviewed question-and-answer set and a list of documentation contradictions or missing defenses.

### Member 2 — Repository and product-contract owner

Owns:

- alignment with the project baseline,
- canonical terminology and product positioning,
- document consistency,
- P0/P1/P2 scope control,
- architecture and data-contract refinement,
- incorporation of reviewed team feedback, and
- approval tracking for repository changes.

Output: one coherent, implementation-ready source of truth for the project.

### Member 3 — Project understanding and comprehension review

Owns:

- learning the complete capture-to-diff workflow,
- explaining every component and boundary without reading the files verbatim,
- checking that diagrams, terms, and contracts are understandable,
- identifying ambiguous concepts or hidden assumptions,
- walking through the golden demo event by event, and
- confirming that a new implementer could use the documentation.

Output: a teach-back of the project plus questions and ambiguity reports for the repository owner.

## Preparation coordination

1. The repository owner publishes a coherent revision.
2. The comprehension reviewer explains it back and identifies ambiguity.
3. The grill-me owner attacks the claims and technical assumptions.
4. The repository owner incorporates accepted corrections without expanding P0.
5. All three approve the E0 contracts before implementation ownership is frozen.

## Post-E0 implementation workstreams

These are component streams, not automatic assignments to the three current members.

### Workstream A — Demo capture and incident reconstruction

Owns demo services, instrumentation, event normalization, persistence, failure-oracle evaluation, graph construction, timeline construction, and incident inspection.

Primary handoff:

```text
instrumented execution -> normalized events -> incident graph and timeline
```

### Workstream B — Capsule and isolated replay

Owns capsule compilation, validation, fixtures, replay runtime, simulators, isolation enforcement, lifecycle, reset, intervention application, and replay event capture.

Primary handoff:

```text
incident + fixtures -> Replay Capsule -> replay result and isolation evidence
```

### Workstream C — Diff, API, and Command Center

Owns orchestration APIs, event alignment, Replay Diff, first divergence, incident and capsule views, replay controls, status views, timeline comparison, and demo integration.

Primary handoff:

```text
baseline + what-if results -> Replay Diff -> judge-visible workflow
```

## Shared responsibilities

- Versioned contract review.
- Integration tests and deterministic fixtures.
- Clean reset and fallback recording.
- Golden demo rehearsal.
- Claim and limitation review.
- Final pitch preparation.

## Coordination rule

E0 contracts are frozen before parallel implementation. A breaking change requires explicit agreement, a version update, identification of affected workstreams, and confirmation that it does not expand P0.
