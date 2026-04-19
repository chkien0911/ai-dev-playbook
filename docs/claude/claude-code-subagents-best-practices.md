# Claude Code Subagents — Best Practices

> **Source:** Anthropic official documentation ([code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)) and authoritative community references (PubNub Engineering, Skywork AI).

---

## What Is a Subagent?

Each subagent runs in its own isolated context window with a custom system prompt, specific tool access, and independent permissions. When Claude encounters a task that matches a subagent's description, it delegates to that subagent, which works independently and returns only the result — keeping the main conversation context clean.

> **Rule of thumb (Anthropic):** Use a subagent when a side task would flood your main conversation with search results, logs, or file contents you won't need to reference again.

---

## Core Principles

### Single Responsibility

Each subagent must have **exactly one job** with clear inputs, outputs, and a defined handoff rule. Examples of well-scoped jobs:

- "Write unit tests for a given module"
- "Review code for security vulnerabilities"
- "Validate that a database query is read-only"

A "Backend Agent" or "Frontend Agent" is **not** a job — it is a domain. Agents scoped by layer (backend/frontend/API) violate single responsibility and degrade into general-purpose prompting, defeating the purpose of specialization.

### Context Isolation

Subagents exist to protect the main conversation context. Running tests, fetching documentation, or processing logs can consume significant tokens. By delegating these to a subagent, verbose output stays in the subagent's context while only the relevant summary is returned.

**Anthropic heuristic:** If a task requires exploring **10 or more files**, or involves **3 or more independent pieces of work**, subagents are worth the overhead. Below that threshold, handle it in the main conversation.

---

## Configuration Reference

Subagents are defined as Markdown files with YAML frontmatter, stored in `.claude/agents/` (project-level) or `~/.claude/agents/` (user-level).

```yaml
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality,
  security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: sonnet
memory: project
---

You are a senior code reviewer. When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Provide feedback organized by priority: Critical / Warnings / Suggestions
```

### Key Frontmatter Fields

| Field | Required | Notes |
|---|---|---|
| `name` | ✅ | Lowercase + hyphens. Used for `@`-mention and explicit invocation. |
| `description` | ✅ | Claude uses this to decide when to delegate. Write it carefully. |
| `tools` | No | Allowlist. If omitted, the subagent **inherits all tools**. Be intentional. |
| `disallowedTools` | No | Denylist. Applied before `tools` allowlist. |
| `model` | No | `haiku` / `sonnet` / `opus` / `inherit`. Defaults to `inherit`. |
| `memory` | No | `user` / `project` / `local`. Enables persistent knowledge across sessions. |
| `permissionMode` | No | `default` / `acceptEdits` / `auto` / `bypassPermissions` / `plan`. |
| `maxTurns` | No | Hard cap on agentic turns before the subagent stops. |
| `skills` | No | Preload skill content into the subagent's context at startup. |
| `background` | No | Set `true` to always run as a background (non-blocking) task. |
| `isolation` | No | `worktree` gives the subagent an isolated git worktree copy. |

---

## ✅ SHOULD — Design Guidelines

### 1. Write action-oriented descriptions with trigger hints

Claude uses the `description` field to decide when to delegate. A vague description means Claude rarely invokes the agent automatically.

**Pattern:** `[Trigger condition] + [What the agent does] + [What it returns]`

```yaml
# ✅ Good
description: Expert code review specialist. Proactively reviews code for quality,
  security, and maintainability. Use immediately after writing or modifying code.

# ❌ Bad
description: A code review agent.
```

Include explicit phrases like `"Use proactively"`, `"Use immediately after..."`, or `"Invoke when..."` to encourage automatic delegation.

### 2. Restrict tools to the minimum required

If you omit the `tools` field, the subagent inherits **all** tools from the main conversation, including MCP servers. This is almost never what you want for specialized agents.

```yaml
# ✅ Read-only reviewer — cannot modify files
tools: Read, Grep, Glob, Bash

# ✅ Implementer — needs write access
tools: Read, Write, Edit, Bash

# ✅ Security auditor — read-only, no Bash execution
tools: Read, Grep, Glob
```

Apply the **principle of least privilege**: grant only what the agent needs for its single job.

### 3. Match the model to the task complexity

| Task type | Recommended model |
|---|---|
| File search, exploration, codebase navigation | `haiku` (fast, cheap) |
| Code review, analysis, test writing | `sonnet` |
| Architecture decisions, complex reasoning | `opus` or `inherit` |

Routing research tasks to Haiku while keeping complex reasoning on Sonnet/Opus significantly reduces cost without sacrificing quality.

### 4. Enable `memory` for agents that accumulate knowledge

Agents that learn from repeated use (e.g., a code reviewer that discovers project-specific patterns) benefit from persistent memory.

```yaml
memory: project   # stored in .claude/agent-memory/<name>/ — shareable via git
memory: user      # stored in ~/.claude/agent-memory/<name>/ — personal, cross-project
memory: local     # stored locally, not committed to version control
```

Include instructions in the agent's system prompt to actively maintain its memory:

```
After each review session, update your agent memory with patterns,
conventions, and recurring issues you discovered in this codebase.
```

### 5. Check project agents into version control

Project-level agents (`.claude/agents/`) should be committed to the repository so the entire team benefits from the same configurations. Treat them like code — review changes, iterate, and improve collaboratively.

### 6. Use `SubagentStop` hooks to chain pipeline stages

Register a `SubagentStop` hook that reads a task queue and prints the next suggested command. This enables a reproducible pipeline (e.g., architect → implementer → reviewer) without manual prompting between stages.

---

## ❌ SHOULD NOT — Common Mistakes

