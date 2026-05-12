# Meta-Prompt: Create or Update `review-task` Skill

You are building or improving a `review-task` skill for this specific codebase.
Your goal is to produce a skill that lets Claude conduct deep, accurate, project-aware
code reviews — not generic ones. Work through the phases below in order.
Do not create or modify any files until Phase 4.

-----

## PHASE 1 — Orient yourself (silent, no output)

Read the following to build a working model of the project before doing anything else.

### 1a. Project structure & domain

- Top-level directory layout
- README, docs/, wiki/, or onboarding documents
- Solution/project files (`.sln`, `package.json`, `pyproject.toml`, `Cargo.toml`, etc.)
- Any domain glossary, ADRs (Architecture Decision Records), or design docs

### 1b. Existing AI configuration — read everything

- `CLAUDE.md` (root and any nested)
- `.claude/rules/*.md`
- `.claude/skills/*/SKILL.md` — check specifically for any existing `review` or `review-task` skill
- `.claude/commands/*.md`
- `.claude/agents/*.md`
- `.github/copilot-instructions.md`, `.github/prompts/`, `.github/instructions/`

### 1c. Architecture & tech stack signals

- Frameworks, libraries, major dependencies (infer from imports, lock files, config)
- Layering conventions: which projects/folders are API, domain, infrastructure, UI, tests
- DI registration patterns, error handling conventions, transaction boundaries
- Test project naming, test framework, test organisation patterns
- CI/CD config (workflows, pipelines, Dockerfiles)
- Database migration patterns and EF/ORM conventions (if applicable)

### 1d. Code patterns — find the non-obvious local conventions

Run targeted reads on representative files. Focus on finding things that:

- Deviate from framework defaults in a non-trivial way
- Are enforced by convention, not by the compiler
- Would cause a reviewer to say “that’s not how we do it here” if violated

Examples to look for (adapt to stack):

- Endpoint/controller structure and validation approach
- Command/query handler structure (CQRS if applicable)
- How errors are returned (ProblemDetails? custom envelope? exceptions?)
- How services are scoped and registered
- How events/messages are published and consumed
- Naming rules for files, classes, methods, tests
- How tests are structured (Arrange/Act/Assert? builders? fixtures?)
- Any shared utilities or base classes that must be used

### 1e. Domain knowledge

- What business domain does this codebase serve?
- Key entities, aggregates, or bounded contexts visible in the code
- Any domain rules enforced in code that a reviewer should know
  (e.g. “an order cannot transition from X to Y directly”)

### 1f. Git baseline detection

Run:

```bash
git branch -a | grep -E 'main|master|develop' | head -10
git remote show origin 2>/dev/null | grep 'HEAD branch'
```

Identify the likely default branch (main, master, or develop). You will use this
as the comparison base when generating diffs unless the user overrides it.

-----

## PHASE 2 — Discovery Report (output this)

Produce a **Discovery Report** with the following sections.
Be specific — generic observations are not useful.

### 2a. Project snapshot

- Stack (languages, frameworks, major libraries)
- Architecture style and layering
- Test strategy (framework, naming, structure)
- Domain summary (what the system does, key entities)
- Detected default branch for diff comparison

### 2b. Non-obvious conventions a reviewer must know

For each convention:

- **What it is** — specific rule or pattern
- **Where it lives** — file paths or naming patterns that demonstrate it
- **What goes wrong if violated** — concrete consequence (runtime error, data corruption,
  test false-positive, contract break, etc.)

Only include conventions that are genuinely non-obvious to an AI working from
framework knowledge alone. Skip anything Claude would get right without guidance.

### 2c. Domain rules visible in code

List business rules enforced in code that affect review correctness.
A reviewer who misses these would approve incorrect logic.

### 2d. Existing skill check

- Does `.claude/skills/review-task/SKILL.md` exist? If yes, show its content and
  assess: what does it cover well, what is missing, what is stale?
- Does any other skill or command partially cover code review? List them.

-----

## PHASE 3 — Confirm with the developer (ask, then wait)

Before writing any files, ask the developer these questions.
Present as a numbered list.

