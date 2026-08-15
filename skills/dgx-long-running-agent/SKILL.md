---
name: dgx-long-running-agent
description: Keep long-running local-LLM tasks productive on DGX Spark by using bounded working context, persistent filesystem checkpoints, fresh-session recovery, context isolation, and delegated workers instead of allowing a single 128K conversation to fill and compact repeatedly.
---

# DGX Long-Running Agent

Use this skill when a task must continue for many steps on a local model, especially when generation is relatively slow (for example 10-15 tokens/s) and the model supports a large context window such as ~128K tokens.

The core rule is:

> Treat the model context window as a safety ceiling, not as persistent memory.

Do not try to preserve the entire task history in one conversation. Preserve durable state outside the model and reconstruct only the context required for the next unit of work.

## Objectives

1. Keep working context bounded while allowing enough room for serious coding tasks.
2. Avoid expensive compaction near the model context limit.
3. Preserve progress across fresh sessions.
4. Separate workers so tool output and intermediate work do not pollute the coordinator context.
5. Make every completed step recoverable after process failure, model restart, or context reset.
6. Favor deterministic files, git state, test output, and structured checkpoints over conversational memory.

## Recommended Context Budget

For a model with ~128K maximum context, use these defaults unless runtime measurements show a better operating point:

- Normal working context: 40K-60K tokens.
- Heavy coding / cross-file reasoning: 60K-80K tokens.
- Aggressive checkpoint zone: 80K-95K tokens.
- Prefer a fresh session: 95K-110K tokens.
- Avoid starting a new large work unit: 110K-120K tokens.
- 128K: emergency safety ceiling only.

These are operational ranges, not targets that must be filled. Use the smallest context that preserves enough code, tests, constraints, and state to reason correctly.

For coding work, 60K-80K is intentionally allowed because a realistic session may need room for system/tool instructions, project rules, persistent state, several source files, tests/configuration, and enough free space for reasoning and tool output.

Example budget for a heavy coding session:

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

Do not preload 80K just because it is available. Grow toward the heavy-coding range only when cross-file reasoning genuinely benefits from it.

If exact token usage is unavailable, approximate growth from message size, large tool results, file reads, logs, and repeated source code.

## Persistent Workspace

At the root of the working repository, maintain a directory named `.agent-state/`:

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

Create it at the beginning of a long-running task if it does not exist.

### TASK.md

Contains the stable user goal, explicit constraints, acceptance criteria, and things that must not be changed.

Do not rewrite the original goal merely to make execution easier.

### STATE.md

Contains only the current durable state:

- what is known
- what is completed
- what is currently broken
- relevant files/components
- current validation status
- next best action

Keep this concise enough to load into every fresh session.

### TODO.md

Contains actionable remaining work. Use states such as:

- `[ ]` pending
- `[-]` in progress
- `[x]` completed
- `[!]` blocked

Break large items into units that can normally finish inside one bounded context session.

### DECISIONS.md

Record architectural decisions and non-obvious constraints that future workers must not rediscover or accidentally reverse.

Each important decision should include:

- decision
- reason
- affected paths/components
- date or checkpoint identifier when useful

### HANDOFF.md

This is the minimum recovery packet for the next fresh session. It should answer:

- What is the overall goal?
- What has already been done?
- What is currently being attempted?
- What should be done next?
- Which files should be read first?
- Which commands/tests should be run next?
- What must not be repeated?

Keep HANDOFF.md smaller than STATE.md when possible.

## Execution Loop

For every long-running task, follow this loop.

### 1. Initialize

Read:

1. user request / stable task definition
2. `.agent-state/TASK.md`
3. `.agent-state/STATE.md`
4. `.agent-state/TODO.md`
5. `.agent-state/DECISIONS.md`
6. `.agent-state/HANDOFF.md` if resuming

Do not immediately read the entire repository.

### 2. Select one bounded work unit

Choose the highest-value unblocked TODO item that can be completed without loading unrelated project history.

A good unit has:

- clear input files
- clear expected outcome
- a validation command or observable result
- limited scope

### 3. Load only relevant context

Prefer targeted reads and searches over full-file or full-repository ingestion.

Load, in order:

1. stable task constraints
2. concise state/handoff
3. relevant source files
4. directly relevant test/config files
5. only the logs needed for the current failure

Avoid reloading old tool output when its durable conclusion already exists in STATE.md or findings files.

For coding tasks, it is acceptable to load several related implementation and test files when the relationship between them is essential. Prefer one coherent dependency slice over many unrelated files.

### 4. Execute

Perform the work for the selected unit.

For code tasks:

- inspect before editing
- make the smallest coherent change
- run the most relevant narrow validation first
- expand validation only after narrow validation succeeds

Do not create speculative changes solely to keep the agent busy.

### 5. Checkpoint immediately

After any meaningful progress, update durable state before continuing.

Checkpoint when any of these happens:

