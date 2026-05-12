# Skill Audit — Execution Guide

Reference for Phase 5 (deletion review) and Phase 6 (file writing).
Read this when the user has approved proposals and you are ready to act.

-----

## Phase 5 — Deletion review (2-step, only if user requests)

**Step 5a — Show impact before asking**

For each deletion candidate:

- Show full file content (or summary if >100 lines)
- List all references across the codebase (CLAUDE.md, agents, other skills, docs)
- State what degrades or breaks if deleted

Ask: *“Do you want to delete [path]? I’ll show you the exact action and wait for confirmation.”*

**Step 5b — Explicit per-item confirmation**

Only after the user confirms each item individually:

- State the exact file path
- Say: “Deleting [path]. Type yes to confirm.”
- Execute only after “yes”

Never batch-delete. Each item is a separate confirmation step.

-----

## Phase 6 — Execute approved changes

### Skill directory layout

```
.claude/skills/[skill-name]/
├── SKILL.md              ← required entrypoint
├── references/           ← long docs, loaded on demand
│   └── [topic].md
├── scripts/              ← executable scripts Claude can run
│   └── [script].sh
└── examples/             ← sample outputs showing expected format
    └── [sample].md
```

Only create subdirectories that are actually needed. An empty folder adds noise.

### SKILL.md frontmatter format

```yaml
---
name: skill-name
description: >
  Primary use case front-loaded in the first sentence. Include specific trigger
  phrases and task contexts. Keep description + when_to_use combined under
  1,536 characters (truncated in the skill listing).
when_to_use: >
  Additional trigger contexts. Use only if description is already dense.
disable-model-invocation: true
  # Add ONLY for manual-only skills: destructive ops, deployments, or anything
  # the user should always invoke explicitly.
context: fork
  # Add ONLY if the skill should run isolated in a subagent context.
allowed-tools: Read Grep Bash
  # Add ONLY if tool restriction is intentional. Omit to inherit all tools.
---
```

### Content rules

- **Imperative form**: “Run the tests” not “You should run the tests”
- **Explain why**: “Use FluentValidation, not data annotations — chosen for testability” gives Claude better reasoning context than just “use FluentValidation”
- **Keep SKILL.md lean**: target under 200 lines for instructions; push templates, boilerplate, long reference tables to `references/`
- **Reference files explicitly**: name them in SKILL.md with guidance on when to read — Claude won’t load them unless directed
- **If SKILL.md exceeds ~300 lines before adding reference material**: split into multiple skills or restructure

### Promoting a command to a skill

1. Create `.claude/skills/[name]/SKILL.md` with proper frontmatter
1. Add supporting files if needed
1. Confirm with user the skill works as expected
1. Flag `.claude/commands/[name].md` for deletion via Phase 5
1. Note: skill and command with the same name can temporarily coexist — skill takes precedence automatically

### After each file write

Confirm: *“Created/updated [path]. Move to next item?”*

-----

## Tier decision reference

|Tier           |Criteria                                                                               |
|---------------|---------------------------------------------------------------------------------------|
|MUST           |Systematic errors without it. Multi-step, non-obvious conventions, multiple times/week.|
|SHOULD         |Meaningful improvement. Recurring but less frequent. Worth maintenance cost.           |
|COULD          |Low frequency or marginal gain. Build if bandwidth allows.                             |
|NOT RECOMMENDED|Better as CLAUDE.md fact, rules/*.md entry, a command, or Claude’s training.           |

## Command vs skill decision

|Keep as command                       |Promote to skill                                  |
|--------------------------------------|--------------------------------------------------|
|Short, single-purpose explicit trigger|Needs supporting files (templates, scripts, refs) |
|No supporting files needed            |Benefits from auto-invocation by Claude           |
|User always invokes manually          |Has branching logic Claude needs to reason through|
|Example: `/commit`, `/deploy`         |Example: code review workflow, migration generator|