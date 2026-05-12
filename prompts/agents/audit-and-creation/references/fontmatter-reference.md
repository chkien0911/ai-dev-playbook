# Agent Frontmatter Reference

All supported YAML frontmatter fields for `.claude/agents/*.md`.
Read this when writing or reviewing agent definitions.

-----

## Required fields

|Field        |Description                                                                                                                                                                                  |
|-------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`name`       |Unique identifier. Lowercase letters and hyphens only.                                                                                                                                       |
|`description`|When Claude should delegate to this agent. Write as a routing rule, not a job posting. Front-load the primary use case. Use “PROACTIVELY” if Claude should auto-delegate without being asked.|

## Tool control

|Field            |Description                                                                                                                                                                                       |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`tools`          |Comma-separated allowlist: `Read, Grep, Glob, Bash, Write, Edit, Agent`, etc. Inherits all tools if omitted. Supports `Agent(agent_type)` syntax to restrict which subagents this agent can spawn.|
|`disallowedTools`|Tools to deny, removed from inherited or specified list. Use one of `tools` OR `disallowedTools`, not both.                                                                                       |

**Common tool sets:**

- Read-only: `tools: Read, Grep, Glob` or `disallowedTools: Write, Edit, Bash`
- No web: `disallowedTools: WebFetch, WebSearch`
- No spawning subagents: `disallowedTools: Agent`

## Model and effort

|Field   |Values                                             |Notes                                                                                                                                     |
|--------|---------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
|`model` |`haiku`, `sonnet`, `opus`, full model ID, `inherit`|Default: `inherit` (follows main session). Use `haiku` for read-only/simple tasks to save cost. Use `opus` for high-stakes reasoning only.|
|`effort`|`low`, `medium`, `high`, `max`                     |Opus 4 only. Default: inherits from session. `max` = maximum reasoning depth.                                                             |

## Permissions

|Field           |Values                                               |Notes                                                                                                                                                                                                                                                            |
|----------------|-----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`permissionMode`|`default`, `plan`, `acceptEdits`, `bypassPermissions`|`plan` = read-only until explicitly approved (good for risky ops). `acceptEdits` = auto-approve file edits. `bypassPermissions` = no gates (CI/automation only, never interactive). If parent uses `bypassPermissions` or `acceptEdits`, it overrides this field.|

## Execution control

|Field       |Description                                                                                                                                                         |
|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`maxTurns`  |Max agentic turns before the subagent stops. Omit unless runaway risk is real.                                                                                      |
|`background`|`true` = always run as a background (async) task. Omit for foreground.                                                                                              |
|`isolation` |`worktree` = run in a temporary isolated git worktree. Cost: setup time. Benefit: no risk of polluting the working tree. Add for destructive or experimental agents.|

## Skills and memory

|Field   |Description                                                                                                                                                                                                                                               |
|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`skills`|List of skill names to preload at startup. Full content is injected into the agent’s context — not just made available. Only list skills the agent actually uses; each adds to context cost.                                                              |
|`memory`|`project` = persists to `.claude/agent-memory/[name]/`, committed to git, shared with team. `user` = persists to `~/.claude/agent-memory/[name]/`, personal only. `none` = no persistence (default). First 200 lines of MEMORY.md are injected at startup.|

## MCP and hooks

|Field       |Description                                                                                         |
|------------|----------------------------------------------------------------------------------------------------|
|`mcpServers`|MCP servers scoped to this agent only. Limits tool description tokens to only what this agent needs.|
|`hooks`     |Lifecycle hooks scoped to this agent. `PreToolUse`, `PostToolUse`, and `Stop` are most common.      |


> **Note:** Plugin subagents do not support `hooks`, `mcpServers`, or `permissionMode`. Copy to `.claude/agents/` if you need these.

-----

## Full example — security reviewer

```yaml
---
name: security-reviewer
description: >
  Reviews code for security vulnerabilities. Use PROACTIVELY after writing
  authentication, authorization, data-handling, or input-validation code.
  Triggers on: "check for security issues", "audit my auth", "review this for vulns".
tools: Read, Grep, Glob
model: opus
effort: high
permissionMode: default
skills:
  - security-patterns
memory: project
---

You are a senior application security engineer. Review the specified code for
security vulnerabilities before it ships.

For each finding, return:
- File path and line number
- Vulnerability class (e.g. SQL injection, broken auth, insecure deserialization)
- Severity: Critical / High / Medium / Low
- Specific remediation with code example

Stop after 15 findings. If no issues found, say so explicitly.
Do not suggest architectural refactors — focus only on security.
```

## Full example — read-only explorer with memory

```yaml
---
name: repo-explorer
description: >
  Maps unfamiliar codebases: entry points, core data flow, risk areas.
  Use when starting work on an unfamiliar module or after a large refactor.
  Triggers on: "explore this module", "map the architecture", "where does X start".
tools: Read, Grep, Glob
disallowedTools: Write, Edit, Bash
model: haiku
permissionMode: plan
memory: project
---

Map the requested area of the codebase. Return:
- Entry points and their file paths
- Core data flow (3–5 steps, with file references)
- Key abstractions and their locations
- Likely risk areas or complexity hotspots
- Open questions for the developer

Be concise. Do not read more files than necessary to answer. Stop when you have enough.
```