# Implementation Roadmap

This roadmap describes the critical implementation sequence. [E0–E3](E0-E3.md) owns detailed deliverables and exit criteria; [Scope](SCOPE.md) owns priority.

## Phase 1 — Freeze the proof

**Status:** Complete. The authority is [docs/CONTRACTS.md](../docs/CONTRACTS.md).

- Select and record the application stack.
- Freeze versioned contracts and validation behavior.
- Freeze the golden demo values and deterministic fixture contract.
- Define the single duplicate-effect System Pack boundary.
- Define isolation enforcement and clean reset behavior.

**Gate:** E0 exit criterion passes before independent feature implementation.

## Phase 2 — Capture a real incident

- Build the four-service demo path.
- Emit normalized events for request, dependency, timeout, retry, and effects.
- Persist events and evaluate the duplicate-effect oracle.
- Build the execution graph and ordered timeline.
- Expose incident inspection to the Command Center.

**Gate:** E1 exit criterion passes repeatedly from reset.

## Phase 3 — Reproduce safely

- Compile and validate a Replay Capsule.
- Start a clean isolated replay runtime.
- Load sanitized state and controlled dependency fixtures.
- Capture the replay through the same event contract.
- Persist isolation evidence and evaluate reproduction.
- Exercise blocked, failed, and not-reproduced paths.

**Gate:** E2 exit criterion passes; unsafe capsules are demonstrably blocked.

## Phase 4 — Change one thing and diff

- Validate the approved latency intervention.
- Run the what-if replay without mutating the capsule.
- Align baseline and what-if events.
- Find the first meaningful divergence.
- Compare retry, effect count, and oracle result.
- Render the evidence in one judge-visible flow.

**Gate:** E3 exit criterion passes from reset.

## Phase 5 — Reliability and submission

- Add integration tests for the complete reset-to-diff workflow.
- Repeat the golden demo and record run stability.
- Make failure states understandable in the UI.
- Freeze feature work.
- Prepare architecture visuals, pitch, screenshots, and fallback recording.
- Rehearse defensible claims and limitations.

**Gate:** the team can complete the workflow without manual data edits, hidden commands, or production access.

## Verification matrix

| Concern | Required proof |
| --- | --- |
| Capture | Golden request produces the expected normalized events. |
| Detection | Duplicate-effect oracle creates one incident with evidence. |
| Capsule | Valid input passes; tampering and missing fixtures are blocked. |
| Isolation | Production destinations and uncontrolled egress cannot be reached. |
| Reproduction | Baseline yields the expected timeout, retry, effects, and oracle result. |
| Intervention | Exactly one approved value changes. |
| Diff | Expected first divergence and effect delta are reported. |
| Reset | Repeated clean runs produce the same structural outcome. |

## Non-goal

Do not build a production observability backend, arbitrary replay platform, causal-inference engine, or AI root-cause system during the hackathon.
