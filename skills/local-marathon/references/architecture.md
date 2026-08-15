# Architecture Reference

## Recommended topology

```text
User explicitly invokes /local-marathon
   |
   v
Coordinator (bounded working context)
   |
   +-- Worker: code investigation
   +-- Worker: tests/log analysis
   +-- Worker: final scrutiny review
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

## Activation model

Local Marathon is explicit opt-in only. A long task, coding task, large repository, or high context usage must not activate the skill automatically. The user's request must contain `/local-marathon`.

Once activated for a task, deliberate context rotation may resume from `.agent-state/HANDOFF.md` without requiring the user to type the command again, because it is still the same user-directed task.

## Context lifecycle

```text
/local-marathon
  -> initialize durable state
  -> restore concise state when resuming
  -> select one bounded unit
  -> lightweight intent sanity check for meaningful code changes
  -> load relevant dependency slice
  -> execute
  -> validate/tests
  -> checkpoint
  -> full outsider scrutiny
       -> question intent/simpler alternative
       -> trace actual code path end-to-end
       -> verify claims, edges, contracts, tests
       -> verdict
  -> if fix/rework: return to implementation loop
  -> if ship: complete unit
  -> continue while context remains useful
  -> otherwise write handoff and reset
  -> resume same marathon task
```

## Why scrutiny is placed twice

A small intent check before implementation is cheap insurance against spending a slow local-model session building the wrong approach.

The expensive review belongs after implementation and validation because the reviewer can inspect the real change, trace its actual path through unchanged code, and verify whether the tests exercise the behavior that matters.

The final scrutiny pass should preferably use a fresh worker/subagent context when practical. That gives the review a more outsider-like perspective and reduces confirmation bias from the agent that authored the implementation.

The full review procedure is documented in `scrutiny-loop.md`.

## Completion rule for code

```text
implementation
  -> tests pass
  -> scrutiny
       -> ship            => completion gate allowed
       -> fix-then-ship   => TODO + fix + validate + scrutinize again
       -> rework          => replace approach + validate + scrutinize again
       -> reject          => stop current approach and preserve rationale
```

Passing tests alone is not sufficient for a coding completion claim. The final review should trace the real code path and confirm that the tests actually exercise it.

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

Above ~80K, preserve progress more aggressively. Above ~95K, compare the cost of keeping historical context with reconstructing from durable state. Above ~110K, avoid beginning a broad new investigation and finish or checkpoint the current atomic step before resetting.

These are operational defaults, not hard model limits. Tune them using real prompt-processing latency, prefix-cache hit rate, KV-cache capacity, task type, and serving concurrency.