```
Before I build the skill, I want to validate a few things:

1. My discovery shows [detected branch] as the default comparison branch.
   Is that correct, or should I use a different one as the default?

2. Are there review concerns that come up repeatedly that I haven't captured —
   things you always check manually because Claude gets them wrong?

3. Are there areas of the codebase that are especially high-risk for regressions
   (shared utilities, payment flows, auth, event consumers, etc.)?
   I want to weight the impact-analysis section appropriately.

4. Do you want the skill to attempt Jira integration when a ticket is provided?
   (I'll include logic to read ticket content if a ticket ID or URL is supplied,
   and skip that section if not.)

5. Looking at the conventions I listed in the Discovery Report — anything wrong,
   missing, or that you want emphasised more strongly in the review checklist?

6. Any output format preferences beyond GitHub-style PR comments with severity labels?
   (e.g. summary table at the top, grouping by file vs. by concern, etc.)
```

Incorporate all answers before writing any files.

-----

## PHASE 4 — Build the skill

### File layout to create

```
.claude/skills/review-task/
├── SKILL.md                         ← required entrypoint
└── references/
    ├── review-output-template.md    ← GitHub-style output format with severity guide
    ├── conventions.md               ← project-specific conventions checklist
    └── domain-rules.md              ← domain/business rules to validate against
```

Only create `references/` files that will actually contain project-specific content.
If a file would just restate generic framework knowledge, omit it.

-----

### SKILL.md — what it must contain

#### Frontmatter

```yaml
---
name: review-task
description: >
  Conduct a structured code review for a feature branch or set of file changes in
  this codebase. Use when the user asks to review a branch, PR, diff, or specific
  files — or phrases like "review my changes", "check this before I raise a PR",
  "can you review [branch]", or "review ticket [ID]". Covers requirement validation,
  code quality, impact analysis, regression risk, integration concerns,
  non-functional issues, and test coverage. Outputs GitHub-style PR comments with
  severity labels and suggested fixes.
---
```

#### Body — ordered instructions Claude must follow

**Step 1 — Establish scope**

```
Determine what to review:

a) If the user specifies a branch name:
   - Determine the comparison base:
     1. If the user stated the base branch explicitly → use it
     2. Otherwise run: git branch -a | grep -E '^(\*| ) (main|master|develop)$'
        and use the first match
     3. If still ambiguous, ask: "I couldn't determine the default branch.
        Which branch should I compare against — main, master, or develop?"
   - Run: git diff <base>..<feature-branch> --name-only
     then: git diff <base>..<feature-branch> -- <files>
   - Also run: git log <base>..<feature-branch> --oneline
     to understand commit history and intent

b) If the user provides specific files or a diff directly → use that as scope

c) If a Jira ticket ID or URL is provided:
   - Read the ticket content (description, acceptance criteria, sub-tasks)
   - Use it as the requirement source for Section 1 of the review
   - If no ticket is provided, note "No ticket provided — reviewing against
     inferred intent from code changes and any user-supplied context"
```

**Step 2 — Load project context**

```
Before reviewing, read:
- references/conventions.md        → local conventions checklist
- references/domain-rules.md       → business rules to validate against
- CLAUDE.md (root)                 → project-wide constraints

Do not rely on general framework knowledge for convention checks.
Use only what is documented in these files plus what you observe in the diff.
```

**Step 2.5 — Determine which sections apply (adaptive scope)**

