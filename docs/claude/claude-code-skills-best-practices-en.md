# Claude Code Skills — Best Practices

> **Sources:** Anthropic official documentation ([code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)), Anthropic API Docs ([platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)), Anthropic Engineering blog ([anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)), and authoritative community references.

---

## What Is a Skill?

A skill is a directory containing a `SKILL.md` file — organized instructions, scripts, and resources that give Claude additional capabilities. Skills extend what Claude can do without requiring you to paste the same playbook into every conversation.

> **Anthropic's analogy:** Building a skill for an agent is like putting together an onboarding guide for a new hire. Instead of building custom-designed agents for each use case, anyone can now specialize their agents with composable capabilities by capturing and sharing their procedural knowledge.

### How Skills Load (Progressive Disclosure)

This is the core design principle that makes skills scalable and token-efficient:

| Level | What loads | When |
|---|---|---|
| **Level 1** — Metadata | `name` + `description` from frontmatter | At startup, always |
| **Level 2** — Instructions | Full body of `SKILL.md` | When Claude decides the skill is relevant |
| **Level 3** — Supporting files | `reference.md`, scripts, examples | Only when needed for the specific task |

This means you can install many skills without a context penalty — Claude only knows each skill exists and when to use it. The actual content loads on demand.

---

## Skills vs. Commands vs. Subagents

Understanding where skills fit relative to other Claude Code features prevents misuse.

### Skills vs. Commands

Custom commands (`.claude/commands/`) have been **merged into skills**. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Skills are the recommended path going forward because they support additional features: supporting files, auto-invocation control, and subagent execution.

### Skills vs. Subagents

| | Skills | Subagents |
|---|---|---|
| **Context** | Runs inline in main conversation | Isolated context window |
| **History access** | Full conversation history | Fresh context, no history |
| **Output** | Adds to main thread | Returns summary only |
| **Best for** | Reference knowledge, reusable workflows, conventions | Self-contained heavy tasks, parallel work, verbose output |
| **Can spawn others?** | No | No (subagents cannot spawn subagents) |

**Use a Skill when** you want reusable instructions that run in the current conversation context.
**Use a Subagent when** a task would flood your main context with logs, file contents, or output you won't reference again.

> **Note:** These two features are complementary, not competing. You can preload skills into a subagent's context using the `skills:` frontmatter field, and you can run a skill in a subagent using `context: fork`.

---

## Skill Directory Structure

```
.claude/skills/
└── my-skill/
    ├── SKILL.md           # Main instructions (required)
    ├── reference.md       # Detailed reference loaded on demand
    ├── examples/
    │   └── sample.md      # Example output showing expected format
    └── scripts/
        └── validate.sh    # Executable script Claude can run
```

The `SKILL.md` is the only required file. Supporting files are optional and let you keep the core skill lean while still bundling detailed reference material that loads only when needed.

---

## Two Types of Skills

### 1. Capability Uplift Skills

Give Claude abilities it does not have on its own — document creation, browser automation, web scraping, PDF manipulation, specific CLI workflows.

```yaml
---
name: processing-pdfs
description: Extract text and tables from PDF files, fill forms, merge documents.
  Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

Use pdfplumber for text extraction:

```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

For form filling, see [forms.md](forms.md).
For advanced extraction, see [reference.md](reference.md).
```

### 2. Encoded Preference Skills

Guide Claude to follow your team's specific workflow for things it already knows how to do — NDA reviews, commit message formats, API design conventions, incident report templates.

```yaml
---
name: api-conventions
description: API design patterns and conventions for this codebase. Use when
  writing new endpoints, reviewing API changes, or designing new resources.
---

