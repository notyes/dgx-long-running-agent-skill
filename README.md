# DGX Long-Running Agent Skill

A reusable Agent Skill for long-running tasks on local LLMs, especially DGX Spark-class systems where decode speed can make repeated 100K+ context processing and compaction expensive.

## Main idea

Do not use the model context window as long-term memory.

Use a bounded 10K-30K working context, persist durable state to `.agent-state/`, checkpoint continuously, isolate noisy subtasks, and restart into a fresh context before reaching the model's context ceiling.

## Files

- `SKILL.md` — behavior/instructions for the agent.
- `templates/` — starter persistent-state files.
- `references/architecture.md` — architecture and lifecycle reference.

## Install

Copy this directory into the skills directory supported by your agent runtime, keeping `SKILL.md` at the skill root.

The skill itself is framework-agnostic. It can be adapted for Hermes, Claude/Codex-style skills, or a custom orchestrator in front of LiteLLM/vLLM.