```
Before starting the review, scan the diff and determine which sections are
relevant. Skip sections that have no surface area in the changes.

Use this decision table:

  Section 1 — Requirement & Logic     → ALWAYS apply
  Section 2 — Code Quality & Design   → ALWAYS apply
  Section 3 — Impact Analysis         → ALWAYS apply
  Section 4 — Regression Risk         → ALWAYS apply
  Section 5 — System & Integration
    └─ API contract checks            → only if endpoint signatures / DTOs changed
    └─ DB / query checks              → only if EF entities, migrations, or raw
                                         queries are in the diff
    └─ Message/event schema checks    → only if consumers, publishers, or message
                                         contracts are in the diff
    └─ Config / dependency checks     → only if .csproj, appsettings, DI
                                         registrations, or Dockerfiles changed
  Section 6 — Performance             → only if the diff touches query paths,
                                         loops over collections, or adds I/O
  Section 7 — Security                → only if the diff touches auth, input
                                         handling, data exposure, or external calls
  Section 8 — Concurrency & Async     → only if the diff introduces async/await,
                                         locks, shared state, or background workers
  Section 9 — Backward Compatibility  → only if the diff changes public API
                                         signatures, serialized payloads, event
                                         contracts, or shared library interfaces
  Section 10 — Deployment & Ops       → only if the diff touches migrations,
                                         feature flags, config, infra, or
                                         environment-specific behaviour
  Section 11 — Testing & Observability → ALWAYS apply

State the active sections at the start of the review output:
  "Reviewing sections: 1, 2, 3, 4, 5 (API contract, DB), 7, 11"
  "Skipping sections: 6, 8, 9, 10 — no relevant surface area in this diff"

This prevents generating empty boilerplate sections and keeps the review focused.
```

**Step 3 — Conduct the review across active sections**

```
Work through each active section in order. For each finding, assign a severity:

  [CRITICAL]  — Must fix before merge. Correctness, security, data integrity,
                contract break, or production risk.
  [MAJOR]     — Should fix before merge. Logic gap, missing edge case,
                significant convention violation, meaningful regression risk.
  [MINOR]     — Recommended improvement. Style, readability, test coverage gap,
                non-urgent refactor.
  [NIT]       — Optional polish. Naming, formatting, minor cleanup.
  [INFO]      — Observation with no action required. Noted for awareness.

Section 1 — Requirement & Logic Validation
  - If a ticket was provided: does the implementation match the acceptance criteria?
    List each criterion and whether it is met, partially met, or missing.
  - Are there missing cases or incorrect logic relative to the ticket or inferred intent?
  - Are edge cases handled? (null inputs, empty collections, concurrent access,
    boundary values, locale/timezone if applicable)
  - Does any business rule from references/domain-rules.md get violated?

Section 2 — Code Quality & Design
  - Is the code readable and maintainable without needing extra context?
  - Does it follow the patterns in references/conventions.md?
    Flag deviations specifically — cite the convention being violated.
  - Is there unnecessary complexity, duplication, or over-engineering?
  - Are abstractions at the right level (not too early, not missing)?

Section 3 — Impact Analysis  ← weight this heavily
  - Which existing components does this change touch beyond the obvious diff?
    (shared utilities, base classes, interfaces, config, DI registrations)
  - Does modifying shared code change behaviour for callers not in this diff?
  - Are there implicit contracts (event payloads, API response shapes,
    database schema assumptions) that downstream consumers depend on?
  - Flag any change where the blast radius is larger than it appears.

Section 4 — Regression Risk
  - Could any change break existing functionality that is not covered by the
    tests included in this diff?
  - Look specifically at: method signature changes, removal of fields,
    changed default values, altered control flow in shared paths.
  - Are there behaviour changes that the test suite would not catch?

Section 5 — System & Integration  (apply only relevant sub-checks)
  - API contract: added/removed/renamed fields, changed status codes, error shapes?
  - DB/queries: index impact, N+1 risk, migration safety, data loss on rollback?
  - Events/messages: schema changes that break consumers not in this diff?
  - Config/dependencies: changes that affect deployment, secrets, or other services?

Section 6 — Performance  (apply only if diff touches query paths or I/O)
  - Unnecessary or duplicated queries (N+1, missing .AsNoTracking())?
  - Missing pagination on collections that can grow unbounded?
  - Large in-memory materialisation (ToList() on large sets)?
  - Synchronous blocking in async paths (.Result, .Wait(), blocking I/O)?
  - Expensive operations inside loops?

Section 7 — Security  (apply only if diff touches auth, input, or external calls)
  - Input validation: is user-supplied data validated before use?
  - Authorisation: are permission checks present and correct?
  - Sensitive data: any secrets, PII, or credentials logged or exposed in responses?
  - Injection: SQL, command, or path injection risk in dynamic queries or shell calls?
  - External calls: are timeouts, retries, and failure handling in place?

Section 8 — Concurrency & Async  (apply only if diff introduces async or shared state)
  - Async/await correctness: missing await, fire-and-forget without handling,
    ConfigureAwait usage where required by project convention?
  - Deadlock risk: .Result or .Wait() called inside async context?
  - Shared mutable state: static fields, singleton state mutated concurrently?
  - Race conditions: check-then-act patterns without proper locking or transactions?
  - Background worker lifecycle: is cancellation handled? Are tasks awaited?

Section 9 — Backward Compatibility  (apply only if public contracts changed)
  - Public API: are any existing consumers broken by signature or shape changes?
  - Serialized payloads: are renamed/removed fields handled with [JsonPropertyName]
    or equivalent to avoid breaking existing clients?
  - Event/message contracts: are consumers of this event in other services or
    workers still compatible with the new shape?
  - Shared library interfaces: is the change additive, or does it force callers
    to update?

Section 10 — Deployment & Ops  (apply only if infra or config changed)
  - Migration safety: can the migration run against production data without
    downtime or data loss? Is it reversible?
  - Feature flags: if this change affects live users, is it gated?
  - Config changes: are environment-specific values updated in all target
    environments (dev, staging, prod)?
  - Zero-downtime: if both old and new code may run simultaneously during
    a rolling deploy, is the change safe in both states?

Section 11 — Testing & Observability
  - Are tests meaningful — asserting on behaviour, not just code paths?
  - Are the important edge cases from Section 1 covered by tests?
  - Is error handling adequate and tested (not just happy-path coverage)?
  - Is logging sufficient to diagnose failures in production?
    (structured logging, correct log levels, no sensitive data in log messages)
  - Are there missing tests for the sections that had [CRITICAL] or [MAJOR] findings?
```

