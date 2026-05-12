# Config Audit Prompt — General Purpose

-----

You are conducting a **Config Audit** to create or improve the project’s `CLAUDE.md`
and `.claude/rules/` files. Goal: a lean always-on config that tells Claude exactly
what it needs — no more, no less.

Work through phases in order. Do not write any files until Phase 5.

-----

## PHASE 1 — Orient (silent, no output yet)

Read all of the following before producing any output:

**Existing Claude config**

- `CLAUDE.md` (root) and `.claude/CLAUDE.md` if either exists
- `CLAUDE.local.md` if present — note but do NOT modify (personal overrides)
- `.claude/rules/*.md` — note each file name, frontmatter, and content
- `.claude/skills/*/SKILL.md` — to avoid duplicating skill content in CLAUDE.md
- `.claude/agents/*.md` — to avoid duplicating agent context in CLAUDE.md
- `.claude/commands/*.md` — to understand what workflows are already documented

**Project structure**

- Top-level layout, solution/project files (`.sln`, `package.json`, `Cargo.toml`, etc.)
- README, docs/, onboarding docs — what would a new dev need to know?
- `git log --oneline -50` — what task types dominate?
- CI/CD config, Dockerfile, Makefile, taskfile — build and test commands

**Tech stack signals**

- Frameworks, test setup, linting and formatter config
- Any `.editorconfig`, `eslint.config.*`, `prettier.config.*`, `pyproject.toml`
- Monorepo signals: multiple `package.json` or project files at different depths

Compile a mental model: *what must every Claude session know about this project?
What does a dev day look like? What would a new team member need explained first?*

-----

## PHASE 2 — Discovery scan (output a findings report)

### 2a. Project snapshot

Stack, architecture style, test strategy, build commands, monorepo vs single-repo.

### 2b. What belongs in CLAUDE.md

For each candidate, apply the **cut test**: *“If I remove this line, will Claude make a mistake?”*
If yes → belongs. If no → cut.

**Categories that typically belong:**

- Build, test, lint commands Claude needs to run
- Project layout (where things live, especially in monorepos)
- Stack facts Claude can’t infer reliably (“uses Bun not Node”, “targets .NET 8 not Framework 4.8”)
- Non-obvious coding conventions that apply everywhere
- Critical constraints (“never use X”, “always Y before committing”)
- `@`-import pointers to detailed docs Claude should fetch on demand

**Categories that typically don’t belong:**

- Multi-step procedures → skill
- Path-specific conventions → rules with `paths:` frontmatter
- Rarely-used workflows → command
- Things Claude already knows from framework training → nowhere

### 2c. Existing CLAUDE.md assessment (if one exists)

- Line count and structure quality
- Lines that fail the cut test (candidates for removal)
- Lines that belong in `rules/*.md` instead (topic-specific, long, or path-scoped)
- Lines that belong in a skill (multi-step procedures)
- Anything contradicting other config files

### 2d. Existing rules inventory (if `.claude/rules/` exists)

For each rule file:

- Has YAML frontmatter with `paths:`? (determines load behaviour — see Phase 6)
- Still relevant to current codebase?
- Overlaps with CLAUDE.md content?
- Correctly placed, or should it move to CLAUDE.md / skill?

-----

## PHASE 3 — Interview (ask, wait for answers before proceeding)

```
Before I propose changes, I want to validate what I found:

1. What does Claude get wrong most often despite any existing config?

2. Are there instructions you find yourself repeating every session?
   (Those belong in CLAUDE.md.)

3. Are there conventions that only apply to specific files or folders?
   (Those belong in path-scoped rules, not CLAUDE.md.)

4. Are there CLAUDE.md sections you never maintain or look at?
   (Candidates for pruning or moving.)

5. Does my discovery report look accurate, or is something missing?
```

Developer-reported pain points take priority over scan-inferred ones.

-----

## PHASE 4 — Proposal (output only, no file changes yet)

### 4a. CLAUDE.md proposal

State one of:

- **CREATE** — no CLAUDE.md exists; propose full content
- **UPDATE** — CLAUDE.md exists; propose specific changes (add / remove / move)
- **NO CHANGE** — existing CLAUDE.md is already well-structured

