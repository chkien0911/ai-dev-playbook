# Sync Copilot Configuration from Claude

Reads all Claude Code configuration and synchronizes it to `.github/`,
keeping content identical in intent with only syntax/structural differences allowed.

Run from the repository root with no arguments.

-----

## Reference: Full Structure of Both Tools

### Claude Code — project layout

```
<project-root>/
├── CLAUDE.md                          # Project-wide always-on rules (loaded every session)
├── CLAUDE.local.md                    # Personal overrides, gitignored
├── <subdir>/CLAUDE.md                 # Subdirectory-scoped rules (auto-loaded when working in that dir)
│
└── .claude/
    ├── rules/                         # Split rules files — all auto-loaded alongside CLAUDE.md
    │   └── *.md                       # e.g. code-style.md, testing.md, security.md
    ├── agents/
    │   └── <name>.md                  # Subagent definitions (YAML frontmatter + prompt body)
    ├── commands/                      # Legacy slash commands; still supported
    │   └── <name>.md
    └── skills/
        └── <skill-name>/              # One directory per skill
            ├── SKILL.md               # Required entrypoint (YAML frontmatter + instructions)
            ├── <supporting-file>.md   # Optional: templates, examples, reference docs
            └── scripts/
                └── <script>          # Optional: scripts SKILL.md may reference
```

**CLAUDE.md / rules file** — no frontmatter, plain markdown rules.

**SKILL.md frontmatter fields:**

```yaml
---
name: skill-name                   # Slash command name (optional, defaults to directory name)
description: When to use this      # How Claude decides to auto-invoke
when_to_use: Additional context    # (optional) More trigger guidance
disable-model-invocation: true     # (optional) Prevent Claude from auto-invoking
allowed-tools: Read Grep           # (optional) Space-separated tool allowlist
context: fork                      # (optional) "fork" = run in subagent
---
```

**Agent frontmatter fields:**

```yaml
---
name: agent-name
description: What this agent does and when to use it
model: sonnet                      # (optional) model override
tools: [Read, Write, Bash]         # Claude Code tool names
---
```

-----

### GitHub Copilot — `.github/` layout

```
.github/
├── copilot-instructions.md            # Repo-wide always-on rules (maps from CLAUDE.md)
│
├── instructions/                      # Path-scoped or global supplemental rules
│   └── <name>.instructions.md        # Maps from .claude/rules/*.md
│
├── prompts/                           # On-demand reusable task templates
│   └── <name>.prompt.md              # Maps from .claude/commands/*.md
│
├── agents/                            # Custom agent personas
│   └── <name>.agent.md               # Maps from .claude/agents/*.md
│
└── skills/                            # Agent skills — same open standard as Claude
    └── <skill-name>/                  # Maps from .claude/skills/<skill-name>/
        ├── SKILL.md                   # Same format, same frontmatter
        └── <supporting-files>         # Mirrored as-is
```

**`.instructions.md` frontmatter:**

```yaml
---
applyTo: "**"                          # Required glob. "**" = all files (global)
description: "Optional description"   # (optional)
excludeAgent: "code-review"            # (optional) "code-review" | "coding-agent"
---
```

**`.prompt.md` frontmatter:**

```yaml
---
description: 'What this prompt does'  # Required — shown in UI
mode: 'agent'                          # 'ask' | 'edit' | 'agent'
tools: ['codebase', 'editFiles', 'runCommands', 'search']  # (optional)
model: 'claude-sonnet-4-5'            # (optional)
---
```

**`.agent.md` frontmatter:**

```yaml
---
name: 'Display Name'                  # (optional) defaults to filename
description: 'Agent purpose'          # Shown in agent picker
tools: ['read', 'edit', 'search', 'runCommands']  # (optional) omit = all tools
model: 'claude-sonnet-4-5'            # (optional)
---
```

**`skills/<name>/SKILL.md`** — identical format and frontmatter to Claude’s SKILL.md.
No additional Copilot-specific frontmatter needed. Copilot also reads `.claude/skills/`
directly, so `.github/skills/` is the canonical shared location.

-----

## Mapping Rules

### 1. `CLAUDE.md` → `.github/copilot-instructions.md`

