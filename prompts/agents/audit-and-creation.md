# Agent Audit Prompt — General Purpose

-----

You are conducting an **Agent Audit** for this codebase. Your goal is to identify which subagents would genuinely improve AI-assisted development here — with tight tool scopes, right-sized models, and no duplication with existing skills, commands, or built-in agents.

Work through the following phases in order. Do not create, modify, or delete any files until Phase 5.

-----

## PHASE 1 — Orient yourself (silent, no output yet)

Read the following before doing anything else:

**1. Existing AI configuration**

- `.claude/agents/*.md` — existing custom agents
- `.claude/skills/*/SKILL.md` — existing skills (agents can preload these)
- `.claude/commands/*.md` — existing commands
- `CLAUDE.md` and `.claude/rules/*.md` — standing rules and constraints

**2. Project structure signals**

- Top-level layout, solution/project files (`.sln`, `package.json`, `Cargo.toml`, etc.)
- Test setup, CI/CD config, Dockerfile, migration files
- `git log --oneline -100` — what task types dominate?
- Any Makefile, taskfile, or `scripts/` folder

**3. Built-in agents to be aware of — do NOT recreate these**
Claude Code ships these built-ins. Custom agents that duplicate them are waste:

- **Explore** — read-only codebase search and exploration (Haiku, fast)
- **Plan** — read-only research during plan mode
- **General-purpose** — all-tools complex multi-step tasks
- **Claude Code Guide** — answers questions about Claude Code itself

Only create a custom agent when built-ins are genuinely insufficient for the project’s needs.

Compile a mental model: *what specialized roles would benefit from their own isolated context, constrained tools, and focused system prompt — that the built-ins don’t already cover?*

-----

## PHASE 2 — Discovery scan (output a findings report)

### 2a. Project snapshot

- Stack and architecture summary
- Complexity areas that benefit from specialization (security-sensitive code, complex domain, multi-layer changes, etc.)
- Recurring task types from git history

### 2b. Task-to-agent candidates

For each potential agent role, assess:

- **What the role does** — one specific job (not “helps with everything”)
- **Why it needs its own context** — what would pollute the main conversation if done inline? (search results, logs, large file reads, multi-step exploration)
- **Tool scope** — which tools does this role actually need? Be minimal. Read-only roles should be read-only.
- **Model fit** — does this need Opus reasoning, or is Haiku/Sonnet sufficient?
- **Built-in coverage** — can the built-in Explore or General-purpose agent handle this adequately?

Apply this filter strictly: *only flag a role as an agent candidate if isolated context, constrained tools, or a specialized system prompt meaningfully changes the outcome.*

### 2c. Existing agent inventory

For each agent in `.claude/agents/`:

- Name and current purpose
- Tool scope assessment: too broad / appropriate / unnecessarily restrictive
- Model choice assessment: over-engineered / appropriate / under-resourced
- Skills preloaded: correct / missing relevant skills / loading irrelevant skills
- System prompt quality: focused / vague / overlaps with CLAUDE.md
- Overall verdict: keep / update / merge / delete

### 2d. Overlap check

Flag any of these anti-patterns found:

- Agent that duplicates what a skill with `context: fork` already does
- Agent with `tools: Read, Grep, Glob` that overlaps with built-in Explore
- Agent with all tools and a vague prompt — that’s General-purpose, not a custom agent
- Two agents with similar descriptions that Claude would struggle to choose between
- Agent whose system prompt re-states rules already in CLAUDE.md

-----

## PHASE 3 — Interview (ask the developer, wait for answers)

```
Before I propose agents, I want to validate my findings with you:

1. Are there tasks where you've noticed the main conversation filling up fast 
   with search results, logs, or file contents that aren't needed afterward?
   (Those are isolation candidates — strong agent signals.)

2. Are there recurring specialized roles you invoke repeatedly — like a security 
   reviewer, a test writer, a migration validator — where consistent constraints 
   and a focused prompt would help?

3. Are there expensive exploration tasks you run often where persistent memory 
   across sessions would save re-exploration time?

4. Do you have tasks that should never modify files, or never run Bash? 
   (Those need tool-restricted agents, not just instructions.)

5. Looking at my discovery report — does anything look wrong or missing?
```

Developer-confirmed pain points take priority over scan-inferred ones.

