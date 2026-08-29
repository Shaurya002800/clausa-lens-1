# Replay Capsule

A Replay Capsule is an immutable, versioned artifact containing the minimum controlled information required to safely reproduce one supported incident.

It is not a full production snapshot. It contains only sanitized inputs, fixtures, policies, dependency behavior, and verification rules required by the selected System Pack.

## Version 1 contract authority

The sole normative `ReplayCapsule` v1.0 field set, nested types, required/nullable rules, validation behavior, integrity rules, and golden example are frozen in [CONTRACTS.md](CONTRACTS.md#replaycapsule-v10).

This document explains capsule invariants and compiler responsibilities. It must not introduce alternate field names or a second schema.

## Required semantics

### Source and provenance

The capsule must identify the incident, trace, capture environment class, System Pack version, and schema version that produced it. This allows a replay to reject incompatible or untraceable input.

### Trigger

The trigger contains the sanitized request or message that starts the supported replay scenario. Secrets, production credentials, and uncontrolled destination URLs must never be copied directly.

### Events and graph

Events conform to the canonical `ExecutionEvent` contract. The graph includes only the event identifiers and typed relationships required to reconstruct the supported execution path.

### State fixtures

State fixtures contain or reference the minimum replay-only records needed by the scenario. Every fixture declares its source type, sanitization status, content digest, and reset behavior.

### Dependency fixtures

Every external interaction required for replay resolves to a controlled simulator response. A fixture declares the request match, response, latency, failure behavior, and invocation limit.

### Timing policy

The timing policy distinguishes exact controlled delays from comparison tolerances. Wall-clock timestamps are not expected to match the original execution byte for byte.

### Replay plan

The replay plan declares the supported entrypoint, required service sequence, fixture-loading order, and clean-reset strategy. Arbitrary scripts embedded in captured input are not permitted.

### Failure oracle

The failure oracle specifies what must be observed for reproduction. For the golden scenario it checks that the checkout retry occurred and the same logical operation created two ledger effects.

### Allowed interventions

The capsule lists intervention types permitted by the active System Pack. A run may select zero interventions for baseline or exactly one for what-if; it may not add a new intervention type at runtime.

### Safety metadata

Safety metadata proves that captured values were sanitized and identifies default-denied and explicitly allow-listed destinations. A replay-only credential profile is referenced, never embedded.

### Integrity

The digest covers all replay-relevant capsule content. Validation must fail if content changes after compilation.

## Immutability model

- A compiled capsule never changes in place.
- A baseline run references the capsule with no intervention.
- A what-if run references the same capsule plus one separate intervention record.
- If fixtures, policies, or the failure oracle change, the compiler creates a new capsule with a new identifier and provenance.

## Validation gate

A capsule is `VALID` only when:

1. its schema is supported,
2. its integrity digest matches,
3. its System Pack version is available,
4. every required fixture resolves,
5. the trigger and fixtures are sanitized,
6. its replay plan is supported,
7. its failure oracle is executable, and
8. its isolation policy contains no production destination or credential.

Any failed condition blocks replay with a specific machine-readable and human-readable reason.

## MVP minimization rule

Minimization is explicit and scenario-driven for the hackathon. The duplicate-effect System Pack selects the known checkout inputs, ledger fixtures, payment behavior, timing, and oracle evidence. Automated general-purpose state minimization is future work.
