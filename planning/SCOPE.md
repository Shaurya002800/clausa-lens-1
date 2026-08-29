# Scope

This file is the authority for hackathon feature priority. A feature is not part of the MVP unless it appears under P0.

## P0 — Required vertical slice

1. **Instrumented distributed demo**
   - Gateway, Checkout, Payment, and Ledger participate in one traceable request.
   - A controlled payment delay exceeds the checkout timeout.
   - The timeout causes a retry and a duplicate ledger effect.

2. **Normalized execution capture**
   - Every event required by the golden incident conforms to the `ExecutionEvent` contract.
   - Events retain trace, parent, attempt, timing, dependency, and effect relationships.

3. **Incident detection and inspection**
   - The duplicate-effect failure oracle creates an incident.
   - The incident exposes an execution graph and ordered timeline.

4. **Immutable Replay Capsule**
   - The capsule passes schema, integrity, safety, and adapter-version validation.
   - It contains or safely references every controlled input required by the golden scenario.

5. **Safe baseline replay**
   - Production data stores and uncontrolled outbound network access are blocked.
   - The baseline replay reproduces the duplicate-effect oracle result.

6. **One controlled what-if replay**
   - Exactly one approved variable changes.
   - The original capsule remains immutable; the run records the intervention separately.

7. **Replay Diff**
   - The UI compares the baseline and what-if timelines.
   - It highlights the first meaningful divergence and the final effect-count difference.

8. **One judge-visible workflow**
   - A user can move from incident detection to trace, capsule, safe replay, what-if replay, and diff without editing raw data.
   - The complete golden demo succeeds repeatedly from a clean reset.

## P1 — Add only after P0 is reliable

- A second single-variable intervention, preferably deduplication while retaining the latency fault.
- Three repeat trials per replay condition to show stability.
- Replay history and run comparison.
- Capsule integrity and isolation evidence surfaced more prominently in the UI.
- Replay from one explicitly supported safe checkpoint.
- Richer graph and timeline interaction.
- A concise evidence-strength summary.
- A second failure variant using the existing System Pack boundary.

## P2 — Future work, not hackathon work

- A second independently implemented System Pack.
- AI-assisted hypothesis generation or narrative summarization.
- Automated remediation.
- Production telemetry and infrastructure integrations.
- Arbitrary state minimization.
- Multi-variable experiment planning or optimization.
- Advanced causal discovery.
- Kubernetes, Kafka, Redis, or a graph database unless already required by the selected MVP stack.
- Production-scale ingestion, retention, or tenancy.

## Explicit non-goals

- Arbitrary production-environment replay.
- Perfect deterministic replay of every distributed system.
- Mathematical proof of general causality.
- Full observability-platform replacement.
- Unrestricted real-world side effects.
- More than one mandatory demo incident.

## Scope-control rule

New work may enter P0 only if it is necessary for safe failure reproduction or the single judge-visible workflow. P1 work begins only after the P0 reset-to-demo path passes repeatedly.
