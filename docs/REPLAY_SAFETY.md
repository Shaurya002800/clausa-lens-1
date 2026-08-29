# Replay Safety

Replaying captured incident traffic can repeat payments, messages, notifications, or writes. CausaLens therefore treats isolation as a product feature and a hard execution gate, not an operational suggestion.

## Default-deny policy

A replay starts with no production credentials, no production data-store routes, and no general outbound network access. Access is granted only to replay-scoped services or explicitly allow-listed simulators.

## Preflight checks

Before creating a replay runtime, the validator must confirm:

1. the capsule schema and integrity digest are valid,
2. the System Pack and failure oracle versions are supported,
3. all captured inputs and fixtures are marked sanitized,
4. production database, queue, cache, webhook, and API destinations are absent,
5. only replay-specific credentials are referenced,
6. every required external dependency resolves to an approved simulator, and
7. the reset strategy targets replay-only state.

Failure blocks the run before any service starts.

## Runtime controls

- Run each replay in a clean, separately identified namespace or process boundary.
- Block uncontrolled outbound connections.
- Route payment, email, webhook, SMS, and similar dependencies to simulators.
- Write only to replay-scoped databases, queues, caches, and file storage.
- Record every attempted dependency interaction, including denied attempts.
- Terminate the replay on prohibited egress or a production-destination match.
- Tear down or reset replay state after the result is persisted.

## Sensitive data

- Remove secrets and production credentials during capture.
- Tokenize or replace personal and payment data with scenario-safe fixtures.
- Store content digests when the original value is unnecessary.
- Permit the UI to reveal only sanitized replay evidence.

## Isolation evidence

Each replay result records:

- runtime namespace identifier,
- capsule and System Pack versions,
- network policy result,
- credential profile identifier,
- data-store destinations used,
- simulator interactions,
- denied interactions,
- reset/teardown result, and
- overall isolation verdict.

The Command Center must show this evidence before presenting a replay as safely reproduced.

## Baseline gate

A baseline replay passes only when:

1. the run completed,
2. the isolation verdict passed,
3. no prohibited interaction occurred,
4. the configured failure oracle matched,
5. the expected effect and retry evidence was observed, and
6. the result is structurally comparable to the captured original within declared tolerances.

Failure at this gate prevents all what-if replays for that baseline.

## Failure classification

- Validation failure or unsafe configuration: status `BLOCKED`, no outcome.
- Prohibited runtime interaction: terminate with status `BLOCKED`, no outcome.
- Internal service, simulator, or fixture failure: status `FAILED`, no outcome.
- Safe completed baseline without the expected incident: status `COMPLETED`, outcome `NOT_REPRODUCED`.
- Safe completed baseline with the expected incident: status `COMPLETED`, outcome `REPRODUCED`.
- Safe completed what-if without the baseline failure: status `COMPLETED`, outcome `MITIGATED`.
- Safe completed what-if with the same failure: status `COMPLETED`, outcome `UNCHANGED`.

CausaLens must never assign an outcome to a blocked or failed run.
