# High-Level Design

## Simplified HLD

```text
+---------------------------+
| Distributed Demo System   |
+-------------+-------------+
              |
              | execution evidence
              v
+---------------------------+
| Capture & Evidence Layer  |
+-------------+-------------+
              |
              | normalized events
              v
+---------------------------+
| Propagation Graph Builder |
+-------------+-------------+
              |
              | incident graph
              v
+---------------------------+
| Replay Capsule Compiler   |
+-------------+-------------+
              |
              | immutable capsule
              v
+---------------------------+
| Isolated Replay Runtime   |
+------+------+-------------+
       |      |
       |      +--------------------+
       |                           |
       v                           v
+-------------+             +----------------+
| Baseline    |             | Counterfactual |
| Replay      |             | Replay         |
+------+------+             +-------+--------+
       |                            |
       +-------------+--------------+
                     |
                     v
           +----------------------+
           | Differential Analyzer|
           +----------+-----------+
                      |
                      v
           +----------------------+
           | Causal Explanation   |
           | + Visualization      |
           +----------------------+
```

## Key boundaries

### Capture boundary
Capture enough evidence for replay while avoiding unnecessary full-system snapshots.

### Capsule boundary
A Replay Capsule is the handoff between incident reconstruction and replay execution.

### Isolation boundary
Everything beyond the replay runtime boundary must be controlled, mocked, intercepted, namespaced, or explicitly allow-listed.

### Experiment boundary
Each counterfactual run changes one variable so the resulting differential remains interpretable.

## HLD-to-LLD transition
Before parallel coding begins, freeze:
- event schema,
- graph node/edge model,
- Replay Capsule schema,
- replay run contract,
- intervention contract,
- result/diff contract,
- System Pack interface.