Show the **complete proposed CLAUDE.md** in a code block so the developer can
review it in full before anything is written. For updates, also show:

- Lines removed and why
- Lines added and why
- Lines moved to rules/ and why

**Check before outputting:** proposed file must be under 200 lines. If it exceeds
that, identify what to cut or move — do not just condense wording.

### 4b. Rules proposal

For each proposed rule file:

```
### .claude/rules/[filename].md
Action: CREATE | UPDATE | DELETE | NO CHANGE
Load behaviour: ALWAYS (no paths frontmatter) | PATH-SCOPED (paths: frontmatter)
Paths scope: [glob patterns — only if path-scoped]
Content summary: [what this rule covers in one sentence]
Why a rule not CLAUDE.md: [too long / path-specific / topic-isolated]
Overlaps with: [any CLAUDE.md section, skill, or other rule — and resolution]
```

### 4c. Deletion candidates

```
## Candidates for deletion

### [filename]
Reason: [redundant / stale / superseded / content moved elsewhere]
⚠️ Requires explicit per-item approval. Handled in Phase 5.
```

End with:

> “Phase 4 complete. Please review the proposed CLAUDE.md and rules above.
> Reply with which changes to proceed with, and whether to open deletion review.
> No files will be written until you confirm.”

-----

## PHASE 5 — Deletion review (2-step, only if user requests)

**Step 5a — Show impact before asking**

For each deletion candidate:

- Show full file content (or summary if >100 lines)
- List all `@`-imports or references to this file from other config files
- State what changes if deleted (what instruction Claude permanently loses)

Ask: *“Delete [path]? I’ll show the exact action and wait for confirmation.”*

**Step 5b — Explicit per-item confirmation**

Only after the user confirms each item:

- Say: “Deleting [exact path]. Type yes to confirm.”
- Execute only after “yes”

Never batch-delete. One confirmation per file.

-----

## PHASE 6 — Execute approved changes

### CLAUDE.md — file format

**Location** (pick one, not both):

- `./CLAUDE.md` — project root, most common, checked into git
- `./.claude/CLAUDE.md` — inside .claude folder, also valid
- `./CLAUDE.local.md` — personal overrides only; add to `.gitignore`

**Structure:**

```markdown
# [Project Name]

[One sentence: what this project is and what stack it uses.]

## Commands

- Build: `[command]`
- Test: `[command]`  (prefer running single tests, not full suite)
- Lint: `[command]`

## Project Layout

[Only if non-obvious — skip for standard framework structures.]

- `src/api/` — [what lives here]
- `src/domain/` — [what lives here]

## Stack

[Facts Claude cannot reliably infer from code alone.]

- Runtime: Bun (not Node)
- Test framework: xUnit (not MSTest) — never mix
- Target: .NET 8 (not Framework 4.8)

## Conventions

[Non-obvious local conventions that apply across the whole codebase.]

- Use FluentValidation, not DataAnnotations
- All API responses use ProblemDetails (RFC 7807)

## Constraints

[Hard rules. Use IMPORTANT or YOU MUST for critical ones.]

- NEVER run the full test suite — run single tests for performance
- YOU MUST run `npm run typecheck` after any TypeScript changes
- Never use `--no-verify` on git commits

## References

[Pointers to detailed docs — don't embed their content inline.]

- Git workflow: @docs/git-workflow.md
- API style: see .claude/rules/api-style.md
```

**Size rules:**

- Target: **under 200 lines** — adherence is highest here
- Hard ceiling: ~300 lines — above this Claude loses signal in the noise
- If approaching 200 lines, move content to `rules/*.md`, don’t just condense wording
- `@`-imported files load into context at launch — they count against the budget too

**Writing rules:**

- Concrete over abstract: “Run `npm test`” not “Run tests”
- Constraints need alternatives: “Never use –foo; prefer –bar” — not just “Never use –foo” (Claude gets stuck without an alternative)
- Use `IMPORTANT` or `YOU MUST` sparingly — only for rules that must never be missed
- No explanations, no tutorials, no philosophy — context injection only