### 1. Do not create agents scoped by layer (backend / frontend / API)

"Backend Agent" is a domain, not a job. Such agents end up doing too many different things and provide no specialization benefit. Divide by **workflow concern**, not by technical layer:

| ❌ Avoid | ✅ Prefer |
|---|---|
| `backend-agent` | `ef-core-reviewer`, `masstransit-consumer-reviewer` |
| `frontend-agent` | `accessibility-auditor`, `typescript-type-checker` |
| `api-agent` | `endpoint-contract-validator`, `openapi-linter` |

### 2. Do not let subagents spawn other subagents

Subagents **cannot** spawn other subagents. If your workflow requires nested delegation, either chain subagents from the main conversation or use Skills for reusable steps within a single context.

### 3. Do not use `bypassPermissions` unless necessary

`bypassPermissions` skips all permission prompts. Use it only in fully automated, non-interactive CI pipelines where you have complete control over the environment and understand the implications.

### 4. Do not use a subagent for quick questions

For quick questions about content already in your conversation, use `/btw` instead. It has access to full context, no tool access, and the answer is discarded rather than added to history — no context cost.

### 5. Do not skip the description trigger hint

Without a trigger hint, Claude defaults to handling tasks itself and rarely auto-delegates. This is the single most common reason teams find that "agents never get used."

### 6. Do not create an agent when a Skill is sufficient

| Use a **Skill** when... | Use a **Subagent** when... |
|---|---|
| Reusable instructions running in main context | Work needs isolated context window |
| You need the agent's full conversation history available | Output would pollute main context with logs/files |
| Low volume of output | High volume of output (test runs, file scans) |
| Quick, targeted changes | Self-contained tasks returning only a summary |

---

## When to Use Subagents vs. Main Conversation

| Scenario | Recommendation |
|---|---|
| Task needs frequent back-and-forth | Main conversation |
| Multiple phases sharing significant context | Main conversation |
| Quick, targeted change (one-sentence diff) | Main conversation |
| Task produces large verbose output (test logs, file dumps) | Subagent |
| Enforce specific tool restrictions or permissions | Subagent |
| Independent, self-contained work returning a summary | Subagent |
| Research across 10+ files | Subagent |
| 3+ independent pieces of work | Subagents (parallel) |

---

## Parallel vs. Sequential Patterns

**Sequential** — use when each stage depends on the previous output:

```
planner-researcher → architect-reviewer → implementer → code-reviewer → security-auditor
```

**Parallel** — use when tasks are independent:

```
[ef-core-reviewer]  [masstransit-reviewer]  [test-writer]
        ↓                    ↓                    ↓
              [main conversation synthesizes results]
```

Instruct Claude explicitly in `CLAUDE.md` which pattern to apply, since Claude cannot infer task dependencies on its own:

```markdown
## Subagent Delegation Rules

During implementation, delegate as follows:
- Architecture decisions → `architect-reviewer` (sequential, blocking)
- Code review + security audit → run in **parallel** after implementation
- DB/query concerns → `ef-core-reviewer` (can run in parallel with implementation)
```

---

## Pre-Creation Checklist

Before creating a new subagent, answer these questions:

```
□ Does this agent have exactly ONE job?
□ Is the description action-oriented with a clear trigger hint?
□ Are tools restricted to the minimum required?
□ Is the task complex enough to justify context isolation?
  (10+ files to explore, or 3+ independent pieces of work)
□ Will this task be repeated often enough to justify a dedicated agent?
□ Does this agent need persistent memory across sessions?
□ Is the model choice appropriate for the task complexity?
□ Should this agent be committed to version control?
```

If most answers are "no" — consider a **Skill** instead of a subagent.

---

## Recommended Agents for a .NET Billing/Payments Platform

The following agent set is ordered by priority for a dual-stack .NET platform (Framework 4.8 + .NET 8/10) with MassTransit, EF Core, FastEndpoints, and payment gateway integrations.

### Tier 1 — Must Have

| Agent | Scope | Tools | Model | Memory |
|---|---|---|---|---|
| `architect-reviewer` | Architecture decisions before implementation | `Read, Glob, Grep` | `opus` | `project` |
| `code-reviewer` | Code quality, security, patterns after every change | `Read, Grep, Glob, Bash` | `sonnet` | `project` |
| `implementer` | Feature implementation from approved spec/ADR | `Read, Write, Edit, Bash` | `inherit` | — |

### Tier 2 — Strongly Recommended

| Agent | Scope | Tools | Model | Memory |
|---|---|---|---|---|
| `migration-auditor` | .NET Framework 4.8 → .NET 10 migration tasks | `Read, Glob, Grep, Bash` | `sonnet` | `project` |
| `test-writer` | xUnit test authoring for existing code; MSTest → xUnit migration | `Read, Write, Edit, Bash` | `sonnet` | `project` |
| `ef-core-reviewer` | EF Core query analysis, N+1 detection, index decisions | `Read, Grep, Glob` | `sonnet` | `project` |

### Tier 3 — As Needed

| Agent | Scope | Tools | Model | Memory |
|---|---|---|---|---|
| `planner-researcher` | Pre-implementation research and planning | `Read, Glob, Grep` | `haiku` | — |
| `security-auditor` | Payment code (Stripe/BPoint/Braintree), auth endpoints | `Read, Grep, Glob` | `sonnet` | `project` |

---

## References

- [Anthropic — Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Anthropic — Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic Engineering — Seeing like an agent](https://claude.com/blog/seeing-like-an-agent)
- [PubNub Engineering — Best practices for Claude Code subagents](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/)
- [Skywork AI — Mastering Claude Agent SDK](https://skywork.ai/blog/?p=10874)
