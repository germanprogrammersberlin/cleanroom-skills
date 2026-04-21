---
name: anthropic-compiler-pattern
description: Architecture pattern from Anthropic's C-Compiler project. 16 Claude instances, Git as shared state, failing tests as task queue, emergent prioritization. Reference material for designing multi-agent production systems.
---

# Anthropic C-Compiler Pattern

In early 2026, Anthropic built a C compiler using 16 Claude instances working in parallel. The architecture is documented at: https://www.anthropic.com/engineering/building-c-compiler

This skill is a reference — study it and decide what applies to your work.

## Core Architecture

- **16 stateless Claude instances** working on the same Git repo
- **Git as shared state** — all agents commit to the same repo, everyone sees what others did
- **Failing tests as task queue** — each failing test is a task, agents pick tasks and fix them
- **No central coordinator** — the environment steers, not an orchestrator
- **Lock files for task claiming** — agent writes a lock file, others skip that task

## Key Insight: Emergent Prioritization

The agents had no prescribed order. All tests existed simultaneously, all failing. The prompt said only: "Fix the next problem."

Despite this, agents naturally picked the easiest tasks first — not because they were told to, but because the model inherently chooses the most approachable problem. A "unknown function: printf" error is obviously easier than a cascading kernel error. The difficulty progression (easy → hard) emerged naturally.

This works because the model has domain knowledge. It was trained on compiler code, papers, textbooks. It knows what a parser is, what a good Results section is. The environment just needs to show: what is missing and where is the data.

## Phases

- **0-80%**: Generic agents, broad strokes, many easy fixes
- **80-100%**: Specialized roles emerge — deduplication, performance, style, documentation
- The last 20% required more targeted expertise

## Principles

1. **Environment design over orchestration** — design the workspace, not the coordination
2. **Stateless sessions** — each run starts fresh, reads current state, no accumulated context
3. **Clean data** — the workspace must clearly show what exists, what is missing, what is broken
4. **Simple prompts** — "Fix the next problem. Break it into small pieces. Keep going until it's good."
5. **Quality signal** — there must be a clear signal of what is wrong (tests, reviews, findings)

## Applicability

This pattern was designed for code compilation but the principles may apply to other production tasks:
- Any work where quality can be measured (tests, reviews, scores)
- Any work where multiple specialists can operate on a shared artifact
- Any work where the artifact improves incrementally

Read the original blog post for full context before applying these ideas.
