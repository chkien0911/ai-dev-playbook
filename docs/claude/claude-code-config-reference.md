# Claude Code — Config Constructs Reference

> **Purpose:** Complete reference for every configuration construct in Claude Code.  
> Know what each one is, when it fires, how deterministic it is, and when to pick it over the others.

---

## Table of Contents

1. [CLAUDE.md](#1-claudemd)
2. [Rules / Instructions](#2-rules--instructions)
3. [Skills](#3-skills)
4. [Commands (Slash Commands)](#4-commands-slash-commands)
5. [Agents (Subagents)](#5-agents-subagents)
6. [Hooks](#6-hooks)
7. [Quick Decision Matrix](#7-quick-decision-matrix)
8. [Mental Model — 4 Layers](#8-mental-model--4-layers)
9. [Sample Project Structure](#9-sample-project-structure)

---

## 1. CLAUDE.md

### What
A Markdown file that Claude reads **automatically at the start of every session**. It is the project's persistent memory — architecture decisions, tech stack, conventions, and key commands that Claude needs to know without being told each time.

Claude searches for `CLAUDE.md` by walking up the directory tree from the current working directory, so nested `CLAUDE.md` files work for monorepos. There is also a global `~/.claude/CLAUDE.md` for personal cross-project preferences.

### Who configures it / who uses it
- **Configured by:** the developer, once.  
- **Used by:** Claude, automatically, every session — no trigger needed.

### When it activates
At session start, after session resume, and after context compaction. Claude also re-reads nested `CLAUDE.md` files when it navigates into subdirectories.

### Activation mechanism
**Automatic** — injected into the system prompt before any user message.

### Determinism
**Probabilistic.** Claude treats `CLAUDE.md` as contextual guidance, not hard law. A study on LLM instruction-following found frontier models reliably follow approximately 150–200 instructions before quality degrades. Claude Code's own system prompt already consumes ~50 slots, leaving ~100–150 for your `CLAUDE.md`. Rules buried in a long file get ignored.

> **Rule of thumb:** If compliance needs to be 100%, use a Hook instead.

### Use cases
- Document the tech stack, banned libraries, preferred patterns
- Record architectural decisions ("we use MassTransit, not raw RabbitMQ")
- List commands to build, test, and run the project
- Note project-specific idioms that Claude wouldn't know by default

### Anti-patterns
- Writing safety-critical rules here (use Hooks — they are deterministic)
- Files longer than ~150 meaningful lines (Claude starts ignoring the middle)
- Duplicating things Claude already does correctly without instruction

### Example

```markdown
# MyProject

## Tech Stack
- .NET 10 / ASP.NET Core — FastEndpoints, MassTransit, EF Core
- Frontend: React + TypeScript (Orval-generated clients)

## Key Rules
- Use `class` for FastEndpoints request DTOs — records cause source-gen fallback
- Always add `[FromRoute]` for route params when Orval separates them from body
- Fee rounding: Largest Remainder Method only (see docs/rounding.md)

## Commands
- Build: `dotnet build`
- Test:  `dotnet test --collect:"XPlat Code Coverage"`
```

---

## 2. Rules / Instructions

### What
Rules are natural-language instructions that guide Claude's behavior. In Claude Code they live in two places with different loading strategies:

| Location | Loading | Scope |
|---|---|---|
| Inside `CLAUDE.md` | Always — every session start | Global to the project |
| `.claude/rules/*.md` (no `paths`) | Always — same priority as `CLAUDE.md` | Global to the project |
| `.claude/rules/*.md` (with `paths`) | **On demand** — only when Claude reads a matching file | Path-scoped |

The `.claude/rules/` directory (added in Claude Code 2.0.64, December 2025) solves a key problem: putting all rules in one `CLAUDE.md` makes it bloated and causes Claude to ignore some of them. You can split rules into focused files, and use `paths` frontmatter to load domain-specific rules only when relevant.

**`paths`-scoped rules trigger when Claude reads a file matching the glob pattern — not on every tool call.** This means if Claude writes a new file without reading it first, the scoped rule may not be active. The official workaround is to ensure Claude reads at least one file in the directory before editing.

> **≈ Copilot:** `.claude/rules/*.md` with `paths` is the direct equivalent of `.github/instructions/*.instructions.md` with `applyTo`. Same concept, different syntax. Copilot triggers on file context in chat; Claude Code triggers when Claude reads a matching file during a session.

### Two containers — when to use which

| Want to | Use |
|---|---|
| Rules every session, always visible | `CLAUDE.md` (or `.claude/rules/` without `paths`) |
| Rules only for test files, consumers, or a specific directory | `.claude/rules/` with `paths` frontmatter |
| Rules that activate automatically when a task matches | Skill (`user-invocable: false`) |
| Rules that must hold 100% of the time | Hook (deterministic) |

### `.claude/rules/` frontmatter

```yaml
---
paths:
  - "src/**/Consumers/**"
  - "tests/**/*.test.ts"
---
```

- **No `paths`** → loaded unconditionally at session start, same as `CLAUDE.md`
- **With `paths`** → loaded on demand when Claude reads a file matching any glob
- Supports brace expansion: `"src/**/*.{ts,tsx}"`
- Files are discovered recursively — subdirectories within `.claude/rules/` are supported

### Good rule structure (applies everywhere)
1. **What** to do (or not do)
2. **Why** — context helps Claude apply the rule correctly in edge cases
3. **Example** when the pattern is non-obvious

### Determinism
**Probabilistic.** Claude interprets rules with judgment. Specific, example-backed rules are followed more reliably than vague ones. Rules that must hold 100% belong in Hooks, not here.

### Use cases
- `CLAUDE.md` → universal project standards (tech stack, build commands, bans)
- `.claude/rules/consumers.md` with `paths: ["src/**/Consumers/**"]` → MassTransit-specific conventions, loaded only when working in that directory
- `.claude/rules/tests.md` with `paths: ["**/*.test.cs"]` → xUnit patterns, naming conventions, fixture requirements
- `.claude/rules/api-design.md` (no `paths`) → API conventions that apply project-wide

### Anti-patterns
- Vague rules (`"write clean code"` — meaningless to Claude)
- Conflicting rules across files (Claude picks one non-deterministically)
- Mandatory enforcement rules (use Hooks instead)
- Very long rule files — keep individual files under 40,000 characters (Claude Code's documented limit)

### Example

```markdown
<!-- .claude/rules/consumers.md -->
---
paths:
  - "src/**/Consumers/**"
---

# MassTransit Consumer Rules

- Always check idempotency via distributed cache before processing any message.
  Pattern: `if (await _cache.GetAsync(context.Message.MessageId.ToString()) != null) return;`

- Consumers must implement `IConsumer<T>` — never call service methods directly.

- Register in `DependencyBuilder` under the correct bus endpoint after creation.
```

```markdown
<!-- .claude/rules/api-endpoints.md -->
<!-- No paths frontmatter → loaded globally, same priority as CLAUDE.md -->

# API Endpoint Rules

- Use `ICommandHandler<TCommand>` for all business operations — never call
  service methods directly from endpoints. Reason: consistent testability.

- Never use `Parallel.ForEach` with scoped services — it exhausts the
  connection pool. Use `Task.WhenAll` with explicit scope creation instead.
```

---

## 3. Skills

### What
A **Skill** is a directory containing a `SKILL.md` file (plus optional supporting files and scripts). It packages domain-specific knowledge, workflows, or procedures that Claude can load on demand.

**Two-level progressive disclosure:**
1. At startup, Claude reads only each skill's `name` and `description` from the YAML frontmatter — roughly ~100 tokens per skill. This is the discovery layer.
2. When Claude determines a skill is relevant to the current task, it reads the full `SKILL.md` body into the context window. Supporting files (templates, reference docs) are read only if the instructions reference them. Scripts are executed, and only their *output* enters context — the script source code does not.

This design means you can install dozens of skills without a context-window penalty on unrelated tasks.

### Skills vs. Commands
Previously, Claude Code had two systems: `commands` (`.claude/commands/*.md`) and `skills` (`.claude/skills/*/SKILL.md`). **They have since been merged** — both locations create the same `/slash-command` interface. Skills are the **recommended going-forward approach** because they support supporting files, richer frontmatter control, and progressive disclosure.

`.claude/commands/` files still work and are the simpler choice for one-off prompts with no supporting resources.

### Frontmatter fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Becomes the `/slash-command`. Lowercase, hyphens only, max 64 chars. |
| `description` | Recommended | How Claude discovers the skill. Written in third person. Include *what* it does **and** *when* to use it. |
| `allowed-tools` | No | Restrict which tools Claude can use inside this skill (e.g. `Read Grep` for read-only). |
| `disable-model-invocation` | No | `true` = only user can trigger via `/name`. Claude cannot auto-invoke. Use for destructive operations. |
| `user-invocable` | No | `false` = only Claude can invoke (background knowledge, never a slash command). |
| `context` | No | `fork` = run in an isolated subagent context so main conversation stays clean. |

### Who configures it / who uses it
- **Configured by:** the developer (or installed from a plugin/marketplace).  
- **Used by:** Claude automatically when intent matches description, **or** the developer via `/skill-name`.

### When it activates
- **Auto:** Claude detects the current task matches the skill's description.
- **Manual:** developer types `/skill-name` or `/skill-name <arguments>`.

### Activation mechanism
**Semi-automatic (probabilistic).** Claude uses LLM judgment to decide whether a skill is relevant. It can be wrong — if you need guaranteed execution, use a Hook.

### Determinism
**Probabilistic on auto-detection.** Manual invocation via `/name` is deterministic at the trigger level; execution is still LLM-based.

### Two types of skills
- **Capability Uplift:** gives Claude an ability it does not have natively (e.g. generating `.pptx` files, web scraping, browser automation).
- **Encoded Preference:** guides Claude to follow your team's specific workflow for something it already knows how to do (e.g. PR review checklist, commit message format, log classification template).

### Use cases
- Project-specific multi-step workflows (migration audits, release checklists)
- Reusable code generation templates with supporting reference files
- Domain knowledge packages (API conventions, rounding rules, naming standards)
- Tasks where consistency across sessions matters

### Anti-patterns
- Safety/security enforcement (use Hooks — probabilistic != safe)
- Simple one-liner prompts with no supporting resources (use a command instead)
- Rules you want active on every single task (put those in `CLAUDE.md`)

### Example

```
.claude/skills/create-masstransit-consumer/
├── SKILL.md
└── templates/
    └── consumer.cs.template
```

```markdown
---
name: create-masstransit-consumer
description: Generates a MassTransit consumer with idempotency check, distributed
  cache guard, and xUnit test stub. Use when asked to create a new message consumer
  or event handler.
allowed-tools: Read Write Bash
---

## Steps

1. Read `$ARGUMENTS` to identify the message type and namespace.
2. Load `templates/consumer.cs.template`.
3. Replace placeholders: `{{MessageType}}`, `{{Namespace}}`, `{{QueueName}}`.
4. Write the consumer to `src/Consumers/{{MessageType}}Consumer.cs`.
5. Create a placeholder xUnit test in `tests/Consumers/{{MessageType}}ConsumerTests.cs`.
6. Remind the developer to register the consumer in `DependencyBuilder`.
```

---

## 4. Commands (Slash Commands)

### What
Commands are **prompt templates stored in `.claude/commands/`** that the developer triggers explicitly by typing `/command-name`. They are the simpler predecessor to Skills — no supporting files, no progressive disclosure, just a Markdown prompt.

As noted above, `.claude/commands/` and `.claude/skills/` now share the same `/slash-command` interface. Use commands for short, standalone prompts; use Skills when you need supporting files, frontmatter control, or multi-step workflows.

### Frontmatter fields (same as Skills, simplified)

| Field | Use |
|---|---|
| `allowed-tools` | Restrict tools for this command |
| `context: fork` | Run in isolated subagent context |

Commands support `$ARGUMENTS` — text passed after the command name is substituted.

### Who configures it / who uses it
- **Configured by:** developer.  
- **Used by:** developer explicitly via `/command-name`.

### When it activates
Only when the developer types `/command-name`. Never auto-triggers.

### Activation mechanism
**Manual (user-initiated).**

### Determinism
**Deterministic trigger, probabilistic execution.** The `/name` always loads the exact prompt; Claude's interpretation of that prompt is still LLM-based.

### Use cases
- Frequently-typed prompts (PR review, write-test, deploy checklist)
- Team-shared workflows committed to the repo under `.claude/commands/`
- One-off tasks with no supporting files or templates

### Anti-patterns
- Workflows that need to auto-fire on events (use Hooks)
- Multi-step workflows with templates or reference files (use Skills)
- Rules that should always apply (put in `CLAUDE.md`)

### Example

```markdown
<!-- .claude/commands/review-pr.md -->
---
allowed-tools: Read Glob Grep
---

Review the current `git diff --staged` for:
- Bugs and logic errors
- Security issues (SQL injection, unvalidated input, exposed secrets)
- Missing idempotency checks on MassTransit consumers
- Correct use of `[FromRoute]` when route params are present
- FastEndpoints DTO type (must be `class`, not `record`)

Be specific — include file name and line number for each finding.
Arguments: $ARGUMENTS
```

Invoked as: `/review-pr focus on auth changes only`

---

## 5. Agents (Subagents)

### What
An **Agent** (also called a *subagent*) is a specialized worker that runs in its **own isolated context window**, with its own system prompt, tool restrictions, model choice, and permission mode. The main Claude session can spawn subagents to delegate complex or focused tasks — the main context remains clean.

Agents are defined as Markdown files with YAML frontmatter in `.claude/agents/`. Claude discovers them automatically and can invoke them when the task matches the description, or you can instruct Claude explicitly ("use a subagent to...").

### Frontmatter fields

| Field | Description |
|---|---|
| `name` | Agent identifier |
| `description` | **Critical** — Claude reads this to decide when to invoke the agent automatically |
| `tools` | Comma-separated allow list (e.g. `Read, Grep, Glob` for read-only) |
| `disallowedTools` | Block specific tools |
| `model` | `opus`, `sonnet`, `haiku`, or `inherit`. Use Haiku for fast/cheap agents, Opus for high-stakes review |
| `permissionMode` | `default`, `auto`, or `bypassPermissions` (use with caution) |
| `maxTurns` | Cap on how many turns the agent runs |
| `skills` | List of skill names to pre-load into the agent's context at startup |
| `hooks` | Hooks scoped to this agent only |
| `color` | UI color for visual distinction |

### Who configures it / who uses it
- **Configured by:** developer (or generated via `/agents` command).  
- **Used by:** Claude main session (auto-delegates) or the developer (explicit instruction).

### When it activates
- **Auto:** Claude's orchestrator matches the task to the agent's `description`.
- **Manual:** developer says "use a subagent to audit all 50 controllers."

### Activation mechanism
**Auto-delegation (description-matching) or explicit instruction.**

### Determinism
**Structurally isolated and deterministic in scope** — the agent runs in its own context, cannot contaminate main session state. Its internal behavior is still LLM-based.

### Why this matters — context window economics
After an hour of coding, your main context is full. Sending a research task to an agent means 50 files get read in a separate window; the main conversation never sees those tokens. The agent returns a clean summary. This is the primary reason to use agents.

### Use cases
- Auditing large numbers of files without filling main context
- Parallel work: builder agent + validator agent running simultaneously
- Specialized focus: security reviewer with read-only tools, no ability to modify files
- Research tasks that shouldn't pollute the main conversation

### Anti-patterns
- Simple tasks that fit in main context (overhead not worth it)
- When you need results interleaved with main conversation in real time
- When agents need to communicate mid-task (use experimental Agent Teams instead)

### Example

```markdown
<!-- .claude/agents/security-reviewer.md -->
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Invoke automatically after
  writing authentication, authorization, payment, or data-handling code. Also use
  when asked to "audit for security" or "check my auth logic".
tools: Read, Grep, Glob
model: opus
effort: high
---

You are a senior application security engineer.

Review code for:
- Injection vulnerabilities (SQL, command, XSS)
- Authentication and authorization flaws
- Secrets or credentials hardcoded in source
- Insecure data handling or deserialization

Output: one finding per line with file path, line number, severity (High/Medium/Low),
and a one-sentence fix recommendation.
```

---

## 6. Hooks

### What
Hooks are **shell commands, prompts, HTTP calls, or agent invocations that run automatically at specific lifecycle events** — completely outside the LLM. They do not ask Claude to do something; they do it themselves, unconditionally. This is the only mechanism in Claude Code that guarantees 100% enforcement.

Hooks are configured in `.claude/settings.json` under the `hooks` key, or `~/.claude/settings.json` for global scope.

### The core distinction
| | CLAUDE.md rules | Hooks |
|---|---|---|
| Enforcement | ~70% — Claude may ignore | 100% — always fires |
| Mechanism | LLM guidance | Shell/script outside LLM |
| Can block action | No | Yes (exit code 2) |
| Response to complexity | Degrades with context length | Constant |

### Lifecycle events

| Event | Cadence | Description |
|---|---|---|
| `SessionStart` | Once/session | Session begins, resumes, or context clears. Stdout is injected into Claude's context. |
| `SessionEnd` | Once/session | Session exits (any reason). |
| `UserPromptSubmit` | Once/turn | Immediately after user submits a prompt, before Claude processes it. |
| `PreToolUse` | Every tool call | **Before** a tool executes. **Only event that can block (exit 2).** |
| `PostToolUse` | Every tool call | After a tool completes successfully. Cannot undo the action. |
| `PostToolUseFailure` | Every tool failure | After a tool fails. Add context to help Claude recover. |
| `PermissionRequest` | On demand | When Claude would show a permission dialog. Auto-approve or auto-deny. |
| `Stop` | Once/turn | Claude finishes responding. Exit 2 forces Claude to continue. |
| `StopFailure` | On API error | Turn ended due to rate limit or auth failure. |
| `SubagentStart` / `SubagentStop` | Per agent | Subagent lifecycle. |
| `Notification` | Async | Claude sends a notification (idle, needs permission, etc.). |
| `PreCompact` / `PostCompact` | On compaction | Context compaction. Use `SessionStart` with matcher `compact` to re-inject critical context. |

### Handler types

| Type | When to use |
|---|---|
| `command` | Shell script. Simple, fast. Best for formatting, blocking, notifications. |
| `prompt` | LLM evaluation. Semantic checks ("is this a destructive bash command?"). |
| `agent` | Full subagent with tools. Deep analysis (e.g. check all files touched by a write). |
| `http` | POST to a remote server. Team-wide policy enforcement, audit logging. |

### Exit code semantics

| Exit code | Effect |
|---|---|
| `0` | Allow — continue normally |
| `2` in `PreToolUse` | **Block** the tool call entirely. Stderr message goes to Claude as explanation. |
| `2` in `Stop` | **Force Claude to continue** — it cannot finish the turn until exit 0. |
| Any other non-zero | Non-blocking error — execution continues, warning logged. |

### Who configures it / who uses it
- **Configured by:** developer in `settings.json`.  
- **Executed by:** Claude Code runtime — no human or LLM involved.

### When it activates
Automatically at the matching lifecycle event. Matcher is a **regex applied to the tool name** (for tool-based events), not to file paths. To filter by file path, inspect `tool_input.file_path` inside the script.

### Activation mechanism
**Fully automatic and deterministic.**

### Determinism
**100% deterministic.** Runs every time the event fires and the matcher matches, regardless of context length or Claude's current focus.

### Use cases
- Block dangerous shell commands unconditionally (`rm -rf`, `DROP TABLE`)
- Auto-format files on every write (Prettier, ESLint)
- Run tests before a commit is allowed (Stop hook + exit 2 if tests fail)
- Desktop notification when Claude goes idle
- Inject current git branch into Claude's context on session start
- Auto-approve known-safe permission requests (e.g. `npm run lint`)

### Anti-patterns
- Blocking hooks in the middle of a plan (e.g. on every `Edit` event) — confuses Claude mid-task. Prefer block-at-commit (Stop hook) instead.
- Too many slow hooks — each one pauses Claude's agentic loop.
- Using hooks for things where LLM judgment is fine (use `CLAUDE.md` instead).

### Example

```jsonc
// .claude/settings.json
{
  "hooks": {
    // Block rm -rf and DROP TABLE unconditionally
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$CLAUDE_TOOL_INPUT\" | grep -qE 'rm -rf|DROP TABLE' && exit 2 || exit 0"
          }
        ]
      }
    ],

    // Auto-format every file Claude writes or edits
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
          }
        ]
      }
    ],

    // Inject git branch context at session start
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"additionalContext\": \"Branch: '$(git branch --show-current)'\"}'"
          }
        ]
      }
    ],

    // Force Claude to run tests before finishing; block if they fail
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "INPUT=$(cat); [ \"$(echo $INPUT | jq -r '.stop_hook_active')\" = 'true' ] && exit 0; npm test || exit 2"
          }
        ]
      }
    ]
  }
}
```

---

## 7. Quick Decision Matrix

| Need | Use | Why |
|---|---|---|
| Claude always knows our tech stack and conventions | `CLAUDE.md` | Always-on persistent context |
| Global rules organized by topic, no path scoping | `.claude/rules/*.md` (no `paths`) | Same priority as `CLAUDE.md`, cleaner separation |
| Rules only active when working in a specific directory | `.claude/rules/*.md` with `paths` glob | On-demand, keeps main context lean |
| Prompt I type repeatedly | Command (`.claude/commands/`) | Manual trigger, no overhead |
| Multi-step workflow with templates/scripts | Skill (`.claude/skills/`) | Progressive disclosure, supporting files |
| Audit 50 files without filling main context | Agent (`.claude/agents/`) | Isolated context window |
| Prevent dangerous command 100% of the time | Hook `PreToolUse` + exit 2 | Only truly deterministic option |
| Auto-format after every file edit | Hook `PostToolUse` | Fires unconditionally |
| Notify me when Claude goes idle | Hook `Notification` | Async, non-blocking |
| Run tests before commit is allowed | Hook `Stop` + exit 2 on failure | Block Claude until tests pass |
| Rules that need AI judgment (soft guidelines) | `CLAUDE.md` or `.claude/rules/` | Advisory is fine here |

---

## 8. Mental Model — 4 Layers

Think of the system as four layers from softest to hardest:

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1 — Memory (always-on or path-triggered context)         │
│  CLAUDE.md + .claude/rules/                                     │
│  → Global rules always-on. Path-scoped rules load on demand    │
│    when Claude reads a matching file. Probabilistic.            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Knowledge Packages (on-demand)                       │
│  Skills + Commands                                              │
│  → "Saved playbooks." Available when needed; ignored otherwise. │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Delegation (isolated context)                        │
│  Agents / Subagents                                             │
│  → "Specialist contractors." Own context, own tools, own scope. │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4 — Enforcement (deterministic gates)                    │
│  Hooks                                                          │
│  → "CI/CD gates." No pass = no proceed. No exceptions.         │
└─────────────────────────────────────────────────────────────────┘
```

**The golden rule:**  
Use `CLAUDE.md` for what Claude *should* do. Use Hooks for what Claude *must* do.

---

## 9. Sample Project Structure

```
my-project/
│
├── CLAUDE.md                          # Always-on project memory
│                                      # Tech stack, conventions, build commands
│
├── .claude/
│   │
│   ├── settings.json                  # Hooks config (PreToolUse, PostToolUse, Stop...)
│   │
│   ├── rules/
│   │   ├── api-endpoints.md           # No `paths` → loaded globally like CLAUDE.md
│   │   ├── consumers.md               # paths: ["src/**/Consumers/**"] → load on demand
│   │   └── tests.md                   # paths: ["**/*.test.cs"] → load on demand
│   │
│   ├── agents/
│   │   ├── security-reviewer.md       # Read-only agent; auto-invoked on auth/payment code
│   │   └── migration-auditor.md       # Runs in isolation; audits dependency tree
│   │
│   ├── skills/
│   │   ├── create-masstransit-consumer/
│   │   │   ├── SKILL.md               # Instructions + frontmatter (name, description)
│   │   │   └── templates/
│   │   │       └── consumer.cs.template
│   │   │
│   │   ├── logging-audit/
│   │   │   ├── SKILL.md               # Classifies log statements into 8 categories
│   │   │   └── references/
│   │   │       └── categories.md      # Loaded only when skill is active
│   │   │
│   │   └── wpf-migration/
│   │       ├── SKILL.md               # .NET Framework 4.8 → .NET 10 migration guide
│   │       └── references/
│   │           └── dependency-matrix.md
│   │
│   └── commands/
│       └── review-pr.md               # /review-pr — quick PR review checklist
│                                      # (no supporting files needed → command, not skill)
│
└── .gitignore
```

### Key placement rules

| File / Folder | Scope | Loaded | Committed to repo? |
|---|---|---|---|
| `CLAUDE.md` (project root) | This project | Every session start | ✅ Yes — shared with team |
| `.claude/rules/*.md` (no `paths`) | This project | Every session start | ✅ Yes |
| `.claude/rules/*.md` (with `paths`) | This project | On demand when Claude reads matching file | ✅ Yes |
| `~/.claude/CLAUDE.md` | All projects | Every session start | ❌ No — personal preferences |
| `.claude/agents/*.md` | This project | When spawned | ✅ Yes |
| `.claude/skills/*/SKILL.md` | This project | On demand when task matches | ✅ Yes |
| `.claude/commands/*.md` | This project | When user types `/name` | ✅ Yes |
| `.claude/settings.json` | This project | Every session start | ✅ Yes (review before committing hooks) |
| `~/.claude/settings.json` | Global (all projects) | Every session start | ❌ No — personal hooks |
| `~/.claude/agents/` | Global | When spawned | ❌ No — personal agents |

---

*Sources: Claude Code official docs (code.claude.com), Anthropic engineering blog, Agent Skills API docs (platform.claude.com) — verified April 2026.*