|Claude source             |Copilot target                                                               |
|--------------------------|-----------------------------------------------------------------------------|
|`CLAUDE.md` (project root)|`.github/copilot-instructions.md`                                            |
|`<subdir>/CLAUDE.md`      |`.github/instructions/<subdir>.instructions.md` with `applyTo: "<subdir>/**"`|

- No frontmatter added to `copilot-instructions.md` — Copilot reads it as-is.
- If the root `CLAUDE.md` has sections that explicitly scope to a path, extract those sections into a separate `.instructions.md` with the appropriate `applyTo` glob.

### 2. `.claude/rules/*.md` → `.github/instructions/*.instructions.md`

One-to-one by filename: `testing.md` → `testing.instructions.md`.

Frontmatter to add:

```yaml
---
applyTo: "**"
---
```

If a rules file is clearly scoped (e.g. `api-rules.md` mentions it applies only to `src/Api/`),
set `applyTo` to the matching glob instead of `"**"`.

### 3. `.claude/commands/*.md` → `.github/prompts/*.prompt.md`

One-to-one by filename: `sync-copilot.md` → `sync-copilot.prompt.md`.

Frontmatter to add:

```yaml
---
description: '<extract from first heading or opening line of the command>'
mode: 'agent'
tools: ['codebase', 'editFiles', 'runCommands', 'search']
---
```

Adjust `tools` based on what the command actually does. Do NOT auto-add a `model` field.

### 4. `.claude/agents/*.md` → `.github/agents/*.agent.md`

One-to-one by filename: `code-reviewer.md` → `code-reviewer.agent.md`.

Frontmatter translation:

|Claude field  |Copilot field |Notes                                      |
|--------------|--------------|-------------------------------------------|
|`name:`       |`name:`       |Copy as-is                                 |
|`description:`|`description:`|Copy as-is                                 |
|`tools:`      |`tools:`      |Translate names — see Tool Name Translation|
|`model:`      |`model:`      |Copy as-is if present; do NOT add if absent|

### 5. `.claude/skills/<name>/` → `.github/skills/<name>/`

**Skills use the same open standard — SKILL.md format is identical.**
Copilot also reads `.claude/skills/` directly, so strictly speaking no copy is needed.
However, `.github/skills/` is the canonical cross-tool location.

Sync rules:

- Mirror the **entire directory** — SKILL.md and all supporting files — preserving structure.
- SKILL.md frontmatter fields are identical; **no translation needed**.
- The only changes are path references in file content (see Path Reference Translation below).
- If `.claude/skills/<name>/` already exists and `.github/skills/<name>/` does not, copy it.
- If both exist, diff only for path reference drift; content should be otherwise identical.

-----

## Path Reference Translation (apply inside ALL file content)

Any path pointing into `.claude/` must be translated to the Copilot-side equivalent:

|Claude path (in file content)|Copilot equivalent                           |
|-----------------------------|---------------------------------------------|
|`CLAUDE.md`                  |`.github/copilot-instructions.md`            |
|`.claude/rules/<name>.md`    |`.github/instructions/<name>.instructions.md`|
|`.claude/commands/<name>.md` |`.github/prompts/<name>.prompt.md`           |
|`.claude/agents/<name>.md`   |`.github/agents/<name>.agent.md`             |
|`.claude/skills/<n>/SKILL.md`|`.github/skills/<n>/SKILL.md`                |
|`.claude/skills/<n>/<file>`  |`.github/skills/<n>/<file>`                  |

This applies to:

- Markdown links: `[text](.claude/skills/x/SKILL.md)` → `[text](.github/skills/x/SKILL.md)`
- Inline text: `"see .claude/rules/testing.md"` → `"see .github/instructions/testing.instructions.md"`
- `@` file references: `@.claude/rules/testing.md` → `#file:.github/instructions/testing.instructions.md`

-----

## Syntax Translations (apply to ALL file content)

### File reference syntax

|Claude         |Copilot             |
|---------------|--------------------|
|`@path/to/file`|`#file:path/to/file`|

### Tool name translation (for `tools:` fields in agents and prompts)

|Claude Code tool        |Copilot tool alias                                       |
|------------------------|---------------------------------------------------------|
|`Read`                  |`read`                                                   |
|`Write`                 |`edit`                                                   |
|`Edit`                  |`edit`                                                   |
|`Bash`                  |`runCommands`                                            |
|`Grep` / `Glob` / `LS`  |`search`                                                 |
|`WebSearch` / `WebFetch`|`web`                                                    |
|`Task` (subagent spawn) |*(no equivalent — flag for review, do not drop silently)*|
|`TodoRead` / `TodoWrite`|*(no equivalent — drop)*                                 |

