# Project Context

## Project
CausaLens — causal replay experimentation for distributed systems.

## Objective
Build a hackathon-grade system that captures distributed execution evidence, reconstructs incident propagation, compiles a minimal reproducible Replay Capsule, reproduces the failure safely, and executes controlled counterfactual experiments to identify causal roles.

## Product thesis
Logs, traces, and metrics are strong at showing correlated observations. CausaLens adds an experimentation layer: reproduce the incident, modify one approved variable, and compare the resulting execution.

## Causal roles
The platform should help separate:
- initiating trigger
- amplification behavior
- latent defect
- temporary mitigation

## Constraints
- DevJams'26 nominal duration: 48 hours.
- Effective implementation budget: approximately 36 hours.
- Architecture process: HLD -> LLD -> contracts -> parallel implementation -> integration -> demo hardening.
- Replay must block real external side effects.
- Scope prioritizes a strong vertical slice over production completeness.
- Domain-neutral core with pluggable System Packs.

## First success milestone
Generate a known distributed failure, capture it, build a Replay Capsule, replay it, change one condition, and visualize how the causal path/outcome changes.
