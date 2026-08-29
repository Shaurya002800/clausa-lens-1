# Hackathon Execution

## Time model

- Nominal event duration: 48 hours.
- Effective engineering budget: approximately 36 hours after reviews, checkpoints, integration, presentation work, and interruptions.
- Planning assumes the team protects the final six effective hours for reliability and presentation rather than feature development.

## Operating principle

Build one repeatable proof that a distributed incident can be captured and safely replayed. What-if analysis matters only after reproduction works.

## Critical path

```text
Freeze contracts
  -> Generate golden incident
  -> Capture and reconstruct trace
  -> Compile valid capsule
  -> Prove isolation
  -> Reproduce baseline failure
  -> Run one what-if replay
  -> Highlight first divergence
  -> Harden reset and presentation
```

## Suggested effective-hour allocation

| Window | Outcome |
| --- | --- |
| Hours 0–4 | E0 contracts, stack, repository layout, and golden fixtures frozen. |
| Hours 4–12 | Instrumented demo, event capture, incident oracle, graph, and timeline working. |
| Hours 12–22 | Capsule compilation, validation, isolated runtime, and baseline reproduction working. |
| Hours 22–28 | One latency intervention, Replay Diff, and first divergence working. |
| Hours 28–30 | Command Center workflow integrated. |
| Hours 30–36 | Reset reliability, tests, failure handling, pitch, screenshots, and fallback recording. |

These windows are control limits, not permission to skip an exit criterion.

## Demo gates

1. **Capture gate:** the golden request always creates the expected incident and trace.
2. **Capsule gate:** validation detects tampering, missing fixtures, and unsafe destinations.
3. **Isolation gate:** replay cannot reach production or uncontrolled external services.
4. **Reproduction gate:** baseline replay reaches `REPRODUCED` through the configured oracle.
5. **Intervention gate:** exactly one approved variable changes.
6. **Diff gate:** the analyzer reports the expected first meaningful divergence and effect delta.
7. **Reset gate:** the complete workflow succeeds repeatedly from a clean reset.

## Stop conditions

Do not begin a second intervention, second System Pack, replay-from-checkpoint feature, AI feature, production integration, or richer infrastructure until all P0 gates pass.

If baseline reproduction remains unreliable, remove nonessential UI work and fix capture, fixtures, timing control, or oracle evaluation first.

If the live demo remains unstable at the final hardening window, freeze features and prepare a recorded fallback of the same genuine workflow rather than substituting fabricated results.

## Scope-control test

A proposed feature must materially strengthen at least one P0 property:

- reproducibility,
- replay safety,
- trace and timeline clarity,
- first-divergence clarity, or
- reliable judge-visible execution.

Otherwise it remains P1 or P2.
