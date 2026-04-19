# CLAUDE.md & Rules — Best Practices

> **Sources:** Anthropic official documentation ([code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)), [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices), [code.claude.com/docs/en/claude-directory](https://code.claude.com/docs/en/claude-directory), and authoritative community references (ClaudeFast Rules Directory Guide, GitHub: shanraisshan/claude-code-best-practice).

---

## What Is CLAUDE.md?

`CLAUDE.md` is a markdown file that gives Claude persistent instructions across sessions. Because each Claude Code session begins with a fresh context window, CLAUDE.md is how you carry knowledge forward — build commands, coding standards, project layout, and "always do X" rules.

> **Anthropic guidance:** Treat CLAUDE.md as the place you write down what you'd otherwise re-explain. Add to it when Claude makes the same mistake a second time, or when you type the same correction into chat that you typed last session.

CLAUDE.md is **context**, not enforced configuration. Claude reads it, weighs it as high-priority instruction, and acts accordingly. The more specific and concise your instructions, the more consistently Claude follows them.

---

## Two Complementary Memory Systems

Claude Code provides two systems for persistent memory. Both load at the start of every session.

| | CLAUDE.md files | Auto memory |
|---|---|---|
| **Who writes it** | You | Claude |
| **What it contains** | Instructions and rules | Learnings and patterns Claude discovers |
| **Scope** | Project, user, or org | Per working tree |
| **Loaded into** | Every session, in full | Every session (first 200 lines or 25 KB) |
| **Use for** | Coding standards, workflows, architecture decisions | Build commands, debugging insights, preferences Claude picks up |

Use CLAUDE.md when you want to explicitly guide Claude's behavior. Auto memory lets Claude learn from your corrections without manual effort.

---

## Where CLAUDE.md Files Live

CLAUDE.md files can exist at multiple scopes. More specific locations take precedence over broader ones.

| Scope | Location | Purpose | Shared with |
|---|---|---|---|
| **Managed policy** | System-level (OS-dependent) | Organization-wide standards enforced by IT/DevOps | All users in org |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team-shared project instructions | Team via version control |
| **User** | `~/.claude/CLAUDE.md` | Personal preferences across all projects | Just you |
| **Local** | `./CLAUDE.local.md` (gitignored) | Personal project-specific preferences | Just you, not committed |

CLAUDE.md and CLAUDE.local.md files are discovered by walking up the directory tree from your working directory. All discovered files are **concatenated into context** — they do not override each other. Within each directory, `CLAUDE.local.md` is appended after `CLAUDE.md`, so personal notes always win on conflict at that level.

---

## The `.claude/rules/` Directory

For larger projects, `.claude/rules/` is the recommended way to organize instructions into modular, maintainable files instead of one monolithic CLAUDE.md.

```
.claude/
├── CLAUDE.md           # Universal instructions — always loads
└── rules/
    ├── code-style.md   # Always loads (no paths frontmatter)
    ├── testing.md      # Always loads
    ├── security.md     # Always loads
    └── api/
        └── endpoints.md  # Loads only when working on matching files
```

Every `.md` file in `.claude/rules/` is discovered automatically — no registration needed. Files load with **the same high priority as CLAUDE.md itself**.

### Why Rules Exist: The Monolithic Problem

When everything is in one CLAUDE.md file, all of it receives high-priority attention simultaneously. React patterns compete with API guidelines compete with database rules — even when you are only working on a migration file.

> **Key insight from community research:** High priority everywhere = priority nowhere. When everything is marked important, Claude struggles to determine what is actually relevant to the current task. Instructions get ignored, context becomes noisy, and behavior becomes unpredictable.

Rules solve this by distributing high-priority instructions across targeted files. Your API rules still get high priority — but only when you are working on API files.

### Path-Specific Rules

Rules can be scoped to file patterns using YAML frontmatter. These rules **only load when Claude works with matching files**.

```yaml
---
paths:
  - src/api/**/*.ts
  - src/api/**/*.cs
---

# API Development Rules

- All endpoints must validate input before processing
- Return consistent error format: `{ "error": { "code": "...", "message": "..." } }`
- Include structured logging on entry and exit of every handler
- Never expose internal stack traces in API responses
```

Rules without a `paths` field load unconditionally every session, just like CLAUDE.md.

### Priority Hierarchy at a Glance

| Source | Priority | Loads when |
|---|---|---|
| **CLAUDE.md** | High | Every session, always |
| **Rules (no paths)** | High | Every session, always |
| **Rules (with paths)** | High | Only when working on matching files |
| **Skills** | Medium (on-demand) | When triggered by relevance or `/skill-name` |
| **Conversation history** | Variable | Accumulates during session, decays after compaction |
| **File contents (Read tool)** | Standard | Normal context weight |

---

## ✅ SHOULD — Design Guidelines

### 1. Start with `/init` and refine from there

Run `/init` to generate a starter CLAUDE.md based on your actual codebase. Claude analyzes build systems, test frameworks, and code patterns to create a solid foundation. Refine from there with instructions Claude would not discover on its own.

Set `CLAUDE_CODE_NEW_INIT=1` for an interactive multi-phase flow that asks which artifacts to set up (CLAUDE.md, skills, hooks) and presents a reviewable proposal before writing any files.

### 2. Keep CLAUDE.md under 200 lines

> **Official Anthropic guidance:** Target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence. Important rules get lost in the noise.

If your file is growing large, the fix is not to trim randomly — it is to **extract procedures and domain-specific rules** into skills and `.claude/rules/` files respectively.

### 3. Write specific, verifiable instructions

Vague instructions are ignored. Specific instructions are followed.

| ❌ Vague | ✅ Specific |
|---|---|
| "Format code properly" | "Use 2-space indentation, no trailing spaces" |
| "Test your changes" | "Run `dotnet test` before every commit" |
| "Keep files organized" | "API handlers live in `src/api/handlers/`" |
| "Handle errors correctly" | "Use `ThrowError()` for validation failures, never throw raw exceptions" |

### 4. Use markdown headers and bullets

Claude scans structure the same way readers do. Organized sections with clear headers are easier to follow than dense paragraphs. Group related instructions under a single header.

```markdown
# Build & Test Commands
- Build: `dotnet build`
- Test: `dotnet test --collect:"XPlat Code Coverage"`
- Format: `dotnet format`

# Code Conventions
- Use `var` for local variables with obvious types
- Prefer expression-bodied members for single-line methods
- Never use `dynamic`
```

### 5. Use `@import` syntax for external files

CLAUDE.md can import additional files using `@path/to/file` syntax. Imported files are expanded and loaded into context at launch. Use this to reference READMEs, package specs, or shared guides without duplicating them.

```markdown
See @README.md for project overview.
See @package.json for available scripts.

# Git workflow
@docs/git-workflow.md
```

### 6. Use path-specific rules for domain concerns

Extract domain-specific instructions that only apply to certain file types into path-targeted rules. This keeps them out of context when irrelevant.

```yaml
# .claude/rules/security.md
---
paths:
  - src/auth/**/*
  - src/payments/**/*
---

# Security-Critical Code Rules

- Never log sensitive data: passwords, tokens, card numbers, PII
- Validate all inputs at function boundaries
- Use parameterized queries exclusively — never string interpolation
- Require explicit authorization check before any data access
```

### 7. Use HTML comments for maintainer notes

Block-level HTML comments in CLAUDE.md are stripped before injection into Claude's context. Use them to leave notes for human maintainers without spending tokens.

```markdown
<!-- Last reviewed: 2026-01 — rules for the legacy WPF stack are in rules/wpf-legacy.md -->

# Project Overview
...
```

### 8. Use `CLAUDE.local.md` for personal, non-committed preferences

Keep personal sandbox URLs, test data preferences, and local tool paths in `CLAUDE.local.md`. Add it to `.gitignore`. It loads alongside `CLAUDE.md` at the same scope and personal notes win on conflict.

### 9. Periodically audit and prune

Review CLAUDE.md and rules files for:
- **Contradictions** — if two rules conflict, Claude may pick one arbitrarily
- **Redundancy** — if Claude already does something correctly without the instruction, delete it or convert it to a hook
- **Scope creep** — if an entry has become a multi-step procedure, move it to a skill

### 10. Check project files into version control

Project CLAUDE.md (`.claude/CLAUDE.md` or `./CLAUDE.md`) and `.claude/rules/` files should be committed so the whole team benefits from the same context. Treat them as code — review changes, track history, roll back mistakes.

---

## ❌ SHOULD NOT — Common Mistakes

### 1. Do not write a CLAUDE.md over 200 lines

> **Official Anthropic guidance:** "The over-specified CLAUDE.md. If your CLAUDE.md is too long, Claude ignores half of it because important rules get lost in the noise. Fix: Ruthlessly prune."

The correct fix is extraction — move procedures to skills, move domain rules to `.claude/rules/`.

### 2. Do not put everything in one monolithic file

A 400-line CLAUDE.md with React patterns, API conventions, DB migration rules, and security requirements all mixed together causes **priority saturation**. Everything is high priority, so nothing is.

Extract into modular rules:
```
# Before: one 400-line CLAUDE.md
# After:
.claude/
├── CLAUDE.md               # ~50 lines: universal facts only
└── rules/
    ├── testing.md           # loads always
    ├── security.md          # loads always
    └── api-endpoints.md     # paths: src/api/**
```

### 3. Do not write vague instructions

Vague instructions are treated as suggestions. Specific, verifiable instructions are treated as rules. "Handle errors correctly" does nothing. "Use `ThrowError()` for 4xx validation failures, never throw raw exceptions" is actionable.

### 4. Do not put multi-step procedures in CLAUDE.md

CLAUDE.md is for facts and rules — always-applicable, session-level context. If an instruction is a multi-step procedure (deploy, run coverage, generate migrations), it belongs in a **skill**. Skills load on demand; CLAUDE.md loads every session whether or not the procedure is relevant.

### 5. Do not use MUST/CRITICAL/IMPORTANT as a substitute for specificity

Capslock emphasis does not reliably increase adherence. Specificity does. Instead of `MUST ALWAYS run tests`, write `Run dotnet test before every commit. Do not proceed to the next step if tests fail.`

### 6. Do not ignore conflicting rules across files

In a monorepo, CLAUDE.md files from other teams' directories may get picked up and create conflicts. Use `claudeMdExcludes` in settings to skip irrelevant files, and run `/memory` to audit what actually loaded in the current session.

### 7. Do not duplicate instructions between CLAUDE.md and rules files

If an instruction is in both CLAUDE.md and a rules file, it consumes double the tokens and can create inconsistencies over time. Pick one location per instruction.

---

## Decision Matrix — CLAUDE.md vs Rules vs Skills

| Instruction type | Where it belongs |
|---|---|
| Build commands, test commands | `CLAUDE.md` |
| Project-wide conventions that always apply | `CLAUDE.md` |
| Domain-specific rules for specific file types | `.claude/rules/` with `paths` frontmatter |
| Topic-organized rules without path scope | `.claude/rules/` (no frontmatter) |
| Multi-step procedures (deploy, migrate, review) | Skills |
| Rarely-needed reference material | Skills (load on demand) |
| Personal preferences not for team sharing | `CLAUDE.local.md` |
| Personal preferences across all projects | `~/.claude/CLAUDE.md` |

---

## Common Patterns

### Pattern 1: Minimal CLAUDE.md + Rules for Everything Domain-Specific

```markdown
# CLAUDE.md (~50 lines)

## Commands
- Build: `dotnet build`
- Test: `dotnet test`
- Lint: `dotnet format --verify-no-changes`

## Architecture
- Dual-stack: .NET Framework 4.8 (legacy) and .NET 8+ (new API)
- All new features go into the .NET 8+ layer
- Do not add new code to the .NET Framework 4.8 layer

## Source Control
- Never commit directly to main
- Branch naming: `feat/`, `fix/`, `chore/`
```

```yaml
# .claude/rules/api-endpoints.md
---
paths:
  - src/**/Endpoints/**/*.cs
  - src/**/Controllers/**/*.cs
---

# API Endpoint Rules
- Use FastEndpoints conventions for new endpoints
- Annotate route params with [FromRoute] when client uses Orval-generated TypeScript
- Use ThrowError() for validation failures, never throw raw exceptions
- Include Datadog structured logging on entry and exit
```

```yaml
# .claude/rules/payments.md
---
paths:
  - src/**/Payments/**/*
  - src/**/Billing/**/*
---

# Payment Code Rules
- Never log card numbers, tokens, or any PII
- Stripe operations must be idempotent — always pass idempotency key
- All payment state changes must emit a domain event via MassTransit
```

### Pattern 2: Importing AGENTS.md for Multi-Tool Repos

If your repo already uses `AGENTS.md` for other AI tools, import it from CLAUDE.md rather than duplicating:

```markdown
# CLAUDE.md
@AGENTS.md

## Claude Code Specific
- Use plan mode before modifying anything in `src/billing/`
- Prefer subagents for research tasks that read 10+ files
```

---

## Useful Commands

| Command | What it shows |
|---|---|
| `/init` | Generate starter CLAUDE.md from codebase analysis |
| `/memory` | Which CLAUDE.md and rules files loaded + auto-memory entries |
| `/context` | Token usage breakdown: system prompt, memory files, skills, messages |
| `/compact` | Summarize conversation to free context space |

---

## Pre-Update Checklist

Before adding a new instruction to CLAUDE.md, answer these questions:

```
□ Is this a fact or rule that applies to every session? → CLAUDE.md or rules/
□ Does it only apply to specific file types? → rules/ with paths frontmatter
□ Is it a multi-step procedure? → Skill
□ Is it a personal preference not for team sharing? → CLAUDE.local.md
□ Is CLAUDE.md already over 200 lines? → Extract first, then add
□ Does a similar instruction already exist? → Update existing, don't duplicate
□ Is it specific enough to verify? ("Run dotnet test" vs "test your changes")
□ Would deleting it change Claude's behavior? If not — delete it
```

---

## References

- [Anthropic — How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Anthropic — Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic — Explore the .claude directory](https://code.claude.com/docs/en/claude-directory)
- [ClaudeFast — Rules Directory: Modular Instructions That Scale](https://claudefa.st/blog/guide/mechanics/rules-directory)
- [GitHub — shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
