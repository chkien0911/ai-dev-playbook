# Agent Audit — Execution Guide

Reference for Phase 5 (deletion/merge review) and Phase 6 (file writing).
Read this when the user has approved proposals and you are ready to act.

-----

## Phase 5 — Deletion / merge review (2-step, only if user requests)

**Step 5a — Show impact**

For each deletion/merge candidate:

- Show full file content (or summary if >100 lines)
- List all references to it (CLAUDE.md, other agents, skills, commands, docs)
- State what degrades or breaks if removed

Ask: *“Do you want to delete/merge [path]? I’ll show you the exact action and wait for confirmation.”*

**Step 5b — Explicit per-item confirmation**

Only after the user confirms each item:

- State the exact file path
- Say: “Deleting/merging [path]. Type yes to confirm.”
- Execute only after “yes”

Never batch. Each item is a separate confirmation step.

-----

## Phase 6 — Execute approved changes

### Agent file location

```
.claude/agents/[agent-name].md      ← project-level, committed to git
~/.claude/agents/[agent-name].md    ← personal, all projects
```

### Agent file format

```markdown
---
name: agent-name
description: >
  Front-load the primary use case. Include specific trigger phrases.
  Use "PROACTIVELY" if Claude should auto-delegate without being asked.
  Keep under 1,536 characters.
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
model: sonnet
permissionMode: default
maxTurns: 20
skills:
  - skill-name-1
  - skill-name-2
memory: project
isolation: worktree
background: true
---

[System prompt — what this agent does, how it reasons, what it returns]
```

### System prompt content rules

**Agents do NOT inherit CLAUDE.md.** They receive only: their own system prompt +
the parent’s task prompt + basic environment (working directory, platform) +
preloaded skills. This means:

- Do not write “follow our coding standards” — state it explicitly or preload the relevant skill
- Do not reference team conventions Claude.md contains — the agent cannot see them
- Do preload skills for domain conventions the agent needs to follow via `skills:`

**Role coherence** — every frontmatter field should reinforce the same job:

- Read-only role → `tools: Read, Grep, Glob` or `disallowedTools: Write, Edit, Bash` + `permissionMode: plan`
- High-stakes role → `model: opus` + `permissionMode: default`
- Repeatable exploration → `memory: project`

**Output contract** — always tell the agent what to return:

- Bad: “Analyze the code.”
- Good: “Return a bullet list of findings with file path, line number, issue, and suggested fix. Stop after 20 findings.”

**Prompt length** — keep under 200 lines. If you need reference material (checklists,
patterns, templates), put it in a skill and preload it via `skills:`.

### After each write

Confirm: *“Created/updated [path]. Move to next item?”*

-----

## Tier decision reference

|Tier           |Criteria                                                                                         |
|---------------|-------------------------------------------------------------------------------------------------|
|MUST           |Systematic context bloat or wrong tool access without it. Well-defined, isolated, recurring role.|
|SHOULD         |Isolates noisy work, enforces tool constraints, provides specialized knowledge. Less frequent.   |
|COULD          |Marginal gain over built-ins. Low frequency.                                                     |
|NOT RECOMMENDED|Covered by built-in, a forked skill, CLAUDE.md rules, or inline prompting.                       |

## Agent vs forked skill

|Use an agent                                |Use a skill with `context: fork`|
|--------------------------------------------|--------------------------------|
|Needs persistent memory across sessions     |Isolation only, no memory needed|
|Needs specific tool restriction             |No tool restriction needed      |
|Needs a different model than the session    |Same model is fine              |
|Recurring role with a focused system prompt |One-off or infrequent task      |
|Domain specialist preloading multiple skills|Simple isolated task            |