When writing API endpoints:
- Use RESTful naming: nouns for resources, verbs via HTTP methods
- Return consistent error format: `{ "error": { "code": "...", "message": "..." } }`
- Always validate input with FluentValidation before processing
- Include Datadog structured logging on entry and exit
```

---

## Frontmatter Reference

```yaml
---
name: my-skill                     # Required. Lowercase, hyphens, max 64 chars
description: What this skill does  # Strongly recommended. Max 1024 chars
when_to_use: Additional trigger context  # Optional. Extends description
disable-model-invocation: true     # Optional. Prevents auto-invocation — explicit /skill-name only
allowed-tools: Read Grep           # Optional. Pre-approve tools for this skill
context: fork                      # Optional. Run in isolated subagent context
---
```

### Key Fields Explained

**`description`** — The most important field. Claude uses this to decide when to auto-invoke. Front-load the key use case. The combined `description` + `when_to_use` text is truncated at 1,536 characters in the skill listing.

**`disable-model-invocation: true`** — Use for procedural/action skills (deployments, migrations, releases) that should only run when you explicitly type `/skill-name`. Prevents accidental auto-invocation.

**`context: fork`** — Runs the skill in an isolated subagent context. The skill content becomes the subagent's prompt. It will not have access to your conversation history. Use for complex analysis that generates verbose output you do not need in the main thread.

**`allowed-tools`** — Pre-approve specific tools so Claude does not prompt for permission mid-skill. Useful for scripted workflows that need Bash, Read, or Write access.

---

## ✅ SHOULD — Design Guidelines

### 1. Be concise — trust Claude's baseline intelligence

Only add context Claude does not already have. Challenge every paragraph: "Does Claude really need this explanation? Can I assume it already knows this?"

```yaml
# ✅ Good — ~50 tokens, assumes Claude knows what PDFs are
## Extract PDF text

Use pdfplumber for text extraction:
```python
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

# ❌ Bad — ~150 tokens, over-explains obvious things
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text and images. To extract text from a PDF, you'll need to use a library.
There are many libraries available but pdfplumber is recommended because...
```

### 2. Match instruction specificity to task fragility

Use the **degree of freedom** model from Anthropic's official best practices:

| Task type | Freedom level | Use |
|---|---|---|
| Code reviews, open-ended analysis | High — natural language guidance | Context determines the best approach |
| Report generation, templated output | Medium — pseudocode with parameters | A preferred pattern exists but variation is acceptable |
| Database migrations, deployments, releases | Low — exact scripts, no deviation | Consistency is critical; errors are costly |

> **Anthropic's analogy:** Think of Claude as a robot on a path. On a **narrow bridge with cliffs** — give exact instructions (low freedom). In an **open field** — give general direction and trust Claude (high freedom).

### 3. Use gerund-form names for clarity

```
# ✅ Preferred
processing-pdfs
analyzing-spreadsheets
reviewing-pull-requests
writing-changelogs

# ✅ Also acceptable
pdf-processing
pr-review

# ❌ Avoid
helper, utils, tools        # Too vague
documents, data             # Overly generic
anthropic-helper            # Reserved words
```

### 4. Use `disable-model-invocation: true` for action skills

Skills that trigger operations (deploy, migrate, release) should never run automatically. Require explicit invocation.

```yaml
---
name: deploy-staging
description: Deploy the current branch to the staging environment
disable-model-invocation: true
---

Deploy to staging:
1. Run the test suite — `dotnet test`
2. Build the Docker image — `docker build -t app:staging .`
3. Push and deploy — `./scripts/deploy.sh staging`
```

### 5. Split large skills into supporting files

When `SKILL.md` grows unwieldy, move sections to separate files and reference them. Claude reads supporting files on demand — no token cost until needed.

```markdown
# In SKILL.md
This skill handles EF Core patterns. For query optimization, see [query-patterns.md](query-patterns.md).
For migration conventions, see [migrations.md](migrations.md).
```

### 6. Use `context: fork` for output-heavy analysis

When a skill's output would pollute the main conversation with logs, traces, or file contents, run it in an isolated context.

```yaml
---
name: analyzing-connection-pool
description: Analyze connection pool usage across the codebase
context: fork
disable-model-invocation: true
---

Analyze all places where DbContext is instantiated. Look for:
- Parallel.ForEach patterns with service locator
- Missing using statements
- Singleton-scoped DbContext registrations

Return a prioritized list of findings with file paths and line numbers.
```

### 7. Iterate with Claude to discover what context it actually needs

> **Anthropic engineering guidance:** As you work on a task with Claude, ask Claude to capture its successful approaches and common mistakes into reusable context within a skill. If it goes off track, ask it to self-reflect on what went wrong. This helps you discover what context Claude actually needs instead of trying to anticipate it upfront.

### 8. Check project skills into version control