### Variable / argument syntax

|Claude                                |Copilot                     |
|--------------------------------------|----------------------------|
|`$ARGUMENTS`                          |`${input:arguments}`        |
|`/command-name` reference in body text|`the \`command-name` prompt`|

### AI tool name references in body text

|Claude text                                 |Copilot text                    |
|--------------------------------------------|--------------------------------|
|`Claude Code`                               |`GitHub Copilot`                |
|`Claude` (referring to the tool/product)    |`Copilot`                       |
|`Claude` (referring to the AI model/persona)|keep as-is or `the AI assistant`|

-----

## Steps

### Step 1 — Inventory

Read all files recursively from:

- `CLAUDE.md`, `CLAUDE.local.md` at project root
- `<subdir>/CLAUDE.md` in any subdirectory
- `.claude/rules/`, `.claude/agents/`, `.claude/commands/`, `.claude/skills/`
- `.github/copilot-instructions.md`, `.github/instructions/`, `.github/prompts/`, `.github/agents/`, `.github/skills/`

Build a mapping table: Claude source → Copilot target → current sync status.

Skip `CLAUDE.local.md` — it is personal/gitignored and has no Copilot equivalent.

### Step 2 — Diff each pair

Classify every mapped pair:

- `[IN SYNC]` — content equivalent after expected translations, no action needed
- `[UPDATE NEEDED]` — content has drifted; describe what changed at section level
- `[MISSING IN COPILOT]` — Claude file exists, no Copilot counterpart → will be created
- `[ORPHAN IN COPILOT]` — Copilot file has no Claude source → flag, do NOT auto-delete

Print the full diff report before writing anything.

### Step 3 — Confirm

```
Files to write/update:  X
Files already in sync:  Y
Orphans flagged:        Z

Proceed? (yes / review first / cancel)
```

Do not write anything until confirmed.

### Step 4 — Translate and write

For each `[UPDATE NEEDED]` and `[MISSING IN COPILOT]` file:

1. Determine target path from Mapping Rules
1. Build correct frontmatter per the mapping tables
1. Translate `@file` references → `#file:` syntax
1. Translate `.claude/` path references throughout content
1. Translate tool names in `tools:` fields
1. Translate AI tool name references in body text (`Claude Code` → `GitHub Copilot` etc.)
1. For skills: mirror entire directory; translate only path references, nothing else
1. For Claude-specific features with no Copilot equivalent, add inline comment:
   `<!-- NOTE: Claude-specific feature — no Copilot equivalent. Kept as context. -->`
1. Write to target path

**Do NOT rewrite, summarize, or reorder content.** Only apply the mechanical translations above.

### Step 5 — Summary

```
Sync complete.

Written/Updated:
  <list of files>

In sync (skipped):
  <list>

Flagged for manual review:
  ORPHANS (exist in .github/ only — verify before deleting):
    <list>
  TOOLS (verify these match your available Copilot tools):
    <file>: tools: [...]
  MODEL FIELDS (not auto-set — add manually if needed):
    <list of agent/prompt files where model: was absent>
  CLAUDE-SPECIFIC FEATURES (kept as inline comments):
    <list>
```

-----

## Known Gaps — Do Not Auto-Fill

|Item                                  |Why                                                                                                          |
|--------------------------------------|-------------------------------------------------------------------------------------------------------------|
|`model:` in agents/prompts            |Project may use a specific model; wrong default causes routing errors. Only copy if present in Claude source.|
|`Task` tool (subagent spawn)          |No direct Copilot equivalent — flag, keep as comment                                                         |
|`context: fork` in SKILL.md           |Copilot skills always run inline; no fork mode                                                               |
|`#memories` blocks                    |No Copilot equivalent                                                                                        |
|`ultrathink` / extended thinking hints|No Copilot equivalent                                                                                        |
|`handoffs:` in agents                 |Copilot-only feature — do not generate for Claude-sourced agents                                             |
|CLAUDE.local.md                       |Personal/gitignored — no sync                                                                                |