**Step 4 — Write output to file and report**

```
Determine the output path:
  tasks/review-<branch-name>-<YYYY-MM-DD>.md
  (If no branch name is available, use: tasks/review-<YYYY-MM-DD>.md)

Create the tasks/ directory if it does not exist.
Write the full review output to this file using references/review-output-template.md.

After writing, tell the user:
  "Review written to tasks/review-<filename>.md"
  Then print the Review Summary block inline in the chat so they see the
  verdict immediately without opening the file.

Rules for the file content:
- Open with the active/skipped sections declaration from Step 2.5
- Group findings by section number, ordered as reviewed
- Within each section, order findings by severity descending (CRITICAL first)
- Every finding must include: severity label, location (file:line or file range),
  description of the problem, and a concrete suggested fix or question
- If an active section has no findings, write "✅ No issues found."
- End with the Review Summary block as defined in references/review-output-template.md
- Do not pad with praise. Keep findings actionable.
```

-----

### references/review-output-template.md — what it must contain

This file defines the exact output format. Include:

**1. Per-finding format**

```markdown
**[SEVERITY] `path/to/file.cs` (line N)**
> Brief one-line description of the issue

**Problem:** Clear explanation of what is wrong and why it matters.

**Suggested fix:**
\`\`\`language
// concrete code suggestion or pseudocode
\`\`\`

**Why:** Rationale — link to convention, domain rule, or risk being mitigated.
```

**2. Severity reference table**

|Label       |Meaning                                                       |Merge gate?              |
|------------|--------------------------------------------------------------|-------------------------|
|`[CRITICAL]`|Correctness, security, data integrity, or production risk     |Block merge              |
|`[MAJOR]`   |Logic gap, missing edge case, significant convention violation|Block merge (recommended)|
|`[MINOR]`   |Improvement recommended, non-blocking                         |Author discretion        |
|`[NIT]`     |Optional polish                                               |Author discretion        |
|`[INFO]`    |Observation only                                              |No action needed         |

**3. Review Summary block** (always at the end)

