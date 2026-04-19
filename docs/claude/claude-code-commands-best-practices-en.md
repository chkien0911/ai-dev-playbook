# Claude Code Commands — Best Practices

> **Sources:** Anthropic official documentation ([code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands)), [code.claude.com/docs/en/features-overview](https://code.claude.com/docs/en/features-overview), [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills), and authoritative community references (alexop.dev, MindStudio, Daniel Miessler).

---

## What Are Commands?

Commands control Claude Code from inside a session. They come in two forms:

**Built-in commands** — coded into the CLI. Examples: `/clear`, `/compact`, `/memory`, `/plan`, `/init`. Their behavior is fixed.

**Custom commands (skills)** — markdown files you write yourself. Stored in `.claude/commands/` or `.claude/skills/`, they create `/command-name` shortcuts for repeatable workflows. Since October 2025, custom commands have been **merged into the Skills system** — both mechanisms are identical under the hood.

> **Official Anthropic guidance:** "Custom commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Skills are recommended going forward because they support additional features."

Type `/` in any Claude Code session to see all available commands, or type `/` followed by letters to filter.

---

## Built-in Commands Reference

The most important built-in commands organized by category.

### Context & Session Management

| Command | Purpose |
|---|---|
| `/clear` | Clear conversation history and free context. Aliases: `/reset`, `/new` |
| `/compact [instructions]` | Summarize the conversation to reduce context usage. Pass focus instructions to retain specific information |
| `/context` | Visualize context usage as a colored grid. Shows optimization suggestions and capacity warnings |
| `/branch [name]` | Fork the current conversation at this point, preserving the original. Alias: `/fork` |
| `/resume [session]` | Resume a previous conversation by ID or name |
| `/rewind` | Rewind conversation and/or code to a previous checkpoint. Alias: `/checkpoint` |
| `/btw <question>` | Ask a quick side question without adding it to the conversation history |

### Planning & Workflow

| Command | Purpose |
|---|---|
| `/init` | Generate a starter `CLAUDE.md` from codebase analysis |
| `/plan [description]` | Enter plan mode. Pass a description to start planning immediately |
| `/batch <instruction>` | **[Skill]** Orchestrate large-scale parallel changes. Decomposes work into 5–30 units, spawns one background agent per unit in isolated git worktrees |
| `/simplify [focus]` | **[Skill]** Review recently changed files for code quality issues. Spawns three review agents in parallel, aggregates findings, applies fixes |
| `/debug [description]` | **[Skill]** Enable debug logging and troubleshoot issues by reading the session debug log |

### Inspection & Diagnostics

| Command | Purpose |
|---|---|
| `/memory` | Edit `CLAUDE.md` files, manage auto-memory, view what loaded |
| `/hooks` | View hook configurations for tool events |
| `/skills` | List available skills from project, user, and plugin sources |
| `/agents` | Manage subagent configurations |
| `/permissions` | Manage allow, ask, and deny rules for tool permissions |
| `/doctor` | Diagnose Claude Code installation and settings issues |
| `/cost` | Show token usage statistics for the current session |
| `/diff` | Open interactive diff viewer showing uncommitted changes and per-turn diffs |

### Automation & Scheduling

| Command | Purpose |
|---|---|
| `/loop [interval] [prompt]` | **[Skill]** Run a prompt repeatedly while the session stays open |
| `/schedule [description]` | Create, update, list, or run scheduled routines |
| `/autofix-pr [prompt]` | Spawn a web session that watches a PR and pushes fixes when CI fails or reviewers leave comments |

### Utility

| Command | Purpose |
|---|---|
| `/copy [N]` | Copy the last assistant response to clipboard. Pass `N` to copy the Nth-latest |
| `/export [filename]` | Export the current conversation as plain text |
| `/model [model]` | Select or change the AI model mid-session |
| `/effort [level]` | Set model effort: `low`, `medium`, `high`, `max`, or `auto` |
| `/security-review` | Analyze pending changes for security vulnerabilities |
| `/insights` | Generate a report analyzing Claude Code session patterns and friction points |

---

## Bundled Skills (Invoked as Commands)

Some entries in the command list are **bundled skills** — prompts handed to Claude, not hardcoded CLI behavior. They use the same mechanism as skills you write yourself, and Claude can also invoke them automatically when relevant.

| Command | What it does |
|---|---|
| `/batch` | Parallel large-scale refactors with isolated git worktrees per unit |
| `/simplify` | Three parallel review agents aggregate findings and apply fixes |
| `/debug` | Session debug log analysis |
| `/loop` | Recurring prompt execution, self-paced between iterations |
| `/claude-api` | Load Claude API reference for the current project's language |

---

## Custom Commands (Writing Your Own)

Custom commands are markdown files that create `/command-name` shortcuts for your own workflows.

### Two Storage Locations

```
# Single-file command (legacy format, still works)
.claude/commands/deploy.md        →  invoked with /deploy

# Directory-based skill (recommended, more features)
.claude/skills/deploy/SKILL.md   →  invoked with /deploy
```

Both create the same `/deploy` command. Use the directory form when you need supporting files (scripts, templates, examples).

### Basic Custom Command Structure

```markdown
---
description: Deploy the current branch to staging
allowed-tools: Bash, Read
disable-model-invocation: true
---

Deploy to staging:
1. Run tests — `dotnet test`
2. Build Docker image — `docker build -t app:staging .`
3. Push and deploy — `./scripts/deploy.sh staging`
4. Verify health endpoint — `./scripts/health-check.sh staging`
```

### Using `$ARGUMENTS`

Commands can accept arguments passed after the command name. The `$ARGUMENTS` placeholder is replaced with whatever text the user types after the command.

```markdown
---
description: Look up documentation for a library or API
allowed-tools: WebFetch, Read
---

Fetch current documentation for: $ARGUMENTS

1. Check if there is an llms.txt at the root URL
2. Fetch relevant documentation pages
3. Answer the question using the fetched documentation, not training data
```

Invoke with: `/lookup-docs Dexie.js liveQuery usage`

### Orchestrating Subagents from a Command

Commands can explicitly spawn subagents in parallel by including Task tool instructions in their body:

```markdown
---
description: Research a technical problem using parallel agents
allowed-tools: Task, WebSearch, WebFetch, Grep, Glob, Read, Write
---

Research the following: $ARGUMENTS

Launch these subagents IN PARALLEL using the Task tool:

1. **Web Agent** — search official documentation and GitHub issues
2. **Codebase Agent** — search for existing patterns in this codebase
3. **Stack Overflow Agent** — find community solutions and workarounds

Synthesize findings into `docs/research/$ARGUMENTS.md`.
```

---

## Commands vs Skills vs CLAUDE.md — When to Use Each

This is the most important decision to understand clearly.

| Feature | Loads | Invoked by | Best for |
|---|---|---|---|
| **CLAUDE.md** | Every session, always | Automatic | "Always do X" facts, conventions, project structure |
| **Built-in command** | On demand | You type `/command` | Session control: clear, compact, plan, review |
| **Custom command / skill** | On demand | You type `/command` | Repeatable workflows you trigger manually |
| **Skill (auto-invoke)** | On demand | Claude detects relevance | Reference knowledge, conventions Claude applies automatically |
| **Subagent** | On demand | Claude or explicit delegation | Isolated heavy tasks, parallel work, verbose output |

### The Core Distinction: Commands vs Skills

Both are markdown files. The difference is **how they are invoked and whether Claude can auto-invoke them**.

| | Custom Command | Skill (auto-invocable) |
|---|---|---|
| **Invocation** | Explicit — you type `/name` | Explicit (`/name`) **or** automatic by Claude |
| **Supporting files** | Single `.md` file only | Directory with scripts, templates, examples |
| **Auto-invocation** | Not by default | Yes, based on description matching |
| **Best for** | Deterministic workflows you control | Reference knowledge + workflows |

> **Rule from official docs:** "Save it as a user-invocable skill when you keep typing the same prompt to start a task. Use CLAUDE.md when Claude gets a convention wrong twice."

### Practical Trigger Guide

| You notice this... | Add this |
|---|---|
| Claude gets a convention wrong twice | Add to `CLAUDE.md` |
| You type the same prompt to start every task | Save as a custom command / skill |
| You paste the same multi-step procedure into chat | Capture as a skill |
| A side task floods conversation with verbose output | Route to a subagent |
| You want something to happen every time without asking | Write a hook |

---

## ✅ SHOULD — Design Guidelines for Custom Commands

### 1. Use `disable-model-invocation: true` for action commands

Any command that performs an irreversible operation (deploy, migrate, release, format) must require explicit invocation. Never allow Claude to auto-trigger it based on keyword matching.

```yaml
---
name: run-migrations
description: Run pending EF Core database migrations
disable-model-invocation: true
allowed-tools: Bash
---

Run EF Core migrations:
1. `dotnet ef migrations list` — confirm pending migrations
2. `dotnet ef database update` — apply migrations
3. Verify: `dotnet ef migrations list` should show all as applied
```

### 2. Scope `allowed-tools` to the minimum required

Just like subagents, commands should only have the tools they need.

```yaml
# ✅ Read-only research command
allowed-tools: Read, Grep, Glob, WebFetch

# ✅ Deployment command needs Bash
allowed-tools: Bash, Read

# ❌ No restriction — inherits everything including MCP servers
# (omitting allowed-tools entirely)
```

### 3. Use `$ARGUMENTS` for parameterizable commands

Make commands flexible by accepting arguments rather than hardcoding everything. This avoids creating multiple near-identical commands for similar tasks.

```yaml
---
description: Run tests for a specific module or path
allowed-tools: Bash
---

Run tests for: $ARGUMENTS

Execute: `dotnet test --filter "$ARGUMENTS" --collect:"XPlat Code Coverage"`
Report any failures with their full output.
```

### 4. Write descriptions as trigger conditions

The `description` field is used both for `/` autocomplete display and for Claude's auto-invocation decision. Write it as a trigger condition, not just a label.

```yaml
# ✅ Clear trigger condition
description: Generate a changelog entry for the current branch changes. Use after completing a feature or fix.

# ❌ Just a label
description: Changelog command
```

### 5. Commit project commands to version control

Project-level commands (`.claude/commands/` or `.claude/skills/`) should be committed so the whole team shares the same workflows. Treat them as code.

### 6. Use the directory form for complex commands

When a command needs reference files, templates, or scripts, use the skill directory structure instead of a single `.md` file:

```
.claude/skills/
└── generate-pr-description/
    ├── SKILL.md          # Main instructions
    ├── template.md       # PR description template
    └── examples/
        └── sample-pr.md  # Example output
```

---

## ❌ SHOULD NOT — Common Mistakes

### 1. Do not use commands for always-applicable conventions

If Claude should always know a rule — "never use `dynamic`", "always run tests before committing" — put it in `CLAUDE.md`. Commands only execute when invoked. Rules that belong in CLAUDE.md will not apply unless the command is explicitly run.

### 2. Do not omit `disable-model-invocation` for action commands

A deploy or migration command that Claude auto-triggers based on a keyword in a prompt is dangerous. Always add `disable-model-invocation: true` for anything irreversible.

### 3. Do not build commands for things hooks handle better

If you want something to happen automatically on a specific event (after every file edit, before a commit), use a hook instead of a command. Commands require you to type `/command-name`. Hooks fire deterministically.

```
# You want ESLint to run after every file write
# ❌ Wrong: /lint command you have to remember to run
# ✅ Right: PostToolUse hook on Write/Edit tools
```

### 4. Do not create many near-identical commands

If you have `/test-api`, `/test-db`, `/test-auth` that all do the same thing with different arguments, consolidate into one parameterized command: `/test $ARGUMENTS`.

### 5. Do not confuse `/btw` with a custom command

`/btw` is a built-in for quick side questions that do not get added to conversation history. It is not a custom command trigger. Use it when you want to ask something without polluting the session context.

---

## Key Built-in Commands for Daily Use

### Context Management Workflow

```
# Start of long session
/init               # Generate CLAUDE.md if missing
/plan fix auth bug  # Enter plan mode before coding

# Mid-session when context is getting full
/context            # Check usage
/compact            # Summarize, keep going

# When starting fresh
/clear              # Full reset
```

### Inspection Before Acting

```
/memory             # See what CLAUDE.md content loaded
/context            # See token breakdown
/diff               # Review uncommitted changes before committing
/security-review    # Check for vulnerabilities in current diff
```

### Session Branching for Exploration

```
/branch experiment-approach-a   # Fork current session
# ... try one approach ...
/resume             # Pick the original session
/branch experiment-approach-b   # Try another approach
```

---

## MCP Prompts as Commands

MCP servers can expose prompts that appear as commands in the format `/mcp__<server>__<prompt>`. These are dynamically discovered from connected servers and behave like built-in commands.

For example, if a GitHub MCP server exposes a `create-pr` prompt, it appears as `/mcp__github__create-pr` in the command list.

---

## Pre-Creation Checklist for Custom Commands

```
□ Is this a repeatable workflow you trigger manually? → Custom command / skill
□ Is this something Claude should always know? → CLAUDE.md instead
□ Is this something that should happen automatically on an event? → Hook instead
□ Does the command perform irreversible operations? → Add disable-model-invocation: true
□ Are tools restricted to the minimum needed?
□ Can it be parameterized with $ARGUMENTS to replace multiple similar commands?
□ Does the description serve as a clear trigger condition?
□ Is it committed to version control for the team?
```

---

## References

- [Anthropic — Commands reference](https://code.claude.com/docs/en/commands)
- [Anthropic — Extend Claude Code (feature comparison)](https://code.claude.com/docs/en/features-overview)
- [Anthropic — Skills documentation](https://code.claude.com/docs/en/skills)
- [alexop.dev — Claude Code Customization: CLAUDE.md, Slash Commands, Skills, and Subagents](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/)
- [MindStudio — Claude Skills vs Slash Commands: When to Use Each](https://www.mindstudio.ai/blog/claude-skills-vs-slash-commands)
- [Daniel Miessler — When to Use Skills vs Workflows vs Agents](https://danielmiessler.com/blog/when-to-use-skills-vs-commands-vs-agents)