-----

## PHASE 4 — Proposal (output only, no file changes yet)

### Classification tiers

**MUST HAVE** — Without this agent, developers face systematic context bloat, incorrect tool access, or repeated re-explanation of specialized domain rules. The role is well-defined, isolated, and recurring.

**SHOULD HAVE** — Meaningful improvement: isolates noisy work from the main context, enforces appropriate tool constraints, or provides specialized knowledge that Claude lacks in general use. Recurring but less frequent.

**COULD HAVE** — Low frequency or marginal gain over existing built-ins. Worth creating only if bandwidth allows.

**NOT RECOMMENDED** — Better handled by: a skill with `context: fork`, an existing built-in agent, CLAUDE.md rules, or inline prompting.

-----

For each proposed or reviewed agent, use this format:

```
### [agent-name]
Tier / Action: MUST-CREATE | SHOULD-CREATE | COULD-CREATE | UPDATE | KEEP | MERGE INTO [name] | DELETE
Role in one sentence: [what it does]
Why its own agent: [what would pollute main context / why tools must be restricted]
tools: [comma-separated — be minimal; omit field if inheriting all is intentional]
disallowedTools: [if safer to deny than to allowlist]
model: haiku | sonnet | opus | inherit  — with reason
permissionMode: default | plan | acceptEdits | bypassPermissions  — with reason
maxTurns: [N — only if runaway risk exists; omit otherwise]
skills: [skill names to preload at startup — only those the agent actually uses]
memory: user | project | none  — with reason
isolation: worktree  — only if agent needs a clean isolated git worktree
background: true  — only if agent should always run async
Overlaps with: [built-in / existing agent / skill — and how resolved]
System prompt direction: [2-3 sentences on what the prompt should focus on]
```

-----

Then separately:

```
## Candidates for deletion or merge

### [agent-name]  (DELETE | MERGE INTO [target])
Reason: [duplicates built-in / too broad / overlaps with skill / never used / vague]
References: [anywhere it's mentioned — CLAUDE.md, other agents, commands, docs]
⚠️ Requires explicit per-item approval. Handled in Phase 5.
```

-----

End the proposal with:

> “Phase 4 complete. Please review above.
> Reply with:
> 
> - Which CREATE / UPDATE items to proceed with
> - Which to skip or defer
> - Whether to open deletion/merge review (Phase 5 — one approval per item)
> 
> No files will be touched until you confirm.”

-----

## PHASE 5 — Deletion / merge review (2-step, only if user requests)

**Step 5a — Show impact**

For each deletion/merge candidate:

- Show full file content (or summary if >100 lines)
- List all references to it (CLAUDE.md, other agents, skills, commands)
- State what degrades or breaks if removed

Ask: *“Do you want to delete/merge [path]? I’ll show you the exact action and wait for confirmation.”*

**Step 5b — Explicit per-item confirmation**

Only after the user confirms each item:

- State the exact path
- Say: “Deleting/merging [path]. Type yes to confirm.”
- Execute only after “yes”

Never batch. Each item is a separate confirmation step.

-----

## PHASE 6 — Execute approved changes

### Agent file format

Location: `.claude/agents/[agent-name].md`

```markdown
---
name: agent-name
description: >
  When Claude should delegate to this agent — front-load the primary use case.
  Include specific trigger phrases: "review for security", "run migration",
  "explore this unfamiliar module". Use PROACTIVELY if Claude should auto-delegate
  without being asked. Keep under 1,536 characters combined with when_to_use.
tools: Read, Grep, Glob, Bash
  # Comma-separated allowlist. Omit to inherit all tools.
  # Supports Agent(agent_type) syntax to restrict which subagents this agent can spawn.
disallowedTools: Write, Edit
  # Safer than allowlisting when the deny-set is small.
  # Use one of tools OR disallowedTools, not both.
model: haiku
  # haiku — fast, cheap, read-only or simple tasks
  # sonnet — balanced, most custom agents
  # opus — high-stakes reasoning (security review, architecture decisions)
  # inherit — follow the main session model (default)
permissionMode: default
  # default — standard permission gates (most agents)
  # plan — read-only until explicitly approved (destructive or risky ops)
  # acceptEdits — auto-approve file edits (trusted write agents)
  # bypassPermissions — no gates (CI/automation only, never interactive)
maxTurns: 20
  # Omit unless runaway risk is real. Built-in Explore has no cap.
skills:
  - skill-name-1   # preloaded at startup — full content injected into context
  - skill-name-2   # only list skills this agent actually uses
memory: project
  # project — persists to .claude/agent-memory/[name]/, committed to git, shared with team
  # user    — persists to ~/.claude/agent-memory/[name]/, personal only
  # none    — no persistence (default; fine for stateless agents)
isolation: worktree
  # Add only if the agent needs a clean isolated git worktree.
  # Cost: worktree setup time. Benefit: no risk of polluting working tree.
background: true
  # Add only if this agent should always run async (fire-and-forget style).
---

[System prompt — what this agent does, how it reasons, what it returns]

[Key rules:]
- Be explicit about what the agent should and should NOT do
- Tell it what to return (summary? file list? diff? structured report?)
- Tell it when to stop rather than keep exploring
- Do NOT restate rules already in CLAUDE.md — the agent does not inherit them
```

