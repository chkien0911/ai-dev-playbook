-----

## name: agent-audit
description: >
Audits a codebase to propose which Claude Code custom subagents to create,
update, merge, or delete. Use when setting up .claude/agents/ for a new project,
when existing agents are too broad/overlapping, or when the main conversation
keeps getting flooded with search results and logs that belong in isolated
subagent contexts. Also flags agents that duplicate built-in agents (Explore,
Plan, General-purpose). Triggers on: “audit my agents”, “what agents should I
create”, “review my subagent config”, “agent setup”.
when_to_use: >
Run AFTER skill-audit so you know which skills exist before wiring them into
agent `skills:` fields. Best run when the .claude/agents/ directory is empty
(greenfield setup) or when agents feel misaligned with how the codebase is
actually used.

# Agent Audit

You are conducting an **Agent Audit** for this codebase. Goal: identify subagents
that genuinely improve AI-assisted development — with tight tool scopes, right-sized
models, and no duplication with built-ins or skills. Work through phases in order.
Do not create, modify, or delete any files until Phase 5.

## PHASE 1 — Orient (silent, no output yet)

Read all of the following before producing any output:

**Existing AI config**

- `.claude/agents/*.md` — current custom agents
- `.claude/skills/*/SKILL.md` — existing skills (agents can preload these)
- `.claude/commands/*.md`, `CLAUDE.md`, `.claude/rules/*.md`

**Project structure signals**

- Top-level layout, solution/project files, test setup, CI/CD config
- `git log --oneline -100` — what task types dominate?
- Any Makefile, taskfile, or `scripts/` folder

**Built-in agents — do NOT recreate these:**

- **Explore** — read-only codebase search (Haiku, fast, no write tools)
- **Plan** — read-only research during plan mode
- **General-purpose** — all tools, complex multi-step tasks
- **Claude Code Guide** — answers Claude Code feature questions

A custom agent must offer something a built-in doesn’t: a specialized system prompt,
specific tool restrictions, preloaded skills, persistent memory, or a different model.
If a built-in covers it, a custom agent is waste.

Compile a mental model: *what specialized roles need their own isolated context,
constrained tools, and focused system prompt — that built-ins don’t already cover?*

-----

## PHASE 2 — Discovery scan (output a findings report)

**2a. Project snapshot** — stack, architecture, complexity areas that benefit from
specialization (security-sensitive code, complex domain, multi-layer changes, etc.).

**2b. Task-to-agent candidates** — for each potential role:

- What the role does (one specific job — not “helps with everything”)
- Why it needs its own context (what would pollute the main conversation: search
  results, logs, large file reads, multi-step exploration?)
- Tool scope needed (be minimal — read-only roles must be read-only)
- Model fit (Haiku for simple lookups, Sonnet for most tasks, Opus for high-stakes reasoning)
- Built-in coverage (can Explore or General-purpose handle this adequately?)

Filter strictly: *only flag a role as an agent candidate if isolated context,
constrained tools, or a specialized system prompt meaningfully changes the outcome.*

**2c. Existing agent inventory** — for each agent in `.claude/agents/`:

- Tool scope: too broad / appropriate / unnecessarily restrictive
- Model choice: over-engineered / appropriate / under-resourced
- Preloaded skills: correct / missing relevant ones / loading irrelevant ones
- System prompt: focused / vague / restates CLAUDE.md rules (anti-pattern)
- Verdict: keep / update / merge / delete

**2d. Overlap check** — flag these anti-patterns:

- Agent that duplicates what a skill with `context: fork` already does
- Agent with `tools: Read, Grep, Glob` only → probably redundant with built-in Explore
- Agent with all tools and vague prompt → that’s General-purpose, not a custom agent
- Two agents with similar descriptions Claude would struggle to choose between
- Agent system prompt restating rules already in CLAUDE.md (agents don’t inherit it)

-----

## PHASE 3 — Interview (ask, wait for answers before proceeding)

```
Before I propose agents, I want to validate my findings with you:

1. Are there tasks where the main conversation fills up fast with search results,
   logs, or file contents you won't reference again? (Isolation candidates.)

2. Are there recurring specialized roles — security reviewer, test writer, migration
   validator — where consistent constraints and a focused prompt would help?

3. Are there expensive exploration tasks you run often where persistent memory
   across sessions would save re-exploration time?

4. Do you have tasks that should never modify files, or never run Bash?
   (Those need tool-restricted agents, not just instructions.)

5. Does my discovery report look right, or is something missing?
```

Developer-confirmed pain points take priority over scan-inferred ones.

-----

## PHASE 4 — Proposal (output only, no file changes)

Tier criteria:

- **MUST** — Without this, systematic context bloat, wrong tool access, or repeated re-explanation of specialized domain rules. Well-defined, isolated, recurring role.
- **SHOULD** — Isolates noisy work, enforces appropriate tool constraints, or provides specialized knowledge. Recurring but less frequent.
- **COULD** — Marginal gain over built-ins. Low frequency. Build if bandwidth allows.
- **NOT RECOMMENDED** — Better as a skill with `context: fork`, an existing built-in, CLAUDE.md rules, or inline prompting.

For each proposed or reviewed agent:

```
### [agent-name]
Action: MUST-CREATE | SHOULD-CREATE | COULD-CREATE | UPDATE | KEEP | MERGE INTO [name] | DELETE
Role: [one sentence]
Why its own agent: [what would pollute main context / why tools must be restricted]
tools: [comma-separated — minimal; omit to inherit all]
disallowedTools: [if safer to deny than allowlist]
model: haiku | sonnet | opus | inherit  — reason
permissionMode: default | plan | acceptEdits | bypassPermissions  — reason
maxTurns: [N — only if runaway risk; omit otherwise]
skills: [names to preload — only ones this agent actually uses]
memory: user | project | none  — reason
isolation: worktree  — only if agent needs clean isolated git worktree
background: true  — only if agent should always run async
Overlaps with: [built-in / existing agent / skill — resolution]
System prompt direction: [2–3 sentences on focus and expected output format]
```

List deletion/merge candidates separately — see `references/execution-guide.md`
for the deletion review flow (Phase 5) and file writing specs (Phase 6).

End with:

> “Phase 4 complete. Reply with which items to proceed with, skip, or defer,
> and whether to open deletion/merge review. No files touched until you confirm.”

-----

## Guardrails

- **One job per agent.** Focused role = predictable output. “Helps with everything” = worse version of the main conversation.
- **Tool scope = role scope, enforced by config.** “Don’t edit files” in a prompt can be ignored. `disallowedTools: Write, Edit` cannot.
- **Don’t recreate built-ins.** Explore, Plan, General-purpose cover most delegation needs. Justify every custom agent against them.
- **Skills ≠ Agents.** A skill with `context: fork` gives isolation without a full agent definition. If you only need isolation, a forked skill is simpler and cheaper.
- **Agents don’t inherit CLAUDE.md.** They see only their own system prompt + task prompt + environment + preloaded skills. State conventions explicitly or preload the relevant skill.
- **Memory compounds.** Agents with `memory: project` accumulate codebase knowledge across sessions. Add to agents that run repeatedly on the same codebase.
- **description = routing signal.** Write it like a routing rule, not a job posting. Use “PROACTIVELY” if Claude should auto-delegate without being asked.
- **Never auto-delete.** Propose → confirm per item → execute.

-----

For Phase 5 (deletion/merge review), Phase 6 (file writing specs, frontmatter
reference, system prompt rules, model selection guide), read:

- `references/execution-guide.md` — workflow for phases 5–6
- `references/frontmatter-reference.md` — all supported frontmatter fields with usage notes