```markdown
---
## Review Summary

**Sections reviewed:** [e.g. 1, 2, 3, 4, 5 (API, DB), 7, 11]
**Sections skipped:** [e.g. 6, 8, 9, 10 — no relevant surface area]

| # | Section | Findings | Highest Severity |
|---|---|---|---|
| 1 | Requirement & Logic | N | [label or —] |
| 2 | Code Quality & Design | N | [label or —] |
| 3 | Impact Analysis | N | [label or —] |
| 4 | Regression Risk | N | [label or —] |
| 5 | System & Integration | N | [label or —] |
| 6 | Performance | skipped / N | [label or —] |
| 7 | Security | skipped / N | [label or —] |
| 8 | Concurrency & Async | skipped / N | [label or —] |
| 9 | Backward Compatibility | skipped / N | [label or —] |
| 10 | Deployment & Ops | skipped / N | [label or —] |
| 11 | Testing & Observability | N | [label or —] |

**Overall verdict:** APPROVED / APPROVED WITH MINOR COMMENTS / CHANGES REQUESTED / BLOCKED

**Merge recommendation:** [one sentence — what must happen before this can merge]
```

**4. Jira section format** (used only when ticket was provided)

```markdown
## Ticket Coverage: [TICKET-ID]

| Acceptance Criterion | Status |
|---|---|
| [criterion text] | ✅ Met / ⚠️ Partially met / ❌ Not implemented |

**Gaps:** [list anything from the ticket not addressed in the diff]
```

-----

### references/conventions.md — what it must contain

Populate this file entirely from what you discovered in Phase 1 and confirmed
with the developer in Phase 3.

Structure each entry as:

```markdown
### [Convention name]

**Rule:** [Specific, imperative statement of the convention]
**Where it applies:** [File types, layers, or scenarios]
**Evidence:** [File paths or patterns that demonstrate it]
**Review check:** [What to look for in a diff to detect a violation]
**Consequence if violated:** [What breaks or degrades]
```

Only include conventions that are non-obvious and project-specific.
Do not document things Claude would get right from general framework knowledge.

-----

### references/domain-rules.md — what it must contain

Populate from Phase 1 discovery and Phase 3 developer answers.

Structure each entry as:

```markdown
### [Rule name]

**Rule:** [Precise statement of the business rule]
**Enforced where:** [File(s) or layer(s) where this is implemented]
**Review check:** [What a diff must preserve or implement correctly]
**Risk if violated:** [Data integrity / incorrect billing / wrong state transition / etc.]
```

-----

## Guardrails

**Conventions file must be project-specific.**
If a convention would be correct for any project using the same framework,
it does not belong in `references/conventions.md`. It belongs in Claude’s
training — omit it.

**Domain rules file must reflect actual code.**
Do not invent business rules. Only document rules you found enforced in the
existing codebase. If you are unsure whether a rule exists, ask the developer
in Phase 3.

**SKILL.md must stay lean.**
Target under 200 lines for the body. The section checklist stays in SKILL.md
because Claude needs it in context during every review. Everything else
(output template, conventions table, domain rules) goes in references/
and is explicitly loaded in Step 2.

**Adaptive scope is not optional.**
The Step 2.5 decision table must be applied on every run. Never generate
all 11 sections unconditionally — empty sections with “no issues found”
for sections that have no surface area in the diff add noise and erode trust.

**Output goes to `tasks/`.**
Every review must be written to `tasks/review-<branch>-<date>.md`.
The Review Summary block is also printed inline in chat for immediate visibility.
If the `tasks/` directory does not exist, create it.

**Never fabricate findings.**
If you are uncertain whether something is a violation, flag it as `[INFO]`
with a question rather than asserting it as `[MAJOR]`. A false positive in a
code review erodes trust faster than a missed issue.

**Always confirm base branch.**
Never assume. If detection fails and the user has not told you, ask before
running any diff commands.

**CREATE or UPDATE.**
If `.claude/skills/review-task/SKILL.md` already exists:

- Show the current content
- Diff it against what you would write
- Propose specific changes with rationale
- Do not overwrite until the developer confirms
  If it does not exist, create all files from scratch.

-----

## After completing all files

Confirm each file written:

> “Created/updated [path]. Move to next item?”

Then end with:

> “review-task skill is ready. To use it, run `/review-task` or ask Claude to
> review a branch by name. If you supply a Jira ticket ID, it will validate
> against acceptance criteria. The output will follow the GitHub comment format
> defined in references/review-output-template.md.”