### Content rules for the system prompt

**Agents do NOT inherit CLAUDE.md.** They receive only: their own system prompt + the parent’s task prompt + basic environment (working directory, platform) + preloaded skills. This means:

- Do not write “follow our coding standards” — the agent doesn’t know them unless you preload the relevant skill
- Do not write “as usual, use xUnit” — state it explicitly or preload the relevant skill
- Do preload skills for domain conventions the agent needs to follow

**Role coherence.** Every frontmatter field should reinforce the same job:

- Read-only role → `tools: Read, Grep, Glob` + `disallowedTools: Write, Edit` + `permissionMode: plan`
- High-stakes role → `model: opus` + `effort: high` (Opus 4 only) + `permissionMode: default`
- Repeatable exploration → `memory: project` so it accumulates codebase knowledge across sessions

**Output contract.** Always tell the agent what to return. Vague agents produce vague results:

- Bad: “Analyze the code.”
- Good: “Return a bullet list of findings with file path, line number, issue, and suggested fix. Stop after 20 findings.”

**Prompt length.** Keep system prompts under 200 lines. If you need reference material (checklists, patterns, templates), put it in a skill and preload it via `skills:`.

### Model selection guide

|Use case                                                  |Model    |
|----------------------------------------------------------|---------|
|Codebase search, file reads, simple lookups               |`haiku`  |
|Code generation, test writing, multi-step tasks           |`sonnet` |
|Security review, architecture decisions, complex reasoning|`opus`   |
|Match the user’s current session model                    |`inherit`|

### After each write

Confirm: *“Created/updated [path]. Move to next item?”*

-----

## Guardrails — apply throughout all phases

**One job per agent.** An agent with a focused role produces predictable, usable output. An agent that “helps with everything” is just a worse version of the main conversation.

**Tool scope = role scope.** If the role is read-only, the tools must be read-only. Don’t rely on the system prompt saying “don’t edit files” — use `disallowedTools` or a restrictive `tools` list. Instructions can be ignored; tool restrictions cannot.

**Don’t recreate built-ins.** Explore (read-only, Haiku), Plan (read-only, research), and General-purpose (all tools) cover the majority of delegation needs. A custom agent must offer something a built-in doesn’t: a focused system prompt, a specific tool restriction, preloaded skills, persistent memory, or a model choice the built-in doesn’t make.

**Skills ≠ Agents.** A skill with `context: fork` runs in an isolated subagent context without needing a full agent definition. If you only need isolation (not tool restriction, not memory, not a specialized model), a forked skill is simpler and cheaper.

**Memory is a compounding investment.** Agents with `memory: project` accumulate codebase knowledge (file locations, architecture patterns, recurring issues) across sessions. Over time, they skip redundant exploration. Add memory to agents that run repeatedly on the same codebase.

**Descriptions are routing signals.** Claude reads the `description` field to decide when to delegate. Write it like a routing rule, not a job posting. Include specific trigger phrases. Use “PROACTIVELY” in the description if Claude should delegate without being asked.

**maxTurns is a safety valve, not a default.** Only set it when there’s a real runaway risk (e.g., an agent that loops on retries). Most agents don’t need it.

**No file changes before Phase 5 approval for deletions.** Propose → human confirms per item → execute. No exceptions.