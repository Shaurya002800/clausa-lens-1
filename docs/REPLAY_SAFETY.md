# Replay Safety

Replaying production incident traffic directly is unsafe. CausaLens therefore treats isolation as a core architectural requirement.

## Safety rules
- Run replays in a separate namespace/environment.
- Block uncontrolled outbound network access.
- Replace payment, email, webhook, SMS, and similar side-effecting dependencies.
- Use replay-specific credentials where credentials are unavoidable.
- Prevent writes to production databases, queues, and caches.
- Sanitize or tokenize sensitive captured values where possible.
- Allow-list explicitly approved replay dependencies.
- Record every dependency interaction performed during replay.

## Safety gate
A capsule should not be eligible for counterfactual experiments until a baseline replay has passed a reproducibility and isolation gate.

The gate should answer:
1. Was the original failure reproduced?
2. Were all external side effects contained?
3. Were unexpected external dependencies observed?
4. Is the replay result sufficiently comparable to the captured incident?

Failure at the gate stops experimentation.
