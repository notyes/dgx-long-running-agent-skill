---
name: local-marathon
description: Manual long-running mode for local LLM work. Use ONLY when the user explicitly invokes /local-marathon. Never auto-trigger this skill from task type, duration, coding complexity, repository size, or context usage alone.
---

# Local Marathon

This skill is an explicit opt-in mode for long-running work on a local model, especially when generation is relatively slow and the model supports a large context window such as ~128K tokens.

## Activation Gate

**Do not activate automatically.**

Activate this skill only when the user's current request explicitly contains:

```text
/local-marathon
```

A long task, coding task, large repository, or high context usage is not sufficient by itself.

Once activated, keep Local Marathon active for the current user-directed task until:

- the task completes,
- the user explicitly stops/cancels it,
- the user explicitly disables Local Marathon,
- progress is blocked by something that requires user input.

A deliberate fresh context/session inside the same task should resume Local Marathon from `.agent-state/HANDOFF.md`; the user should not need to type `/local-marathon` again merely because context was rotated.

## Core Rule

> Treat the model context window as a safety ceiling, not as persistent memory.

Preserve durable state outside the model and reconstruct only the context needed for the current unit of work.

## Objectives

1. Keep working context bounded while allowing enough room for serious coding tasks.
2. Avoid expensive compaction near the model context limit.
3. Preserve progress across fresh sessions.
4. Isolate noisy investigations and large tool output.
5. Make completed work recoverable after process failure, model restart, or context reset.
6. Use filesystem state, git state, tests, and structured checkpoints as durable truth.
7. For coding changes, require an outsider-style scrutiny pass before completion.

## Recommended Context Budget

For a model with ~128K maximum context:

- Normal working context: 40K-60K tokens.
- Heavy coding / cross-file reasoning: 60K-80K tokens.
- Aggressive checkpoint zone: 80K-95K tokens.
- Prefer a fresh session: 95K-110K tokens.
- Avoid starting a new large work unit: 110K-120K tokens.
- 128K: emergency safety ceiling only.

These are operating ranges, not targets to fill. Use the smallest context that preserves enough code, tests, constraints, and state to reason correctly.

A representative heavy-coding allocation may look like:

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

Do not preload 80K just because it is available.

## Persistent Workspace

At the root of the working repository, maintain:

```text
.agent-state/
├── TASK.md
├── STATE.md
├── TODO.md
├── DECISIONS.md
├── HANDOFF.md
├── findings/
├── logs/
└── checkpoints/
```

Initialize from this skill's `templates/` directory when useful.

### TASK.md

Stable user goal, explicit constraints, acceptance criteria, and protected behavior/files/APIs.

### STATE.md

Concise durable state only:

- what is known
- what is completed
- what is currently broken
- relevant files/components
- current validation status
- latest scrutiny verdict when coding
- next best action

### TODO.md

Actionable remaining work:

- `[ ]` pending
- `[-]` in progress
- `[x]` completed
- `[!]` blocked

### DECISIONS.md

Record architectural decisions and non-obvious constraints that future sessions should not rediscover or accidentally reverse.

### HANDOFF.md

Minimum recovery packet for a fresh session. Include:

- overall goal
- completed work
- current focus
- files to read first
- commands/tests to run next
- what must not be repeated
- whether Local Marathon remains active
- latest scrutiny status if the current task changes code

## Coding Review Policy

Local Marathon integrates a scrutiny loop inspired by the outsider-review workflow in `thananon/9arm-skills`' `scrutinize` skill. The detailed procedure lives in:

```text
references/scrutiny-loop.md
```

The review has two stages:

1. **Intent sanity check before meaningful implementation.** Briefly question whether the change is necessary, whether an existing mechanism already solves it, whether a smaller solution would work, and whether the change belongs at the correct layer.
2. **Full scrutiny after implementation and relevant validation pass.** Trace the real code path end-to-end, verify claimed behavior and edge cases, inspect whether tests exercise the real path, and produce a verdict.

The full scrutiny verdict must be one of:

- `ship`
- `fix-then-ship`
- `rework`
- `reject`

For coding work, do not pass the Completion Gate unless the latest scrutiny verdict is `ship`, except when the user explicitly accepts a known unresolved finding.

Prefer running the final full scrutiny in a fresh worker/subagent context when practical so the reviewer is less anchored to the implementation reasoning.

## Execution Loop

For every activated Local Marathon task, follow this loop.

### 1. Initialize

Read only what is needed to recover the task:

1. user request / stable task definition
2. `.agent-state/TASK.md`
3. `.agent-state/STATE.md`
4. `.agent-state/TODO.md`
5. `.agent-state/DECISIONS.md`
6. `.agent-state/HANDOFF.md` if resuming

Do not immediately ingest the entire repository.

### 2. Select one bounded work unit

Choose the highest-value unblocked TODO item with:

- clear input files
- clear expected outcome
- a validation command or observable result
- limited scope

### 3. Run intent sanity check for meaningful code changes

Before implementing a new meaningful approach, briefly establish:

- the goal in one sentence
- why the change is needed
- whether a smaller/existing solution is preferable
- whether the chosen layer is appropriate

Record a non-obvious approach decision in `DECISIONS.md`.

Do not repeat this ceremony for every tiny edit when the same validated approach is continuing.

### 4. Load only relevant context

Prefer targeted reads/searches over whole-repository ingestion.

Load, in order:

1. stable task constraints
2. concise state/handoff
3. relevant source files
4. directly relevant tests/configuration
5. only logs needed for the current failure

For coding, several related implementation/test files may be loaded together when correctness depends on their interaction. Prefer one coherent dependency slice over unrelated files.

### 5. Execute

Perform the selected work unit.

For code tasks:

- inspect before editing
- make the smallest coherent change
- run narrow validation first
- expand validation after narrow validation succeeds

Do not create speculative changes solely to keep the agent busy.

### 6. Checkpoint immediately

Checkpoint after meaningful progress, including when:

- a root cause is identified
- a file is changed
- a test begins passing
- a test exposes a new failure
- an architectural decision is made
- an external dependency blocks work
- a work unit completes
- context enters the 80K-95K zone

Write conclusions, not raw conversation history.

### 7. Run full scrutiny for completed coding changes

After implementation and the relevant tests/validation pass, but before marking the coding unit complete:

1. Restate intent.
2. Ask once more whether a materially simpler solution is available.
3. Use the diff as an entry point, then trace the real execution path through changed and unchanged code.
4. Verify each important behavior claim against that path.
5. Check relevant edge/error/retry/concurrency/contract/performance cases.
6. Verify tests exercise the actual path rather than only mocked/intermediate state.
7. Produce findings with evidence and a verdict.

Follow `references/scrutiny-loop.md` for the detailed format.

If verdict is `fix-then-ship` or `rework`:

- add actionable findings to TODO.md
- write detailed findings under `.agent-state/findings/`
- update STATE.md with the verdict
- return to the implementation loop
- validate the fix
- run full scrutiny again

A `reject` verdict means stop the current implementation path, preserve the evidence/decision, and choose a new approach only if it remains within the user's goal.

A `ship` verdict allows the coding unit to proceed toward completion.

### 8. Decide whether to continue or reset

Continue in the same session when the next unit is tightly related and the current context remains useful.

At roughly 95K-110K, prefer writing HANDOFF.md and starting fresh unless finishing the current atomic unit is clearly cheaper.

At 110K-120K, do not begin another broad investigation or large code read. Preserve state and reset as soon as the current atomic step is safe.

## Fresh-Session Protocol

Before resetting:

1. Update STATE.md.
2. Update TODO.md.
3. Update DECISIONS.md if needed.
4. Write HANDOFF.md.
5. Record that `/local-marathon` remains active for the continuing task.
6. Record any pending or completed scrutiny verdict.
7. Save useful raw output under `.agent-state/logs/`.
8. Ensure code changes are visible in git status/diff.

On resume, read HANDOFF.md first, then only referenced state/source files.

## Context Isolation

Use subagents/workers when a subtask would otherwise inject substantial output or unrelated code into coordinator context.

Good worker tasks include:

- investigate one failing test
- inspect one subsystem
- trace one request flow
- review one patch
- analyze logs
- perform the final scrutiny pass

Each worker receives only task-specific objective, relevant constraints, source paths, and expected output.

Store detailed findings under:

```text
.agent-state/findings/<topic>.md
```

Workers return durable conclusions, not full transcripts.

## Large Output Handling

Do not keep large reproducible output in conversational context when it can be stored externally, including:

- build logs
- stack traces
- test reports
- dependency trees
- repository maps
- generated diffs
- benchmark output

Keep only relevant lines, conclusion, and file path in working context.

## Compaction Policy

Use this priority order:

1. Drop irrelevant/reproducible tool output.
2. Write large results to files.
3. Isolate work in subagents.
4. Summarize durable conclusions only.
5. Fresh session + restore from `.agent-state/`.
6. Full conversational compaction only when reset is impossible.

Never wait until context is almost full before preserving state.

## Git as Durable State

For code tasks, git is part of the memory system:

- use `git status` for changed files
- use `git diff` to reconstruct edits
- use small commits/checkpoints when the user's workflow permits
- use branches to isolate experiments

Do not copy a large diff into STATE.md. Record why the change exists and point to affected paths.

## vLLM Optimization Guidance

When serving through vLLM, reuse stable prompt prefixes when possible so prefix caching can reduce repeated prefill work.

Keep stable material before volatile material where the runtime permits:

```text
system instructions
stable tool definitions
stable project rules
concise persistent state
current work unit
volatile tool output
```

Prefix caching can improve prompt/prefill latency but does not solve slow decode throughput.

## Failure Recovery

After crash/restart/model change:

1. Inspect git status/diff.
2. Read TASK.md.
3. Read HANDOFF.md.
4. Read STATE.md and TODO.md.
5. Confirm Local Marathon was active for the same task.
6. Restore pending scrutiny status.
7. Verify important prior validation if needed.
8. Resume the next explicit action.

## Completion Gate

Do not declare the task complete merely because TODO.md is empty.

Before completion:

1. Re-read TASK.md acceptance criteria.
2. Run final relevant validation.
3. For code changes, confirm the latest full scrutiny verdict is `ship` or the user explicitly accepted the remaining finding.
4. Confirm no `[!]` blockers remain unresolved.
5. Confirm the real code path and tests support the claimed result.
6. Summarize actual changes/results.
7. Update STATE.md with final status.
8. Mark HANDOFF.md complete or remove stale next-action instructions.

## Anti-Patterns

Avoid:

- auto-activating without `/local-marathon`
- using the whole 128K merely because it exists
- preloading 80K when a smaller dependency slice is enough
- repeatedly rereading the whole repository
- storing raw logs in STATE.md
- carrying worker transcripts into coordinator context
- postponing checkpointing until context is nearly full
- starting a large new work unit above ~110K
- reviewing only the diff without tracing surrounding code
- treating passing tests as proof that the intended code path was exercised
- rubber-stamping the agent's own implementation
- trusting conversational memory over filesystem/git state
- redoing investigations already captured in findings/DECISIONS.md
- allowing multiple workers to modify the same files concurrently without coordination

## Default Decision Rule

When unsure whether to keep working in the current context or start fresh:

> Preserve state first. Use enough context to reason correctly, and use a fresh outsider review before shipping code when practical.
