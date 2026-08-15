# Architecture Reference

## Recommended topology

```text
User Task
   |
   v
Coordinator (small bounded context)
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
  -> load relevant files
  -> execute + validate
  -> checkpoint
  -> continue if context is still cheap
  -> otherwise write handoff and reset
```

## Why this helps on DGX Spark

With comparatively slow token generation, an autonomous agent can lose substantial wall-clock time when it repeatedly operates near a very large context ceiling or invokes heavy compaction. External durable state allows the next request to begin with a much smaller prompt while preserving task continuity.

The design separates three forms of memory:

1. **Working memory** — current model context; intentionally disposable.
2. **Task memory** — `.agent-state/` files; concise and durable.
3. **Repository truth** — source code, git diff/history, tests, logs; authoritative evidence.

## Suggested budgets for ~128K models

```text
10K-30K   normal working range
30K-40K   checkpoint aggressively
40K-60K   prefer a fresh session
128K      safety ceiling only
```

These are operational defaults, not hard model limits. Tune them using real prompt-processing latency, KV-cache capacity, task type, and serving concurrency.