**`@`-import syntax** for referencing external files:

```markdown
For git workflow, see @docs/git-workflow.md
```

Imported files expand and load at launch — use for frequently-needed references.
Use plain path pointers (no `@`) for large files Claude should fetch selectively.

-----

### .claude/rules/ — file format

**Load behaviour — critical to understand:**

|Frontmatter?      |Has `paths:`?|Load behaviour                                              |
|------------------|-------------|------------------------------------------------------------|
|No frontmatter    |—            |Loads **every session**, same as CLAUDE.md                  |
|Yes, no `paths:`  |No           |Loads every session (description used for `/memory` display)|
|Yes, with `paths:`|Yes          |**Lazy-loaded** — only when Claude touches a matching file  |

Use path-scoped rules for file-type or folder-specific conventions. They don’t
consume context window until Claude actually works in that area. This is the
primary mechanism for keeping overall context lean on large projects.

**Format — no frontmatter (always-on rule):**

```markdown
# [Rule Topic]

[Instructions that apply universally, across all files and tasks.]
```

**Format — with frontmatter (path-scoped, lazy-loaded):**

```markdown
---
description: Brief summary shown in /memory output
paths:
  - "src/api/**"
  - "**/*.endpoint.ts"
  - "tests/**/*.test.*"
---

# [Rule Topic]

[Instructions that apply only when Claude works with matching files.]
```

**Naming:** one topic per file, name makes purpose obvious.

|Good               |Bad             |
|-------------------|----------------|
|`testing.md`       |`rules.md`      |
|`api-style.md`     |`conventions.md`|
|`error-handling.md`|`misc.md`       |

**Size:** target under 80 lines per file. If a rule file exceeds 80 lines,
split by topic or move detail into a skill’s `references/` folder.

**What belongs in rules/ vs CLAUDE.md:**

|CLAUDE.md                     |rules/                                           |
|------------------------------|-------------------------------------------------|
|Universal, cross-cutting facts|File-type or folder-specific conventions         |
|Build/test/run commands       |Testing patterns scoped to `tests/**`            |
|Stack facts, project layout   |API style scoped to `src/api/**`                 |
|Critical always-on constraints|Anything that would push CLAUDE.md over 200 lines|

**What belongs in neither:**

|Content                               |Correct location                      |
|--------------------------------------|--------------------------------------|
|Multi-step procedures                 |`.claude/skills/`                     |
|Explicit user-triggered workflows     |`.claude/commands/`                   |
|Agent-specific context                |Agent system prompt or preloaded skill|
|Framework knowledge Claude already has|Nowhere                               |

-----

### After each write

“Written [path]. Move to next item?”

After writing CLAUDE.md, suggest the developer run:

- `/memory` — verify which files loaded and what’s in context
- `/context` — check token usage breakdown to catch bloat early

-----

## Guardrails — apply throughout

**The cut test is non-negotiable.** Every line in CLAUDE.md must survive: “if I
remove this, Claude makes a mistake.” If it passes without it — cut it. This is
the single most effective way to keep CLAUDE.md lean.

**CLAUDE.md ≠ documentation.** It’s a context injection file. Facts and instructions
only. No explanations, tutorials, or philosophy.

**Under 200 lines is the target.** Adherence drops measurably above 300. When
you can’t fit under 200, move content to rules/ — don’t just tighten the wording.

**Rules ≠ Skills.** Rules are always-on constraints scoped to files or topics.
Skills are on-demand procedural workflows. Never put “how to do X step by step”
in a rule file.

**No-frontmatter rules load every session** like CLAUDE.md — they are not free.
Only use them for truly universal constraints. Everything file-specific gets
`paths:` frontmatter.

**Don’t duplicate config.** If a skill covers a topic, reference it from CLAUDE.md
with `@` or a plain pointer — don’t restate its content. Same for agents and commands.

**Lean rule files.** One topic per file, under 80 lines, name obvious from the
filename alone. If you need a `misc.md` or `general.md` rule file, that’s a sign
the content belongs in CLAUDE.md instead.

**Never auto-delete.** Always: propose → human confirms per item → execute.