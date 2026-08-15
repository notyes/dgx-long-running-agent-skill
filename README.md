# DGX Long-Running Agent Skills

A multi-skill repository for reusable Agent Skills focused on long-running local-LLM workflows, especially DGX Spark-class systems where slow decode speed makes repeated 100K+ context processing and compaction expensive.

## Available skills

### `dgx-long-running-agent`

Keeps long-running autonomous work productive by using bounded working context, persistent filesystem checkpoints, fresh-session recovery, context isolation, and delegated workers.

For ~128K models, the recommended operating ranges are:

```text
40K-60K    normal working context
60K-80K    heavy coding / cross-file reasoning
80K-95K    checkpoint aggressively
95K-110K   prefer a fresh session
110K-120K  avoid starting new large work
128K       safety ceiling only
```

The 60K-80K range is intentionally available for serious coding tasks that need several related source files, tests, configuration, project rules, and reasoning headroom. These ranges are ceilings for each mode, not targets that must be filled.

## Install with `skills`

List all skills published by this repository:

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill --list
```

Install interactively and choose the skill(s):

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill
```

Install one specific skill:

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill --skill dgx-long-running-agent
```

Install globally for a specific supported agent, for example Codex:

```bash
npx skills@latest add notyes/dgx-long-running-agent-skill \
  --skill dgx-long-running-agent \
  -g -a codex -y
```

## Repository layout

Each installable skill lives under `skills/<skill-name>/` and contains its own `SKILL.md`:

```text
skills/
└── dgx-long-running-agent/
    ├── SKILL.md
    ├── references/
    │   └── architecture.md
    └── templates/
        ├── TASK.md
        ├── STATE.md
        ├── TODO.md
        ├── DECISIONS.md
        └── HANDOFF.md
```

Additional skills can be added alongside it:

```text
skills/
├── dgx-long-running-agent/
│   └── SKILL.md
├── another-skill/
│   └── SKILL.md
└── another-skill-2/
    └── SKILL.md
```

Then users can discover and selectively install them with `--list` and `--skill <name>`.

## Design principle

Do not use the model context window as long-term memory.

Use enough context to reason correctly about the current dependency slice, allow heavy coding sessions to grow into the 60K-80K range when justified, persist durable state outside the model, checkpoint before the context becomes expensive, isolate noisy subtasks, and reconstruct only the context required for the next unit of work.