Project skills (`.claude/skills/`) should be committed to the repository so the whole team benefits. Treat them as code — review changes, iterate, and improve collaboratively.

---

## ❌ SHOULD NOT — Common Mistakes

### 1. Do not over-explain things Claude already knows

Every token in `SKILL.md` competes with conversation history once the skill loads. Skills are not documentation — they are targeted additions to Claude's context.

### 2. Do not put action skills without `disable-model-invocation`

A deployment or migration skill that triggers automatically based on a keyword match is dangerous. Always add `disable-model-invocation: true` to any skill that performs irreversible operations.

### 3. Do not write one giant `SKILL.md` with everything

Use supporting files and progressive disclosure. A `SKILL.md` that contains every edge case, every example, and every reference document loads all of it into context every time — defeating the purpose of progressive disclosure.

### 4. Do not use `context: fork` for reference/convention skills

`context: fork` creates a subagent with **no access to your conversation history**. A skill like `api-conventions` needs to see your current code to apply conventions — forking it would make it useless.

```yaml
# ❌ Wrong — reference skills need conversation context
---
name: api-conventions
context: fork  # This subagent can't see your current code
---

# ✅ Correct — run inline
---
name: api-conventions
description: API design conventions. Use when writing or reviewing endpoints.
---
```

### 5. Do not use vague or overly generic names

```
# ❌ Avoid
helper
utils
do-stuff
my-skill

# ✅ Be specific
reviewing-fastendpoints
migrating-mstest-to-xunit
generating-ef-migrations
```

### 6. Do not skip testing across models

Skills behave differently across Haiku, Sonnet, and Opus. A skill that works perfectly on Opus may need more explicit guidance for Haiku.

### 7. Do not install skills from untrusted sources without auditing

Skills can include executable scripts. Always read the full contents of a third-party skill before installing — pay attention to external network calls, data exfiltration patterns, and unsafe shell commands.

---

## Skills vs. Subagents — Decision Matrix

| Question | If YES → | If NO → |
|---|---|---|
| Does the task produce verbose output (logs, file dumps, test results)? | Subagent | Skill (inline) |
| Does the task need full conversation history? | Skill (inline) | Either |
| Is this a reusable workflow/convention applied to current work? | Skill (inline) | Subagent |
| Does the task require context isolation for clean output? | Skill with `context: fork` | Skill (inline) |
| Will the same instructions be reused across many subagents? | Skill preloaded via `skills:` frontmatter | — |
| Is the task entirely self-contained with a clear summary output? | Subagent | Skill |

---

## When to Use Skills vs. CLAUDE.md

Both `CLAUDE.md` and skills store instructions for Claude — but they serve different purposes.

| | `CLAUDE.md` | Skills |
|---|---|---|
| **Loads** | Always, at every session start | On demand, when relevant |
| **Token cost** | Always counted | Near zero until triggered |
| **Best for** | Short facts, project-wide rules, always-applicable context | Long procedures, optional workflows, reference material |
| **Trigger** | Automatic, unconditional | By relevance (auto) or `/skill-name` (explicit) |

> **Anthropic guidance:** If a section of `CLAUDE.md` has grown into a procedure rather than a fact, move it to a skill.

---

## Pre-Creation Checklist

Before creating a new skill, answer these questions:

```
□ Is this instructions Claude doesn't already have? (not just restating what Claude knows)
□ Is the description action-oriented with a clear trigger condition?
□ Is the body concise — only what Claude actually needs?
□ Should supporting files be split out to avoid loading everything at once?
□ Is this an action skill? → Add disable-model-invocation: true
□ Will the output pollute the main context? → Add context: fork
□ Is this project-specific? → Store in .claude/skills/ and commit to version control
□ Is this a fact/rule that applies always? → Consider CLAUDE.md instead
```

---

## References

- [Anthropic — Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Anthropic API Docs — Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Anthropic Engineering — Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Agent Skills open standard](https://agentskills.io)
- [Level Up Coding — A Mental Model for Claude Code: Skills, Subagents, and Plugins](https://levelup.gitconnected.com/a-mental-model-for-claude-code-skills-subagents-and-plugins-3dea9924bf05)
- [Firecrawl — Best Claude Code Skills to Try in 2026](https://www.firecrawl.dev/blog/best-claude-code-skills)