- a bug root cause is identified
- a file is changed
- a test begins passing
- a test exposes a new failure
- an architectural decision is made
- an external dependency blocks work
- the current work unit completes
- context enters the 80K-95K checkpoint zone

Write conclusions, not raw conversation history.

### 6. Decide whether to continue or reset

Continue in the same session when:

- the next work unit is tightly related
- current code context remains useful
- large tool outputs are not accumulating unnecessarily
- context remains below the preferred fresh-session zone

Once context reaches roughly 95K-110K, prefer writing HANDOFF.md and starting a fresh session unless finishing the current bounded unit is clearly cheaper than resetting.

At 110K-120K, do not start another large investigation or broad code read. Preserve state and reset as soon as the current atomic step is safe.

## Fresh-Session Protocol

Before resetting context:

1. Update STATE.md.
2. Update TODO.md.
3. Update DECISIONS.md if needed.
4. Write HANDOFF.md.
5. Save useful raw output under `.agent-state/logs/` instead of embedding it into HANDOFF.md.
6. Ensure all code changes are visible in git diff/status.

A new session should reconstruct state from files, not from a summary of the entire previous conversation.

On resume, read HANDOFF.md first, then only the referenced state/source files.

## Context Isolation

Use subagents/workers when a subtask would otherwise inject substantial tool output or unrelated source code into the coordinator context.

Good worker tasks:

- investigate one failing test
- inspect one subsystem
- trace one request flow
- review one patch
- research one dependency
- analyze one log bundle

Each worker receives only:

1. task-specific objective
2. relevant constraints
3. relevant source paths
4. expected output format

Workers should return durable conclusions, not full transcripts.

Store detailed findings in:

```text
.agent-state/findings/<topic>.md
```

The coordinator should consume the concise conclusion and file path.

## Coordinator Behavior

The coordinator should be efficient in context and reasoning whenever possible.

Its responsibilities are:

- select the next work unit
- delegate isolated investigations
- merge conclusions
- update durable state
- decide when to checkpoint/reset
- verify acceptance criteria

The coordinator should not duplicate a worker's detailed investigation unless verification is required.

## Large Output Handling

Never keep large, reproducible output in conversation context when it can be stored externally.

Examples:

- build logs
- stack traces
- test reports
- dependency trees
- repository maps
- generated diffs
- benchmark output

Store them under `.agent-state/logs/` and place only:

- relevant error lines
- conclusion
- file path

into working context.

## Compaction Policy

Compaction is a fallback, not the primary memory strategy.

Use this priority order:

1. Drop irrelevant/reproducible tool output.
2. Write large results to files.
3. Isolate work in subagents.
4. Summarize only durable conclusions.
5. Fresh session + restore from `.agent-state/`.
6. Use full conversational compaction only when a reset is impossible.

Never wait until the context is almost full before preserving state.

## Git as Durable State

For code tasks, git is part of the memory system.

Use:

- `git status` to identify changed files
- `git diff` to reconstruct current edits
- small commits/checkpoints when the user's workflow permits
- branches to isolate experimental work

Do not duplicate a large diff into STATE.md. Record why the change exists and point to the affected files.

## vLLM Optimization Guidance

When serving through vLLM, reuse stable prompt prefixes when possible so prefix caching can reduce repeated prefill work.

Keep stable material ordered before volatile material where the client/framework allows it:

```text
system instructions
stable tool definitions
stable project rules
concise persistent state
current work unit
volatile tool output
```

Prefix caching can improve prompt/prefill latency but does not solve slow decode throughput. The context-management architecture remains necessary even when prefix caching is enabled.

## Failure Recovery

After crash/restart/model change:

1. Inspect git status/diff.
2. Read TASK.md.
3. Read HANDOFF.md.
4. Read STATE.md and TODO.md.
5. Verify the last claimed validation result if important.
6. Resume the next explicit action.

Do not restart analysis from zero unless durable state is inconsistent or missing.

## Completion Gate

Do not declare the task complete merely because TODO.md is empty.

Before completion:

1. Re-read TASK.md acceptance criteria.
2. Run final relevant validation.
3. Confirm no `[!]` blockers remain unresolved.
4. Summarize actual changes/results.
5. Update STATE.md with final status.
6. Mark HANDOFF.md as complete or remove stale next-action instructions.

## Anti-Patterns

Avoid:

- using the entire 128K context merely because it exists
- preloading 80K of material when a smaller dependency slice is enough
- repeatedly rereading the whole repository
- storing raw logs in STATE.md
- carrying worker transcripts into coordinator context
- postponing checkpointing until context is nearly full
- starting a large new work unit above ~110K
- trusting conversation memory over filesystem/git state
- redoing investigations already captured in findings/DECISIONS.md
- allowing multiple workers to modify the same files concurrently without coordination

## Default Decision Rule

When unsure whether to keep working in the current context or start fresh:

> Preserve state first. Use enough context to reason correctly, but prefer a fresh bounded context before historical context approaches the ceiling.
