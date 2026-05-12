# Skill Audit Prompt — General Purpose

# Chạy trong Claude Code. Paste toàn bộ nội dung này vào đầu session.

-----

You are conducting a **Skill Audit** for this codebase. Your goal is to discover what skills would genuinely help developers working here — not to generate as many skills as possible, but to identify the right ones.

Work through the following phases in order. Do not skip ahead. Do not create, modify, or delete any files until Phase 5.

-----

## PHASE 1 — Orient yourself (silent, no output yet)

Read the following to build a mental model of the project:

**1. Project structure**

- Top-level directory layout
- README, docs/, wiki/, or any onboarding documents
- Solution/project files (`.sln`, `package.json`, `pyproject.toml`, `Cargo.toml`, etc.)

**2. Existing AI configuration — read everything here**

- `CLAUDE.md` (root and any nested)
- `.claude/rules/*.md`
- `.claude/skills/*/SKILL.md`
- `.claude/commands/*.md`
- `.claude/agents/*.md`
- `.github/copilot-instructions.md`, `.github/prompts/`, `.github/instructions/`
- Any `.cursorrules`, `.aider.conf`, or similar

**3. Tech stack signals**

- Major frameworks (infer from imports, config files, lock files)
- Test setup (project naming, frameworks, folder structure)
- CI/CD config (`workflows/`, `azure-pipelines.yml`, `Dockerfile`, etc.)
- Infrastructure-as-code, migration files, seed data

**4. Developer workflow signals**

- `git log --oneline -100` — what types of commits dominate?
- Recurring file types changed together (signals repeated task types)
- Any Makefile, taskfile, justfile, or scripts/ folder with dev tooling

Compile a mental model: *what kind of project is this, what does a typical developer day look like, and what does the existing AI config try to solve?*

-----

## PHASE 2 — Discovery scan (output a findings report)

Produce a **Discovery Report** with these sections:

### 2a. Project snapshot

- Stack summary (languages, frameworks, major libraries)
- Architecture style (monolith / microservices / event-driven / layered / etc.)
- Test strategy observed
- Notable complexity areas

### 2b. Recurring task patterns

For each task pattern observed, list:

- **What the task is** — be specific (“add a FastEndpoints endpoint with FluentValidation” not “add API endpoint”)
- **Evidence** — specific files, naming patterns, or code structures that show this happens repeatedly
- **Why a skill helps** — what does a dev need to remember or look up each time? If Claude handles this well from training alone, say so and exclude it

Apply this filter strictly: *only list tasks where a skill reduces real cognitive load or error rate on this specific project.* Generic framework knowledge is not a skill candidate.

### 2c. Convention hotspots

List non-obvious local conventions that AI gets wrong without guidance:

- Naming rules, file placement, project structure rules
- DI registration, error handling, transaction patterns
- Anything that deviates from framework defaults in a non-trivial way

### 2d. Existing config inventory

**Skills** (`.claude/skills/*/SKILL.md`):
For each: name, what it covers, assessment (still relevant / stale / overlaps with CLAUDE.md or rules).

**Commands** (`.claude/commands/*.md`):
For each: name, what it does, and whether it should stay a command or be promoted to a skill.
Apply this decision rule:

- Keep as **command** if: it’s a short explicit trigger with no branching logic, no supporting files needed, and the user always invokes it manually (e.g., `/commit`, `/deploy`)
- Promote to **skill** if: it has grown beyond a single-purpose prompt, needs supporting files (templates, scripts, reference docs), could benefit from auto-invocation by Claude, or has logic Claude needs to reason through

Note: a `.claude/commands/name.md` and `.claude/skills/name/SKILL.md` with the same name both create `/name` — the skill takes precedence. Promoting a command to a skill makes the command file redundant.

**Rules** (`.claude/rules/*.md` or CLAUDE.md sections):
Flag any rules that have grown into step-by-step procedures — those are better as skills.

-----

## PHASE 3 — Interview (ask the developer, wait for answers)

Before proposing anything, ask these questions. Present as a numbered list, conversational tone.

```
Before I propose changes, I want to validate my findings with you:

1. What are the 3–5 tasks you do most often where you wish Claude understood 
   your project better?

2. What are the most common mistakes Claude makes when working in this codebase?

3. Are there workflows that are multi-step and order-sensitive — where doing 
   steps out of order causes real problems?

4. Do you currently paste a long prompt or a reference file repeatedly into chat? 
   (Those are strong skill candidates.)

5. Are there any commands you barely use, or that feel too heavy for a single .md file?

6. Looking at my discovery report — does anything look wrong or missing?
```

Developer-reported pain points take priority over scan-inferred ones. Incorporate their answers before writing proposals.

-----

## PHASE 4 — Proposal (output only, no file changes yet)

Produce a **Skill Proposal** organised by tier.

Apply these criteria strictly:

**MUST HAVE** — Without this, Claude makes systematic errors or needs constant re-prompting. Task is complex, multi-step, or has non-obvious local conventions. Happens multiple times per week.

**SHOULD HAVE** — Meaningful improvement to AI quality. Task is recurring but less frequent, or conventions are moderately non-obvious. Worth the maintenance cost.

**COULD HAVE** — Nice to have but low frequency or low complexity. Build only if bandwidth allows.

**NOT RECOMMENDED** — Better served by: `CLAUDE.md` (project-wide facts and constraints), `rules/*.md` (topic-scoped always-on instructions), a command (simple explicit trigger), or Claude’s own training.

-----

For each proposed **skill**, use this format:

