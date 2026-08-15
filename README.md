# DGX Long-Running Agent Skills

A multi-skill repository for reusable Agent Skills focused on long-running local-LLM workflows, especially DGX Spark-class systems where slow decode speed makes repeated 100K+ context processing and compaction expensive.

## Available skills

### `local-marathon`

Manual long-running mode for local LLM work. It is intentionally **command-only** and must not auto-trigger.

Activate it explicitly with:

```text
/local-marathon
```

Example:

```text
/local-marathon fix all failing tests and keep going until the acceptance criteria pass
```

A long task, coding task, large repository, or high context usage by itself must **not** activate this skill.

Once `/local-marathon` is active for a user-directed task, the skill may deliberately rotate to a fresh context and resume from `.agent-state/HANDOFF.md` without requiring the user to type the command again for that same continuing task.

## Coding flow

Local Marathon now includes an outsider-style scrutiny loop for code changes:

```text
/local-marathon
  -> intent sanity check
  -> implement bounded work unit
  -> tests / validation
  -> full scrutiny pass
       -> question whether a simpler approach exists
       -> trace the actual code path end-to-end
       -> verify behavior, edge cases, contracts, and tests
       -> verdict: ship / fix-then-ship / rework / reject
  -> fix/rework loops back to implementation
  -> ship proceeds to completion gate
```

The final scrutiny pass should preferably run in a fresh worker/subagent context when practical to reduce confirmation bias from the implementation session.

Passing tests alone is not considered sufficient evidence that a code task is complete; the final review should confirm that the real execution path produces the intended behavior and that tests exercise that path.

Detailed review rules live in `skills/local-marathon/references/scrutiny-loop.md`.

For ~128K models, the recommended operating ranges are:

```text
40K-60K    normal working context
60K-80K    heavy coding / cross-file reasoning
80K-95K    checkpoint aggressively
95K-110K   prefer a fresh session
110K-120K  avoid starting new large work
128K       safety ceiling only
```

The 60K-80K range is intentionally available for serious coding tasks that need several related source files, tests, configuration, project rules, and reasoning headroom. These ranges are operating zones, not targets that must be filled.

## Install with `skills`

List all skills published by this repository:

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill --list
```

Install interactively and choose the skill(s):

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill
```

Install Local Marathon directly:

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill --skill local-marathon
```

Install globally for a specific supported agent, for example Codex:

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill \
  --skill local-marathon \
  -g -a codex -y
```

## Repository layout

```text
skills/
└── local-marathon/
    ├── SKILL.md
    ├── references/
    │   ├── architecture.md
    │   └── scrutiny-loop.md
    └── templates/
        ├── TASK.md
        ├── STATE.md
        ├── TODO.md
        ├── DECISIONS.md
        └── HANDOFF.md
```

Additional skills can be added alongside it and selectively installed with `--list` and `--skill <name>`.

## Design principle

Do not use the model context window as long-term memory.

Use enough context to reason correctly about the current dependency slice, allow heavy coding sessions to grow into the 60K-80K range when justified, persist durable state outside the model, checkpoint before the context becomes expensive, isolate noisy subtasks, and reconstruct only the context required for the next unit of work.
