# DGX Long-Running Agent Skills

A multi-skill repository for reusable Agent Skills focused on long-running local-LLM workflows, especially DGX Spark-class systems where slow decode speed makes repeated 100K+ context processing and compaction expensive.

## Available skills

### `dgx-long-running-agent`

Keeps long-running autonomous work productive by using bounded working context, persistent filesystem checkpoints, fresh-session recovery, context isolation, and delegated workers.

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

Use a bounded working context, persist durable state outside the model, checkpoint continuously, isolate noisy subtasks, and reconstruct only the context required for the next unit of work.
