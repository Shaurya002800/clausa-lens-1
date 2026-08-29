# Replay Differential Analysis

Replay Differential Analysis compares controlled executions. It does not infer a universal root cause from observational telemetry alone.

## Comparison sequence

### 1. Captured original versus baseline replay

This comparison answers: **Did the isolated replay reproduce the supported incident?**

The analyzer checks:

- failure-oracle result,
- required service and operation sequence,
- retry relationships,
- logical effect count,
- dependency behavior,
- terminal status, and
- declared timing tolerances.

The goal is structural and outcome equivalence for the supported scenario, not byte-for-byte equality of timestamps or identifiers.

### 2. Baseline replay versus what-if replay

This comparison answers: **What changed when exactly one approved condition changed?**

The analyzer aligns events using stable semantic keys such as `component.name`, `operation.name`, event type, logical operation identifier, and attempt. It then reports:

- matched events,
- added and removed events,
- changed status, duration, attempt, dependency, or effect values,
- changed failure-oracle result,
- effect-count delta, and
- first meaningful divergence.

## First meaningful divergence

The first meaningful divergence is the earliest aligned point at which an event is added, removed, changes status, crosses a declared timing threshold, or changes a scenario-relevant value.

Incidental timestamp jitter within the System Pack's tolerance is ignored. For the golden latency intervention, the expected divergence is Payment completing before the Checkout timeout instead of crossing it.

## Intervention rule

- Baseline replay uses zero interventions.
- What-if replay uses exactly one approved intervention.
- The intervention must be allowed by the capsule and active System Pack.
- A second changed variable requires a separate run.
- Multiple single-variable results may be discussed together, but they must not be presented as one controlled run.

## Replay Diff contract

The sole normative `ReplayDiff` v1.0 shape, including alignment records, effect summaries, effect delta, first-divergence fields, evidence summary, validation rules, and a golden example, is frozen in [CONTRACTS.md](CONTRACTS.md#replaydiff-v10).

This analyzer must populate that contract rather than defining a local variant. `first_meaningful_divergence` identifies the aligned baseline/comparison events, the rule that made the difference meaningful, their values, and their deterministic timeline indices.

## Evidence-backed interpretation

The analyzer may classify supported conditions as:

- **Trigger candidate:** a changed condition associated with entry into the failing path.
- **Amplifier:** behavior such as retry or fan-out that increases the incident's impact.
- **Exposed defect:** a pre-existing weakness, such as missing deduplication, revealed by the trigger.
- **Mitigation:** a change that prevents the observed symptom without necessarily fixing the underlying weakness.

These labels are interpretations of controlled replay evidence. The UI must show the runs and differences supporting each label.

## Golden-scenario interpretation

With the latency fault active, the expected baseline path is:

```text
Payment delay -> Checkout timeout -> retry -> duplicate ledger effect
```

Reducing only payment latency should remove the timeout, retry, and duplicate effect. If a later P1 run enables deduplication while retaining latency, the retry may remain while the duplicate effect disappears. Together these independent runs support—not mathematically prove—the interpretation that latency is a trigger candidate, retry is an amplifier, and missing deduplication is an exposed defect.

## Limitations

- Results apply to the instrumented system, fixtures, oracle, and System Pack version used by the replay.
- A passed intervention does not establish causality for arbitrary deployments.
- Nondeterminism outside declared controls may reduce comparability.
- AI may summarize an existing Replay Diff in future work, but it must not invent evidence or replace deterministic comparison.
