# Supercode delegation

A portable [Agent Skill](https://agentskills.io) for orchestrating non-trivial coding work (features, refactors, bug fixes) through isolated `product_workflow` agents. The parent session plans and orchestrates; research, implementation, review, fixes, and verification all run in delegated agents, with a structured verdict at the end.

## Prerequisites

- An Agent Skills-compatible harness
- The custom `product_workflow` tool
- Model identifiers mapped to models available in your harness's live catalog
- Optional `subagent_spawn` for one small delegation
- Optional `product_workflow_save` for persisting a stable workflow

Tool installation is separate. This repository contains skill instructions only.

## Install

Clone into your harness's skills directory. For Pi:

```sh
git clone https://github.com/gostak-dd/supercode-delegation.git /tmp/supercode-delegation
cp -r /tmp/supercode-delegation/supercode-delegation ~/.pi/agent/skills/
```

Then run:

```text
/reload
/skill:supercode-delegation <task>
```

A compatible Agent Skills directory such as `~/.agents/skills` or `~/.claude/skills` works too.

## Model mapping

The skill ships with example model identifiers. Check your harness's live catalog and map each one (adding any provider prefix your deployment needs) before running the workflow template:

| Example identifier | Role | Effort |
|---|---|---|
| `gpt-6.1-terra` | Research | high |
| `gpt-6.1-sol` | Implementation and first fix | high |
| `claude-opus-5-5` | Ambiguous implementation and second fix | high |
| `claude-fable-5-2` | Review and hard escalation | high |
| `gpt-6-astra` | Optional second review | high |
| `gpt-6-luna` | Large mechanical scans | medium |

Set `access.models` explicitly for every workflow after mapping.

## Contents

- `supercode-delegation/SKILL.md`: skill instructions, orchestrator rules, and a compact implementation workflow template (definition + script)

## License

MIT