```
### [skill-name]
Tier: MUST / SHOULD / COULD
What it does: [one sentence]
Invocation: AUTO | MANUAL | BOTH
Evidence: [specific files or patterns from scan]
Developer-confirmed: YES / INFERRED
Supporting files needed: references/ | scripts/ | examples/ | none
Overlaps with: [existing skill, command, or rule — and how to resolve]
Action: CREATE NEW | UPDATE [existing name] | PROMOTE from command [command-name]
```

For each **command** that needs a decision:

```
### /[command-name]
Current: [what it does now as a command]
Decision: KEEP AS COMMAND | PROMOTE TO SKILL | DEPRECATE
Reason: [why]
If promoting: skill name, what additions are needed (supporting files, frontmatter fields)
```

-----

Then list deletion candidates separately:

```
## Candidates for deletion

### [name]  (SKILL | COMMAND)
Reason: [redundant with X / stale / superseded / command promoted to skill]
References: [anywhere it's mentioned — CLAUDE.md, agents, other skills, docs]
⚠️ Requires explicit per-item approval. Handled in Phase 5.
```

-----

End the proposal with:

> “Phase 4 complete. Please review above. Reply with:
> 
> - Which CREATE / UPDATE / PROMOTE items to proceed with
> - Which to skip or defer
> - Whether to open deletion review (Phase 5 — one approval per item)
> 
> No files will be touched until you confirm.”

-----

## PHASE 5 — Deletion review (2-step, only if user requests)

**Step 5a — Show impact before asking**

For each deletion candidate:

- Show full file content (or a summary if >100 lines)
- List all references to it across the codebase
- State what degrades or breaks if deleted

Ask: *“Do you want to delete [path]? I’ll show you the exact action and wait for your confirmation.”*

**Step 5b — Explicit per-item confirmation**

Only after the user confirms each item:

- State the exact file path
- Say: “Deleting [path]. Type yes to confirm.”
- Execute only after “yes”

Never batch-delete. Each item is a separate confirmation step.

-----

## PHASE 6 — Execute approved changes

### Skill file layout

```
.claude/skills/[skill-name]/
├── SKILL.md              ← required entrypoint
├── references/           ← long docs, loaded on demand
│   └── [topic].md
├── scripts/              ← executable scripts
│   └── [script].sh
└── examples/             ← sample outputs showing expected format
    └── [sample].md
```

Only create subdirectories that are actually needed. An empty `references/` folder adds noise.

### SKILL.md format

```markdown
---
name: skill-name
description: >
  Primary use case front-loaded in the first sentence — Claude reads this to
  decide whether to load the skill. Include specific trigger phrases and task
  contexts. Keep description + when_to_use combined under 1,536 characters
  (truncated in the skill listing).
when_to_use: >
  Additional trigger contexts. Use only if description is already dense.
disable-model-invocation: true
  # Add ONLY for manual-only skills: destructive ops, deployments, anything
  # the user should always invoke explicitly rather than letting Claude decide.
context: fork
  # Add ONLY if the skill should run isolated in a subagent context.
allowed-tools: Read Grep Bash
  # Add ONLY if tool restriction is intentional. Omit to inherit all tools.
---

[Skill body — imperative instructions]

[Reference supporting files explicitly:]
[See references/conventions.md for the full naming convention table.]
[Run scripts/validate.sh to check output before committing.]
```

### Content rules

- **Imperative form**: “Run the tests before committing” not “You should run the tests”
- **Explain why**: “Validate with FluentValidation, not data annotations — the team chose FluentValidation for testability” gives Claude context to make better decisions than “use FluentValidation”
- **Keep SKILL.md lean**: target under 200 lines for the instructions themselves; push templates, boilerplate, and long reference tables to `references/`
- **Reference files explicitly**: name them in SKILL.md with guidance on when to read them — Claude won’t load them unless directed
- **If SKILL.md exceeds ~300 lines before adding reference material**: split into multiple skills or restructure

### Promoting a command to a skill

1. Create `.claude/skills/[name]/SKILL.md` with proper frontmatter
1. Add supporting files if needed
1. Confirm the skill works as expected with the user
1. Flag `.claude/commands/[name].md` for deletion via Phase 5
1. Note: skill and command with the same name can coexist temporarily — skill takes precedence automatically

### After each write

Confirm: *“Created/updated [path]. Move to next item?”*

-----

## Guardrails — apply throughout

**Lean over verbose.** A 50-line skill that solves one thing well beats a 400-line skill that covers everything loosely. If you feel the urge to add another section, ask whether it belongs in `references/` instead.

**Templates and boilerplate go in `references/`.** SKILL.md should read like a playbook, not a manual. Inline boilerplate inflates context cost every time the skill loads.

**Rules ≠ Skills.** “Always use X pattern when writing Y” is a `rules/*.md` entry or CLAUDE.md fact — not a skill. Skills are for procedural knowledge with ordered steps.

**Short commands are fine.** Don’t promote a command to a skill just because skills seem more sophisticated. A 10-line explicit trigger that never needs supporting files is correct as a command.

**Don’t re-document the framework.** Skill content should be invalid or wrong without this specific project’s context. If it would work equally in any project using the same framework, it belongs in Claude’s training, not here.

**Merge narrow overlapping skills.** If two proposed skills share 70%+ content or are always used together, merge them into one with a broader description.

**description is the trigger.** Front-load the primary use case. The combined `description` + `when_to_use` text is truncated at 1,536 characters in Claude’s skill listing — verbose preambles bury the signal.

**Never auto-delete.** Always: propose → human confirms per item → execute.