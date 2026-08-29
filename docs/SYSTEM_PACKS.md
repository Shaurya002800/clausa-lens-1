# System Packs

System Packs keep CausaLens domain-neutral.

A System Pack contains the knowledge required to adapt the core replay framework to a particular failure domain.

## Responsibilities
A pack may define:
- evidence adapters
- domain event normalization
- relevant state extraction
- dependency mocks/simulators
- invariants
- approved intervention types
- outcome evaluation
- visualization hints

## Initial candidate packs
- duplicate payment
- queue/job redelivery
- retry storm
- stale cache
- notification fan-out
- dependency latency
- idempotency/deduplication

## Design principle
The CausaLens core owns replay mechanics and experiment orchestration. System Packs own domain interpretation.
