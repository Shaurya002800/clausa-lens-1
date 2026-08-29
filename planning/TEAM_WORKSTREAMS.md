# Team Workstreams

E0 separates preparation responsibilities from implementation ownership. The implementation mapping below is frozen with the repository layout in [docs/CONTRACTS.md](../docs/CONTRACTS.md#frozen-repository-layout-and-ownership).

## Preparation responsibilities

### Member 1 — Judge defense and grill-me preparation

Owns:

- adversarial technical and product questions,
- claim-defense practice,
- observability-tool comparisons,
- limitations and failure-case questions,
- concise answers about replay fidelity, safety, determinism, and causality, and
- feedback when documentation creates a weak or unsupported claim.

Output: a reviewed question-and-answer set plus documentation contradictions or missing defenses.

### Member 2 — Repository and product-contract owner

Owns:

- alignment with the approved product baseline,
- canonical terminology and positioning,
- documentation and contract consistency,
- P0/P1/P2 scope control,
- architecture and data-contract refinement,
- incorporation of reviewed team feedback, and
- approval tracking for repository changes.

Output: one coherent, implementation-ready source of truth.

### Member 3 — Project comprehension reviewer

Owns:

- learning the complete capture-to-diff workflow,
- explaining every component and boundary,
- checking that diagrams, terms, and contracts are understandable,
- identifying ambiguity or hidden assumptions,
- walking through the golden demo event by event, and
- confirming that a new implementer could use the documentation.

Output: a project teach-back plus ambiguity reports for Member 2.

## Preparation coordination

1. Member 2 publishes the coherent contract revision.
2. Member 3 explains it back and identifies ambiguity.
3. Member 1 attacks the claims and technical assumptions.
4. Member 2 incorporates accepted corrections without expanding P0.
5. All three approve the E0 freeze before implementation begins.

## Frozen implementation ownership

### Member 1 — Demo, capture, and checkout System Pack

Owns:

- Gateway, Checkout, Payment, and Ledger demo services,
- canonical event emission and capture adapters,
- checkout duplicate-effect System Pack,
- failure oracle and effect classification,
- sanitized state and dependency fixtures,
- deterministic golden scenario and reset fixtures, and
- unit tests for demo and pack behavior.

Primary handoff:

```text
demo execution -> normalized events + System Pack + golden fixtures
```

Owned paths:

- `cmd/demo-*`
- `internal/capture`
- `internal/systempack/checkout`
- `test/fixtures/golden`

### Member 2 — Core API, persistence, replay, and differential engine

Owns:

- canonical Go contract types and validation,
- Core API and PostgreSQL migrations,
- incident creation, graph reconstruction, and deterministic timeline,
- Replay Capsule compilation and integrity,
- replay worker, namespace isolation, and safety evidence,
- baseline authorization and intervention enforcement,
- Replay Diff and first meaningful divergence,
- Docker Compose orchestration, and
- unit tests for core, replay, safety, and diff behavior.

Primary handoff:

```text
events + pack -> incident -> capsule -> replay runs -> Replay Diff API resources
```

Owned paths:

- `cmd/core-api`
- `cmd/replay-worker`
- `internal/contracts`
- `internal/core`
- `internal/graph`
- `internal/capsule`
- `internal/replay`
- `internal/differential`
- `db/migrations`
- `deploy/compose.yaml`

Changes to `internal/contracts` require all-member review.

### Member 3 — Command Center and integration

Owns:

- Next.js Command Center,
- incident, graph, and timeline views,
- capsule validation and isolation evidence views,
- baseline and what-if controls,
- replay lifecycle and error presentation,
- side-by-side Replay Diff and first-divergence UI,
- API client integration,
- end-to-end integration tests, and
- judge-visible demo workflow.

Primary handoff:

```text
Core API resources -> verified judge-visible workflow
```

Owned paths:

- `web`
- `test/integration`

## Shared responsibilities

- Contract-change review.
- End-to-end integration and deterministic reset checks.
- Isolation and side-effect review.
- Golden demo rehearsal and fallback recording.
- Claim and limitation review.
- Final pitch preparation.

## Coordination rule

E0 contract names and interfaces are frozen. A breaking change requires all-member agreement, a contract-version update, affected-workstream review, and confirmation that P0 does not expand. Implementation details inside an owned path may change without cross-team approval when they preserve the frozen contracts and safety boundary.
