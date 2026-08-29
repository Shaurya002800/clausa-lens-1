# Replay Capsule

A Replay Capsule is an immutable artifact containing the minimum information required to safely reproduce an incident.

## Candidate contents
- initiating request/message
- relevant downstream messages
- normalized execution events
- selected state snapshots or fixtures
- configuration/policies
- dependency responses or simulations
- timing metadata
- failure conditions
- propagation relationships
- System Pack identifier/version
- replay instructions
- integrity/version metadata

## Required properties

### Minimal
Avoid copying unrelated system state.

### Immutable
Once compiled, the capsule used for an experiment should not mutate.

### Versioned
Schemas, System Packs, and replay policies must be identifiable.

### Safe
Secrets and unsafe external destinations should not be embedded without sanitization or indirection.

### Reproducible
The capsule should contain or reference all required controlled inputs.

## Conceptual schema

```text
ReplayCapsule
├── metadata
├── trigger
├── events[]
├── graph
├── state[]
├── dependencies[]
├── policies
├── failure_conditions[]
├── replay_plan
└── integrity
```
