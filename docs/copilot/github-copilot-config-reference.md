# GitHub Copilot — Config Constructs Reference

> **Purpose:** Complete reference for every configuration construct in GitHub Copilot (VS Code, CLI, Cloud Agent, Visual Studio 2026).  
> Know what each one is, when it fires, how deterministic it is, and when to pick it over the others.  
> Where a concept is identical or near-identical to Claude Code, a **≈ Claude Code** callout is provided so you can map across tools without re-reading from scratch.

---

## Table of Contents

1. [copilot-instructions.md](#1-copilot-instructionsmd)
2. [Path-Specific Instructions (.instructions.md)](#2-path-specific-instructions-instructionsmd)
3. [AGENTS.md — The Cross-Tool Alternative](#3-agentsmd--the-cross-tool-alternative)
4. [Prompt Files (.prompt.md)](#4-prompt-files-promptmd)
5. [Skills (Agent Skills)](#5-skills-agent-skills)
6. [Custom Agents (.agent.md)](#6-custom-agents-agentmd)
7. [Hooks](#7-hooks)
8. [Quick Decision Matrix](#8-quick-decision-matrix)
9. [Mental Model — 4 Layers](#9-mental-model--4-layers)
10. [Sample Project Structure](#10-sample-project-structure)

---

## 1. copilot-instructions.md

### What
A Markdown file that Copilot reads **automatically on every Chat and Agent Mode request** within that repository. It is the project-wide baseline — tech stack, coding conventions, architectural decisions, and team standards that every Copilot response should respect.

Stored at `.github/copilot-instructions.md`. This is the **primary** always-on instruction file for Copilot across all IDEs (VS Code, Visual Studio, JetBrains, Neovim, GitHub.com).

**≈ Claude Code:** Equivalent to `CLAUDE.md`. Same purpose, same always-on behavior. The key difference is the file location (`.github/` vs project root) and that Copilot has no equivalent of `CLAUDE.md`'s directory-walk hierarchy.

### Who configures it / who uses it
- **Configured by:** the developer, once. Commit to the repo to share with the team.  
- **Used by:** Copilot automatically, on every Chat request in the workspace.

### When it activates
Automatically — injected into every Chat and Agent Mode request. There is **no manual trigger** needed.

> **Note on PR Code Review:** Copilot Code Review uses `copilot-instructions.md` but through a slightly different execution path than Chat. If custom rules are not appearing in automated PR reviews, try reassigning Copilot as reviewer to force a reload. `AGENTS.md` is **not** yet supported by Copilot Code Review.

### Activation mechanism
**Automatic** — Copilot injects the file content into the system prompt before processing any request.

### Determinism
**Probabilistic.** Like all LLM-based instruction-following, Copilot may not apply every rule perfectly every time. Very long files result in some instructions being overlooked. GitHub recommends keeping the file concise and specific.

> **Rule of thumb:** If a rule must be followed 100% of the time, use a Hook instead.

### Use cases
- Project tech stack, language versions, framework choices
- Team-wide coding standards (naming conventions, error handling patterns)
- Banned libraries or deprecated APIs to avoid
- How to run builds, tests, linters
- Architectural decisions team members must follow

### Anti-patterns
- Rules that only apply to specific file types (use path-specific `.instructions.md` instead)
- Safety-critical enforcement rules (Copilot has no deterministic equivalent short of Hooks)
- Extremely long files — quality degrades as length increases

### Example

```markdown
<!-- .github/copilot-instructions.md -->
# MyProject — Copilot Instructions

## Tech Stack
- .NET 10 / ASP.NET Core — FastEndpoints, MassTransit, EF Core
- Frontend: React + TypeScript (Orval-generated clients, do not hand-write API calls)

## Rules
- Use `class` for FastEndpoints request DTOs — records cause source-gen fallback to reflection
- Add `[FromRoute]` attribute when Orval separates route params from body params
- Fee rounding: always use Largest Remainder Method (see docs/rounding.md)
- Never use `Parallel.ForEach` with scoped EF Core DbContext — use `Task.WhenAll` instead

## Build & Test
- Build: `dotnet build`
- Test: `dotnet test --collect:"XPlat Code Coverage"`
```

---

## 2. Path-Specific Instructions (.instructions.md)

### What
One or more Markdown files that apply only **when Copilot is working in the context of files matching a specific glob pattern** — defined via `applyTo` in YAML frontmatter. Stored in `.github/instructions/`. They **extend** (not replace) `copilot-instructions.md` — both are merged at runtime.

This solves a real problem: putting test-specific rules, frontend patterns, and backend conventions all inside one global file means Copilot reads all of them on every request, even when irrelevant. Path-specific files let you keep `copilot-instructions.md` lean and load domain rules only when the context matches.

An additional `excludeAgent` frontmatter property controls **which Copilot agent reads the file:**
- `excludeAgent: "code-review"` → only the coding agent reads it (not PR review)
- `excludeAgent: "cloud-agent"` → only code review reads it (not the coding agent)
- Neither set → both read it

> **Note:** Files with no `applyTo` property are **not applied automatically**. You must set `applyTo` for auto-loading to work.

### How it relates to Claude Code — a more nuanced comparison

This is where people often draw a false equivalence. The table below clarifies the actual comparison:

| | Copilot `.instructions.md` | Claude Code equivalent |
|---|---|---|
| **Core idea** | Scoped rules loaded when file context matches `applyTo` glob | Same idea, different mechanism |
| **Primary mechanism** | `applyTo` glob in frontmatter → Copilot merges matching files automatically | `.claude/rules/*.md` with `paths` frontmatter (added Dec 2025, Claude Code ≥ 2.0.64) |
| **Fallback mechanism** | Nested subdirectory organization in `.github/instructions/` | Nested `CLAUDE.md` files — Claude walks up the directory tree on navigation |
| **Agent-scoping** | `excludeAgent: "code-review"` or `"cloud-agent"` | No equivalent — Claude Code has no separate review agent |
| **Works in PR review** | ✅ Yes (since Sep 2025) | N/A |

**The real Claude Code analog** is `.claude/rules/*.md` files with a `paths` frontmatter field — not a Skill or nested `CLAUDE.md`. Both achieve the same goal: scoped instructions that only apply when working in a specific part of the codebase. The syntax differs but the intent is identical.

```markdown
<!-- Claude Code equivalent: .claude/rules/consumers.md -->
---
paths:
  - "src/**/Consumers/**"
---
Always perform idempotency check before processing a MassTransit consumer...
```

```markdown
<!-- Copilot equivalent: .github/instructions/consumers.instructions.md -->
---
applyTo: "src/**/Consumers/**"
---
Always perform idempotency check before processing a MassTransit consumer...
```

### Who configures it / who uses it
- **Configured by:** the developer.  
- **Used by:** Copilot automatically, when the active context includes files matching `applyTo`.

### When it activates
When Copilot processes a request in the context of files matching the `applyTo` glob. Both the global `copilot-instructions.md` and all matching path-specific files are merged and injected together.

### Activation mechanism
**Automatic — path-context driven.**

### Determinism
**Probabilistic.** Same as `copilot-instructions.md`. Merging multiple instruction sources can produce conflicts — avoid contradictions between files.

### Frontmatter fields

| Field | Required | Description |
|---|---|---|
| `applyTo` | **Yes** (for auto-loading) | Glob pattern. No `applyTo` = file is ignored for automatic injection. |
| `excludeAgent` | No | `"code-review"` or `"cloud-agent"` — prevents one agent from reading this file |

### Use cases
- Test conventions scoped to test files: `applyTo: "**/*.test.*"`
- Frontend patterns scoped to frontend directory: `applyTo: "src/frontend/**"`
- Consumer scaffolding rules for coding agent only (not PR review): `excludeAgent: "code-review"`
- Large monorepos where one global file would become unmaintainable

### Anti-patterns
- Duplicating content from `copilot-instructions.md` (drift, contradictions)
- `applyTo: "**"` — this is just a slower `copilot-instructions.md`
- Rules that need 100% enforcement (use Hooks — instructions are probabilistic regardless of scope)

### Example

```markdown
<!-- .github/instructions/tests.instructions.md -->
---
applyTo: "**/*.test.ts"
---

# Test File Rules

- Framework: xUnit. Use `[Fact]` for single cases, `[Theory]` + `[InlineData]` for parameterized.
- Never access `DbContext` directly — use `BaseTestFixture` with `DependencyBuilder.Reset()`.
- Integration test classes must inherit `BaseTestFixture`.
- Test method naming: `MethodName_Scenario_ExpectedResult`.
```

```markdown
<!-- .github/instructions/consumers.instructions.md -->
---
applyTo: "src/**/Consumers/**"
excludeAgent: "code-review"
---

# MassTransit Consumer Rules (coding agent only, excluded from PR review)

- Always check idempotency via distributed cache before processing any message.
  Pattern: `if (await _cache.GetAsync(context.Message.MessageId.ToString()) != null) return;`
- Consumers must implement `IConsumer<T>` — never call service methods directly.
- Register in `DependencyBuilder` under the correct bus endpoint.
```

---

## 3. AGENTS.md — The Cross-Tool Alternative

### What
`AGENTS.md` is an **open standard** (published December 2025) for repository-wide AI agent instructions, designed to be portable across tools: GitHub Copilot, Claude Code, OpenAI Codex CLI, Cursor, Gemini CLI, and others.

Placed at the **repository root**, it functions identically to `.github/copilot-instructions.md` for Copilot Chat and Coding Agent (both files are used if both exist). Nested `AGENTS.md` files in subdirectories are also supported and apply to their subtree.

> **Important scope note:** As of April 2026, `AGENTS.md` is **not** supported by Copilot Code Review — use `copilot-instructions.md` for that. For Copilot Chat and Coding Agent, it is fully supported.

**≈ Claude Code:** Equivalent to `CLAUDE.md` — Claude Code also reads `AGENTS.md` from the repo root.

### When to use AGENTS.md vs copilot-instructions.md
| Scenario | Recommendation |
|---|---|
| Team uses only GitHub Copilot | Use `.github/copilot-instructions.md` |
| Team uses both Copilot and Claude Code | Use `AGENTS.md` as single source of truth |
| Need Copilot Code Review support | Also keep `.github/copilot-instructions.md` |
| Multi-tool org wanting one file | `AGENTS.md` — avoid duplicating rules into both |

---

## 4. Prompt Files (.prompt.md)

### What
Reusable prompt templates stored in `.github/prompts/` that a developer **manually attaches** to a Chat session. Unlike instruction files (always-on), prompt files are on-demand — they appear in the Chat Customizations editor and can be attached to specific requests.

Prompt files support variable placeholders. They can reference instruction files and be scoped to a specific agent using the `mode` frontmatter field.

**≈ Claude Code:** Equivalent to Commands (`.claude/commands/*.md`). Same concept: reusable, manually triggered prompt templates. Copilot calls them "prompt files"; Claude Code calls them "commands" (or skills without supporting resources).

### Who configures it / who uses it
- **Configured by:** developer.  
- **Used by:** developer — explicitly attached in the Chat panel or referenced in a prompt.

### When it activates
Only when the developer explicitly attaches or references it in a chat. Never auto-triggers.

### Activation mechanism
**Manual — developer attaches to chat session.**

### Determinism
**Deterministic trigger; probabilistic execution.** Attaching the file always includes that prompt; Copilot's interpretation is still LLM-based.

### Use cases
- PR description template — attach when writing PR summaries
- Code review checklist — attach before asking Copilot to review
- Documentation generation — attach when generating API docs
- Refactoring guidance — attach before large refactors

### Anti-patterns
- Instructions you want active on every request (use `copilot-instructions.md`)
- Multi-step workflows with supporting templates/scripts (use Skills)
- Rules that need enforcement (Copilot has no blocking equivalent short of Hooks)

### Example

```markdown
<!-- .github/prompts/review-pr.prompt.md -->
---
mode: 'agent'
---

Review the current diff for:
- Bugs and logic errors
- Missing idempotency checks on MassTransit consumers
- `[FromRoute]` usage when route params are present
- FastEndpoints DTO type (must be `class`, not `record`)
- Security issues: SQL injection, unvalidated input, exposed secrets

Output one finding per line: `<file>:<line> — <severity> — <description>`.
```

---

## 5. Skills (Agent Skills)

### What
A **Skill** is a directory containing a `SKILL.md` file with YAML frontmatter, plus optional supporting scripts, templates, and reference files. Skills give Copilot (and other AI tools) domain-specific knowledge and procedures that load **on demand** — without polluting the always-on context.

**Progressive disclosure:** At startup, Copilot reads only each skill's `name` and `description` (~100 tokens per skill). When Copilot determines the skill is relevant, it reads the full `SKILL.md` body. Supporting files are loaded only when the instructions reference them.

**Open standard:** The `SKILL.md` format is an open standard (December 2025). Skills are **portable** — the same skill folder works in Copilot, Claude Code, Cursor, Codex CLI, and Gemini CLI, with only the install directory path changing.

**≈ Claude Code:** Identical concept, identical file format. The install paths differ:
- Copilot (VS Code): `.github/skills/<skill-name>/` (workspace) or `~/.copilot/skills/` (personal)
- Claude Code: `.claude/skills/<skill-name>/` (project) or `~/.claude/skills/` (personal)

### Frontmatter fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Becomes the `/skill-name` slash command. Lowercase, hyphens only. |
| `description` | Recommended | How Copilot discovers the skill. Write in third person. Include *what* it does AND *when* to use it. |
| `allowed-tools` | No | Restrict which tools can be used (e.g. `Read Grep` for read-only skills). |
| `disable-model-invocation` | No | `true` = only user can trigger via `/name`. Copilot will not auto-invoke. |

### Who configures it / who uses it
- **Configured by:** developer or installed from a community/org library.  
- **Used by:** Copilot automatically when task matches description, **or** manually via `/skill-name`.

### When it activates
- **Auto:** Copilot detects the current task matches the skill's description.
- **Manual:** developer types `/skill-name [arguments]` in Chat.

### Activation mechanism
**Semi-automatic (probabilistic auto-detection) or manual slash command.**

### Determinism
**Probabilistic on auto-detection.** Manual `/name` is deterministic at trigger; execution is still LLM-based.

### Skill directory structure

```
.github/skills/incident-triage/
├── SKILL.md              # Core instructions + YAML frontmatter
├── templates/
│   └── postmortem.md     # Loaded on demand when SKILL.md references it
├── references/
│   └── runbook.md        # Loaded on demand
└── scripts/
    └── collect-logs.sh   # Executed by Copilot; only stdout enters context
```

### Use cases
- Incident triage workflow with postmortem template
- Release checklist with bundled scripts
- Domain-specific patterns (e.g. EF Core migration conventions)
- Documentation generation with org-specific formatting standards

### Anti-patterns
- One-off prompts with no supporting resources (use a prompt file instead)
- Rules that must always apply regardless of task (put in `copilot-instructions.md`)
- Safety enforcement (use Hooks for guaranteed execution)

### Example

```markdown
<!-- .github/skills/incident-triage/SKILL.md -->
---
name: incident-triage
description: Guides through an incident triage workflow including impact assessment,
  timeline capture, and owner assignment. Use when the user mentions an incident,
  outage, production issue, or asks to triage a problem.
---

## Incident Triage Workflow

1. **Identify severity** — ask the user: what is affected? how many users? revenue impact?
2. **Load postmortem template** — open `templates/postmortem.md`.
3. **Fill in the known fields**: incident title, start time, detection method, initial scope.
4. **Identify the on-call owner** — check `references/runbook.md` for the on-call rotation.
5. **Suggest immediate mitigations** based on the symptoms described.
6. **Leave action items** section blank — to be filled post-resolution.

Output the partially completed postmortem as a new file at `docs/incidents/YYYY-MM-DD-<title>.md`.
```

---

## 6. Custom Agents (.agent.md)

### What
A **Custom Agent** is a Markdown file with YAML frontmatter that defines a **specialized AI persona** — with its own system prompt, tool access, model preference, and optional handoffs to other agents. Stored in `.github/agents/` for the repo, or `~/.copilot/agents/` for personal use.

When a developer selects an agent in the Chat panel (or assigns it to a GitHub issue), that agent's configuration replaces Copilot's default persona. The agent cannot use tools outside its declared `tools` list.

Unique feature not present in Claude Code: **handoffs** — buttons that appear after an agent finishes, allowing the developer to transition to the next agent in a workflow with pre-filled context. This enables guided multi-step workflows (Plan → Implement → Review).

**≈ Claude Code:** Equivalent to Claude Code's Agent/Subagent files (`.claude/agents/`). The core concept is identical — specialized isolated context, scoped tools, defined system prompt. Key difference: Copilot agents use **handoffs** for sequential workflows (human reviews each step), while Claude Code agents are spawned automatically by the main Claude session for parallel/background delegation.

### Frontmatter fields

| Field | Description |
|---|---|
| `name` | Display name. If omitted, filename is used. |
| `description` | What the agent does. Shown in the agent picker. **Critical for auto-invocation.** |
| `tools` | List of allowed tools. Omit to allow all. Examples: `['read', 'edit', 'search', 'web/fetch']` |
| `model` | Model to use (e.g. `claude-opus-4-6`, `GPT-5.2 (copilot)`). Can be a list tried in order. |
| `handoffs` | List of transitions to other agents shown as buttons after a response. |
| `mcp-servers` | MCP servers available to this agent. |

### Handoffs structure

```yaml
handoffs:
  - label: "Start Implementation"   # Button label shown to user
    agent: implementation            # Target agent filename (without .agent.md)
    prompt: "Implement the plan."    # Pre-filled prompt in target agent
    send: false                      # false = user reviews; true = auto-submits
```

### Who configures it / who uses it
- **Configured by:** developer.  
- **Used by:** developer — selects from Chat panel dropdown or assigns to a GitHub issue.

### When it activates
When the developer explicitly selects the agent from the Chat panel, uses `@agent-name` syntax, or assigns Copilot Coding Agent to an issue with a custom agent configured.

### Activation mechanism
**Manual selection** (VS Code Chat, GitHub.com issue assignment). Copilot may also suggest relevant agents based on context.

### Determinism
**Structurally scoped** — the agent can only use its declared tools, preserving safety boundaries. Internal behavior is still LLM-based.

### Claude Code vs Copilot agents — key difference

| | Claude Code Agent | Copilot Custom Agent |
|---|---|---|
| Spawned by | Main Claude session automatically | Developer manual selection |
| Context | Isolated — main session stays clean | Replaces default Copilot persona in the session |
| Multi-step | Main Claude orchestrates in background | Handoffs — developer approves each step |
| Scope | Background task delegation | Persistent persona for a session |

### Use cases
- Security reviewer agent: read-only tools, never modifies production code
- Planner agent → handoff → implementation agent → handoff → code review agent
- .NET migration specialist with deep domain knowledge encoded in system prompt
- Test coverage agent focused solely on gap analysis and test generation

### Anti-patterns
- Tasks that don't need a specialized persona (use prompt files or instructions instead)
- Encoding rules that should apply to every request (use `copilot-instructions.md`)
- Building an agent for a one-off task (prompt file is simpler)

### Example

```markdown
<!-- .github/agents/security-reviewer.agent.md -->
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Focuses on authentication,
  authorization, injection, and data handling. Read-only — never modifies code.
tools: ['read', 'search/codebase', 'search/usages']
model: claude-opus-4-6
handoffs:
  - label: "Fix Findings"
    agent: implementation
    prompt: "Fix the security findings identified in the review above."
    send: false
---

You are a senior application security engineer. Your role is to review code only —
never edit or create files.

Review for:
- SQL injection, command injection, XSS
- Authentication and authorization flaws
- Hardcoded secrets or credentials
- Insecure deserialization or data handling

Output: one finding per line — file path, line number, severity (High/Medium/Low),
and a one-sentence fix recommendation.
```

---

## 7. Hooks

### What
Hooks are **shell commands that execute automatically at specific lifecycle events** during Copilot Coding Agent and Copilot CLI sessions. They run outside the LLM — no AI judgment involved. This makes them the only Copilot mechanism with guaranteed execution.

Stored as JSON files in `.github/hooks/` (one or more files, any name). The hook config must be on the **default branch** to be used by the cloud agent.

**≈ Claude Code:** Equivalent concept to Claude Code's Hooks system. The key differences:
- Copilot hooks: configured in `.github/hooks/*.json` (JSON format, `bash`/`powershell` keys)
- Claude Code hooks: configured in `.claude/settings.json` (JSON, single `command` key)
- Copilot has **6 events**; Claude Code has **20+ events** (more granular)
- Only `preToolUse` can block in Copilot; same in Claude Code (`PreToolUse` + exit 2)
- Copilot hooks support both `bash` and `powershell` in the same entry (cross-platform)
- VS Code's hook format (Preview) uses PascalCase event names and maps automatically from Copilot CLI's camelCase format

### Lifecycle events

| Event | When it fires | Can block? |
|---|---|---|
| `sessionStart` | Agent session begins or resumes | No |
| `sessionEnd` | Session completes or is terminated | No |
| `userPromptSubmitted` | User submits a prompt, before Copilot processes it | No |
| `preToolUse` | **Before** the agent uses any tool (bash, edit, view...) | **Yes** |
| `postToolUse` | After a tool completes successfully | No |
| `errorOccurred` | An error happens during agent execution | No |

> **Only `preToolUse` can block.** All other events are for side effects: logging, notifications, audit trails.

### Blocking with preToolUse

Return a JSON `permissionDecision` from the script to approve or deny:

```json
{ "permissionDecision": "deny", "permissionDecisionReason": "Reason shown to agent" }
```

Return nothing (or exit 0) to allow.

### Hook entry fields

| Field | Description |
|---|---|
| `type` | Always `"command"` |
| `bash` | Shell command (Linux/macOS) |
| `powershell` | PowerShell command (Windows) |
| `cwd` | Working directory for the command |
| `timeoutSec` | Seconds before the hook times out |
| `env` | Environment variables to pass to the script |

### Who configures it / who uses it
- **Configured by:** developer (or security/platform team for org-wide enforcement).  
- **Executed by:** Copilot runtime — automatically, no human trigger.

### When it activates
At the corresponding lifecycle event. There is no matcher/regex like in Claude Code — all hooks for an event run for every occurrence.

### Activation mechanism
**Fully automatic and deterministic.**

### Determinism
**100% deterministic.** Fires on every matching event regardless of the LLM's context or focus.

### Use cases
- Block dangerous shell commands (privilege escalation, destructive operations)
- Audit log every tool use for compliance
- Auto-run linters after file edits (postToolUse)
- Inject session context on startup (sessionStart)
- Validate that no secrets were leaked in modified files (postToolUse + secret scanning)

### Anti-patterns
- Adding too many slow hooks — they pause the agent loop sequentially
- Blocking on `postToolUse` — the action has already happened; you cannot undo it
- Using hooks for soft guidelines that don't need 100% enforcement (use instructions instead)

### Example

```json
// .github/hooks/policy.json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "bash": "echo \"Session started: $(date) | Branch: $(git branch --show-current)\" >> .github/hooks/logs/session.log",
        "cwd": ".",
        "timeoutSec": 10
      }
    ],
    "preToolUse": [
      {
        "type": "command",
        "bash": "./.github/hooks/scripts/security-check.sh",
        "powershell": "./.github/hooks/scripts/security-check.ps1",
        "cwd": ".",
        "timeoutSec": 15
      }
    ],
    "postToolUse": [
      {
        "type": "command",
        "bash": "INPUT=$(cat); TOOL=$(echo \"$INPUT\" | jq -r '.toolName'); echo \"{\\\"tool\\\":\\\"$TOOL\\\"}\" >> .github/hooks/logs/audit.jsonl",
        "cwd": "."
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .github/hooks/scripts/security-check.sh
# Block privilege escalation and destructive patterns

INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.toolName')
TOOL_ARGS=$(echo "$INPUT" | jq -r '.toolArgs // ""')

if [ "$TOOL_NAME" = "bash" ]; then
  if echo "$TOOL_ARGS" | grep -qE 'sudo|rm -rf|DROP TABLE|chmod 777'; then
    echo '{"permissionDecision":"deny","permissionDecisionReason":"Blocked by security policy"}' 
    exit 0
  fi
fi
# Allow everything else
exit 0
```

---

## 8. Quick Decision Matrix

| Need | Use | Why |
|---|---|---|
| Copilot always knows our tech stack and conventions | `copilot-instructions.md` | Always-on for all requests |
| Rules only for test files (`*.test.ts`) | `.instructions.md` with `applyTo` | Scoped — no noise for other files |
| Same instructions for both Copilot and Claude Code | `AGENTS.md` at repo root | Cross-tool open standard |
| Prompt I attach before a code review | Prompt file (`.github/prompts/`) | Manual, session-specific |
| Reusable workflow with templates/scripts | Skill (`.github/skills/`) | Progressive disclosure, supporting files |
| Read-only PR review persona | Custom agent (`.github/agents/`) | Scoped tools, cannot edit files |
| Guided Plan → Implement → Review workflow | Custom agents with handoffs | Sequential, human-approved transitions |
| Block dangerous commands 100% of the time | Hook `preToolUse` | Only truly deterministic option |
| Audit every tool use for compliance | Hook `postToolUse` | Fires unconditionally, append to log |
| Rules that need AI judgment (soft guidelines) | `copilot-instructions.md` rules | Advisory is fine here |

---

## 9. Mental Model — 4 Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1 — Memory (always-on context)                           │
│  copilot-instructions.md / AGENTS.md / .instructions.md        │
│  → "Employee handbook." Active on every request. Probabilistic. │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Knowledge Packages (on-demand)                       │
│  Skills + Prompt files                                          │
│  → "Saved playbooks." Skills auto-detect; prompts are manual.  │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Specialist Personas (session-scoped)                 │
│  Custom Agents (.agent.md) + Handoffs                           │
│  → "Specialist contractors." Fixed tools, guided workflow.     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4 — Enforcement (deterministic gates)                    │
│  Hooks (.github/hooks/*.json)                                   │
│  → "CI/CD gates." No pass = no proceed. No exceptions.         │
└─────────────────────────────────────────────────────────────────┘
```

**The golden rule:**  
Use `copilot-instructions.md` for what Copilot *should* do. Use Hooks for what Copilot *must* do.

---

## 10. Sample Project Structure

```
my-project/
│
├── AGENTS.md                              # Optional: cross-tool instructions (Copilot + Claude Code)
│                                          # If present, used alongside copilot-instructions.md
│
├── .github/
│   │
│   ├── copilot-instructions.md            # Always-on repo-wide instructions
│   │                                      # Tech stack, conventions, build commands
│   │
│   ├── instructions/
│   │   ├── tests.instructions.md          # applyTo: "**/*.test.*"
│   │   ├── consumers.instructions.md      # applyTo: "src/**/Consumers/**"
│   │   │                                  # excludeAgent: "code-review" (coding agent only)
│   │   └── frontend.instructions.md       # applyTo: "src/frontend/**"
│   │
│   ├── prompts/
│   │   ├── review-pr.prompt.md            # Manually attached before asking for a PR review
│   │   └── generate-docs.prompt.md        # Attached when generating API documentation
│   │
│   ├── agents/
│   │   ├── security-reviewer.agent.md     # Read-only; reviews auth/payment code for issues
│   │   ├── planner.agent.md               # Planning only; handoff → implementation agent
│   │   └── test-specialist.agent.md       # Generates tests; never modifies production code
│   │
│   ├── skills/
│   │   ├── incident-triage/
│   │   │   ├── SKILL.md                   # Incident response workflow
│   │   │   └── templates/
│   │   │       └── postmortem.md          # Loaded on demand during triage
│   │   │
│   │   └── create-consumer/
│   │       ├── SKILL.md                   # MassTransit consumer scaffold
│   │       └── templates/
│   │           └── consumer.cs.template
│   │
│   └── hooks/
│       ├── policy.json                    # preToolUse security rules, postToolUse audit log
│       └── scripts/
│           ├── security-check.sh          # Block dangerous patterns (Linux/macOS)
│           └── security-check.ps1         # Block dangerous patterns (Windows)
│
└── .gitignore
```

### Key placement rules

| File / Folder | Scope | Committed to repo? |
|---|---|---|
| `.github/copilot-instructions.md` | This repository | ✅ Yes — team-shared |
| `AGENTS.md` (root) | This repository, cross-tool | ✅ Yes — team-shared |
| `.github/instructions/` | This repository | ✅ Yes |
| `.github/prompts/` | This repository | ✅ Yes |
| `.github/agents/` | This repository | ✅ Yes |
| `.github/skills/` | This repository | ✅ Yes |
| `.github/hooks/` | This repository (must be on default branch) | ✅ Yes |
| `~/.copilot/instructions/` | All workspaces (personal) | ❌ No — personal preferences |
| `~/.copilot/agents/` | All workspaces (personal) | ❌ No |
| `~/.copilot/skills/` | All workspaces (personal) | ❌ No |

---

*Sources: GitHub Copilot official docs (docs.github.com), VS Code Copilot customization docs (code.visualstudio.com), GitHub Changelog, GitHub Blog — verified April 2026.*
