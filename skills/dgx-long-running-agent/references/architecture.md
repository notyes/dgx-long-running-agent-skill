# Architecture Reference

## Recommended topology

```text
User Task
   |
   v
Coordinator (bounded working context)
   |
   +-- Worker: code investigation
   +-- Worker: tests/log analysis
   +-- Worker: review/research
   |
   v
.agent-state/
   |
   +-- TASK.md
   +-- STATE.md
   +-- TODO.md
   +-- DECISIONS.md
   +-- HANDOFF.md
   +-- findings/
   +-- logs/
   |
   v
Fresh coordinator session when context grows
```

## Context lifecycle

```text
Fresh session
  -> restore concise state
  -> select one bounded unit
  -> load relevant dependency slice
  -> execute + validate
  -> checkpoint
  -> continue while context remains useful
  -> otherwise write handoff and reset
```

## Why this helps on DGX Spark

With comparatively slow token generation, an autonomous agent can lose substantial wall-clock time when it repeatedly operates near a very large context ceiling or invokes heavy compaction. External durable state allows the next request to begin with a smaller prompt while preserving task continuity.

The design separates three forms of memory:

1. **Working memory** — current model context; intentionally disposable.
2. **Task memory** — `.agent-state/` files; concise and durable.
3. **Repository truth** — source code, git diff/history, tests, logs; authoritative evidence.

## Suggested budgets for ~128K models

```text
40K-60K    normal working context
60K-80K    heavy coding / cross-file analysis
80K-95K    checkpoint aggressively
95K-110K   prefer a fresh session
110K-120K  avoid starting a new large work unit
128K       safety ceiling only
```

The 60K-80K range is useful when coding correctness depends on several related source files, tests, configuration, interface definitions, and project rules being visible together. It is not a target to fill automatically.

A representative heavy-coding allocation can look like:

```text
system + tools        ~10K
project rules          ~5K
state / handoff        ~5K
source code           ~30K
tests / config        ~10K
working headroom      ~20K
--------------------------
total                 ~80K
```

Above ~80K, preserve progress more aggressively. Above ~95K, consider the cost of keeping historical context versus reconstructing from durable state. Above ~110K, avoid beginning a broad new investigation and finish or checkpoint the current atomic step before resetting.

These are operational defaults, not hard model limits. Tune them using real prompt-processing latency, prefix-cache hit rate, KV-cache capacity, task type, and serving concurrency.
