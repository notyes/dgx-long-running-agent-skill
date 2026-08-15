# Scrutiny Loop

This review stage is adapted from the outsider-review ideas in `thananon/9arm-skills`' `scrutinize` skill and integrated directly into Local Marathon so no separate skill is required.

## Purpose

Do not stop at "the code compiles" or "tests pass". Before completing a coding change, review it from an outsider perspective and verify the real behavior end-to-end.

## Two-stage review

### Stage A — pre-implementation intent sanity check

Before a meaningful implementation unit, briefly answer:

1. What is the goal in one sentence?
2. Is the requested change actually necessary?
3. Is there a smaller, simpler, or existing mechanism that achieves the goal with less risk?
4. Is the change being made at the correct layer?

This pass should be short. Its purpose is to avoid spending a long local-LLM session implementing the wrong shape of solution.

Skip re-running this for every tiny edit when the same approach has already been validated and recorded in `DECISIONS.md`.

### Stage B — full post-implementation scrutiny

Run after implementation and relevant tests/validation pass, but before the task or work unit is declared complete.

#### 1. Restate intent

State what the change is supposed to accomplish in one sentence.

Then ask again whether a materially simpler solution became apparent after seeing the implementation.

#### 2. Trace the actual path

Use the diff only as the entry point. Follow the real execution path through unchanged code as needed:

```text
entry point
  -> caller(s)
  -> branches / guards
  -> changed code
  -> downstream calls
  -> state mutation / side effects
  -> return / error / external effect
```

For each claimed behavior, identify the actual path that produces it.

Pay special attention to seams between changed and unchanged code.

#### 3. Verify claims

For every important behavior introduced or changed, verify:

- the traced path actually produces the intended behavior
- error and fallback paths
- null / empty / malformed / boundary inputs where relevant
- retries, concurrency, ordering, or partial failures where relevant
- contract changes for callers or external systems
- performance or observability changes that may be silent
- tests exercise the real path rather than only mocked/intermediate behavior

Do not invent theoretical issues without evidence. Tie every finding to a concrete path, condition, file, test, or reproducible scenario.

#### 4. Produce findings and verdict

Order findings by severity:

```text
blocker
major
minor
nit
```

For each meaningful finding record:

- Finding — specific issue
- Why it matters — consequence
- Evidence — traced path / condition / test gap
- Suggested change — smallest concrete fix

End with exactly one of these verdicts:

- `ship` — no material issue remains
- `fix-then-ship` — targeted fixes are required
- `rework` — approach is structurally wrong or unnecessarily risky
- `reject` — change should not proceed as designed

## Marathon integration

If verdict is `fix-then-ship` or `rework`:

1. add findings to `.agent-state/TODO.md`
2. store detailed review notes in `.agent-state/findings/scrutiny-<topic>.md`
3. update STATE.md with the current verdict
4. return to the implementation loop
5. run targeted validation
6. run scrutiny again on the resulting code path

Do not mark the coding task complete until the latest full scrutiny verdict is `ship`, unless the user explicitly accepts a known unresolved finding.

## Context-cost rule

The scrutiny pass may be performed in a fresh worker/subagent when the implementation context is large. Give the reviewer the stable goal, constraints, git diff, relevant source paths, tests, and enough surrounding code to trace the real path.

Prefer an independent/fresh context for the final scrutiny pass when practical; it reduces confirmation bias from the agent that authored the change.
