-----

## name: skill-audit
description: >
Audits a codebase to propose which Claude Code skills to create, update, or
delete. Use when setting up AI config for a new project, when the .claude/skills/
directory feels stale or bloated, or when Claude keeps making the same mistakes
and existing skills aren’t helping. Also reviews .claude/commands/ and flags
commands that should be promoted to skills. Triggers on: “audit my skills”,
“what skills should I create”, “review my Claude config”, “skill setup”.
when_to_use: >
Run this before creating any new skill from scratch. It scans the codebase,
interviews the developer, and produces a tiered Must/Should/Could proposal —
preventing skill sprawl and ensuring each skill earns its place.

# Skill Audit

You are conducting a **Skill Audit** for this codebase. Goal: identify the *right*
skills — not as many as possible. Work through phases in order. Do not create,
modify, or delete any files until Phase 5.

## PHASE 1 — Orient (silent, no output yet)

Read all of the following before producing any output:

**Project structure**

- Top-level layout, README, docs/, onboarding docs
- Solution/project files (`.sln`, `package.json`, `pyproject.toml`, `Cargo.toml`, etc.)

**Existing AI config — read everything**

- `CLAUDE.md` (root + nested), `.claude/rules/*.md`
- `.claude/skills/*/SKILL.md`, `.claude/commands/*.md`, `.claude/agents/*.md`
- `.github/copilot-instructions.md`, `.github/prompts/`, `.cursorrules`

**Tech stack signals**

- Frameworks (imports, config files, lock files), test setup, CI/CD config

**Workflow signals**

- `git log --oneline -100` — what commit types dominate?
- Recurring file types changed together; any Makefile, taskfile, scripts/

Compile a mental model: *what kind of project is this, what does a dev day look like, and what is the existing AI config trying to solve?*

-----

## PHASE 2 — Discovery scan (output a findings report)

**2a. Project snapshot** — stack, architecture style, test strategy, notable complexity.

**2b. Recurring task patterns** — for each candidate:

- What the task is (specific: “add FastEndpoints endpoint with FluentValidation”, not “add endpoint”)
- Evidence (files, naming patterns, code structures)
- Why a skill helps — what must a dev remember each time? Exclude anything Claude handles well from training alone.

Filter strictly: *only tasks where a skill reduces real cognitive load or error rate on this specific project.*

**2c. Convention hotspots** — non-obvious local conventions AI gets wrong: naming rules,
file placement, DI patterns, error handling, anything that deviates from framework defaults.

**2d. Existing config inventory**

*Skills* — for each: name, coverage, assessment (relevant / stale / overlaps with CLAUDE.md).

*Commands* — for each: name, what it does, decision:

- Keep as **command**: short explicit trigger, no supporting files, always user-invoked (e.g. `/commit`)
- Promote to **skill**: needs supporting files, benefits from auto-invocation, or has logic Claude must reason through
- Note: same-name skill takes precedence over command; old command becomes redundant after promotion.

*Rules* — flag any that have grown into step-by-step procedures (better as skills).

-----

## PHASE 3 — Interview (ask, wait for answers before proceeding)

```
Before I propose anything, I want to validate my findings with you:

1. What are the 3–5 tasks where you most wish Claude understood your project better?
2. What mistakes does Claude make most often in this codebase?
3. Are there multi-step workflows where order matters — wrong order causes real problems?
4. Do you paste the same long prompt or reference file repeatedly? (Strong skill candidate.)
5. Any commands you barely use, or that feel too heavy for a single .md file?
6. Does my discovery report look right, or is something missing?
```

Developer-reported pain points take priority over scan-inferred ones.

-----

## PHASE 4 — Proposal (output only, no file changes)

Tier criteria:

- **MUST** — Without this, Claude makes systematic errors. Multi-step, non-obvious local conventions, multiple times/week.
- **SHOULD** — Meaningful improvement, recurring but less frequent. Worth maintenance cost.
- **COULD** — Low frequency or low complexity. Build only if bandwidth allows.
- **NOT RECOMMENDED** — Better as `CLAUDE.md` fact, `rules/*.md` entry, a command, or Claude’s training.

For each proposed skill:

```
### [skill-name]
Tier: MUST / SHOULD / COULD
What it does: [one sentence]
Invocation: AUTO | MANUAL | BOTH
Evidence: [files or patterns]
Developer-confirmed: YES / INFERRED
Supporting files: references/ | scripts/ | examples/ | none
Overlaps with: [existing skill/command/rule — resolution]
Action: CREATE NEW | UPDATE [name] | PROMOTE from command [name]
```

For each command needing a decision:

```
### /[command-name]
Decision: KEEP AS COMMAND | PROMOTE TO SKILL | DEPRECATE
Reason: [why]
If promoting: skill name, additions needed
```

List deletion candidates at the end — see `references/execution-guide.md` for the
deletion review flow (Phase 5) and file writing specs (Phase 6).

End with:

> “Phase 4 complete. Reply with which items to proceed with, skip, or defer,
> and whether to open deletion review. No files touched until you confirm.”

-----

## Guardrails

- **Lean over verbose.** 50 lines solving one thing beats 400 lines covering everything.
- **Templates/boilerplate → `references/`.** Inline content inflates context cost every load.
- **Rules ≠ Skills.** “Always use X” = `rules/*.md`. Skills = ordered procedural steps.
- **Short commands are fine.** Don’t promote just because skills seem more sophisticated.
- **Don’t re-document the framework.** Skill content must be wrong or invalid without this project’s context.
- **Merge overlapping skills.** 70%+ shared content → merge.
- **description is the routing signal.** Front-load the primary use case. Cap at 1,536 chars combined with `when_to_use`.
- **Never auto-delete.** Propose → confirm per item → execute.

-----

For Phase 5 (deletion review) and Phase 6 (file writing specs, SKILL.md format,
frontmatter reference, content rules), read `references/execution-guide.md`.