# AI Coding Assistant Pain Points — Root Cause Analysis & Remediation Guide

> For each category of pain points, this document explains **why** the problem occurs, **who is responsible** (AI, developer, or both), and provides **remediation strategies** ranked by effectiveness.

**Fault attribution:** 🤖 AI-side &nbsp;|&nbsp; 👨‍💻 Dev-side &nbsp;|&nbsp; ⚙️ Systemic (tooling/architecture gap)

**Fix tiers:**
- **Tier 1 — Immediate:** Prompt habits and techniques. Zero setup, works today.
- **Tier 2 — Short-term:** One-time setup: context files, project rules, templates, CI gates.
- **Tier 3 — Long-term:** Tooling investments, AI platform improvements, team processes.

**Effectiveness rating:** 🟢 High &nbsp;|&nbsp; 🟡 Medium &nbsp;|&nbsp; 🔴 Low (but worth noting)

---

## Category A — Context & Memory

### Why This Category Exists

AI coding assistants are fundamentally **stateless**. Each session starts from zero — the model has no persistent memory of past conversations, decisions made, or approaches rejected. The context window is also finite; in long sessions it fills up and old content is silently dropped. Developers instinctively treat AI like a human colleague who accumulates understanding over time. It does not.

**Primary fault:** ⚙️ Systemic (architecture) + 👨‍💻 Dev (not compensating for it)

---

### A1 — Session Context Loss

**Root cause:**
The AI has no persistent memory between or within long sessions. When the context window fills, the model literally cannot "see" earlier messages anymore. There is no mechanism to flag when this is happening. The AI continues confidently, now working from an incomplete picture.

**Why the AI doesn't warn you:** It cannot detect its own context loss — it doesn't know what it has forgotten.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Maintain a `CLAUDE.md` or `copilot-instructions.md` file with key decisions, architecture choices, and rejected approaches. Feed it at the start of every session. | Tier 2 | 🟢 High |
| 2 | At natural breakpoints, ask: *"Summarize the decisions we've made so far."* Paste the summary back as context if the session continues. | Tier 1 | 🟢 High |
| 3 | Keep sessions focused and short. One session = one task. Don't try to do a full feature in one context window. | Tier 1 | 🟡 Medium |
| 4 | Use a decisions log (ADR format) that you paste as context at session start for complex tasks. | Tier 2 | 🟢 High |

---

### A2 — Rejected Pattern Recurrence

**Root cause:**
The AI has no memory of what it proposed and what was rejected. Every response is generated fresh. Without an explicit record of "do not do X because Y," the model will re-derive the same solution from the same training distribution — especially if the problem description is similar.

**Why it keeps happening:** The model doesn't know the conversation happened. From its perspective, this is a new question.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Maintain a `rejected-patterns.md` section in your context file. When you reject an approach, write it down immediately with the reason. | Tier 2 | 🟢 High |
| 2 | When rejecting, be explicit in your prompt: *"Do not suggest X. I've already ruled it out because Y. Proceed with Z instead."* | Tier 1 | 🟢 High |
| 3 | At the start of a new session on the same task, paste the rejection log into the prompt. | Tier 1 | 🟡 Medium |

---

### A3 — Tech Stack Assumption

**Root cause:**
AI is trained on a broad distribution of codebases. When given a task without explicit context, it defaults to the most statistically common stack for that type of problem — which may not match yours. It assumes rather than asks because RLHF training rewards producing complete answers quickly.

**Fault:** 🤖 AI (bias toward action) + 👨‍💻 Dev (not providing stack context upfront)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include a project header in your context file: framework, language version, key libraries, ORM, test framework, DI container. Feed it with every session. | Tier 2 | 🟢 High |
| 2 | State constraints explicitly in every prompt when relevant: *"Using .NET 8, EF Core 8, xUnit — do not suggest Dapper or NUnit."* | Tier 1 | 🟢 High |
| 3 | Ask AI to state its assumptions before writing code: *"Before you start, state what stack you're assuming."* | Tier 1 | 🟡 Medium |

---

### A4 — Multi-file State Drift

**Root cause:**
AI processes files sequentially. It has no live dependency graph or in-memory model of the codebase. When it edits File B, it may be working from a stale mental snapshot of File A that has already been changed — either by the developer or by the AI itself earlier in the session.

**Fault:** ⚙️ Systemic (no live codebase model) + 🤖 AI (no re-read discipline)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Instruct the AI explicitly: *"Before editing each file, re-read its current state. Do not rely on what you read earlier."* | Tier 1 | 🟢 High |
| 2 | Break multi-file tasks into explicit steps with manual checkpoints. Review changes after each file before proceeding. | Tier 1 | 🟢 High |
| 3 | Run the compiler / type checker after each AI edit batch, not at the end. Catch drift early. | Tier 1 | 🟡 Medium |
| 4 | In Claude Code, use `--no-auto-accept` mode and review each change before confirming. | Tier 2 | 🟢 High |

---

### A5 — Insufficient File Reading

**Root cause:**
AI edits what it is shown. It doesn't know what it hasn't been shown. Without explicit instruction to explore the codebase first, it operates on the target file in isolation — unaware of contracts, base classes, or callers that constrain what changes are safe.

**Fault:** 👨‍💻 Dev (not instructing exploration) + 🤖 AI (doesn't proactively explore)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Before any edit task, prompt: *"Before making changes, read all files that reference or depend on [target]. List them and summarize the contracts."* | Tier 1 | 🟢 High |
| 2 | Provide callers and interfaces as context alongside the target file, not just the file being changed. | Tier 1 | 🟢 High |
| 3 | Use a codebase map or dependency summary in your context file for critical modules. | Tier 2 | 🟡 Medium |

---

## Category B — Intent Misalignment

### Why This Category Exists

This is the most human-solvable category. Misalignment happens at the intersection of **underspecified developer prompts** and **AI's trained bias toward action**. RLHF training systematically rewards producing output over asking clarifying questions — because human raters tend to score helpful-looking answers higher than responses that ask for more information. The result is an AI that defaults to doing rather than confirming.

**Primary fault:** 👨‍💻 Dev (prompt quality) + 🤖 AI (action bias from training)

---

### B1 — Description vs. Request Confusion

**Root cause:**
Developers often share context (an error, a log, a snippet) expecting the AI to engage analytically. The AI is trained to be helpful — and "helpful" in its training distribution usually means producing a solution. The distinction between "I'm sharing context" and "I want you to act" is rarely explicit in the prompt.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Be explicit about mode at the start of the prompt: *"Diagnose only — do not suggest any code changes yet."* or *"I want to understand the root cause first."* | Tier 1 | 🟢 High |
| 2 | Use a two-phase workflow: Phase 1 = "explain what's happening." Phase 2 = "now propose a fix." Never combine. | Tier 1 | 🟢 High |
| 3 | Add a default instruction to your context file: *"Unless I explicitly say 'fix this' or 'implement', explain before acting."* | Tier 2 | 🟡 Medium |

---

### B2 — No Clarifying Questions

**Root cause:**
RLHF training: human raters score responses that produce something useful higher than responses that ask questions. Over many training iterations, the model learns that acting looks more helpful than asking. This is a known alignment problem — the incentive to appear helpful overrides the incentive to be correctly helpful.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | End ambiguous prompts with: *"If anything is unclear, ask me one question before starting."* | Tier 1 | 🟢 High |
| 2 | Explicitly invite questions in complex tasks: *"Before you write any code, list your assumptions and flag anything you're unsure about."* | Tier 1 | 🟢 High |
| 3 | Use a prompt template for new features: include scope, constraints, and what not to change. Forces you to specify rather than leave gaps. | Tier 2 | 🟡 Medium |

---

### B3 — Convention Mismatch

**Root cause:**
AI generates code that follows its training distribution's conventions — which is a blend of every codebase it was trained on. Without explicit convention context, it has no way to know your project's specific style, folder structure, or naming rules.

**Fault:** ⚙️ Systemic (no project-aware conventions) + 👨‍💻 Dev (not providing them)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include a coding conventions section in your context file: naming rules, file organization, patterns to use/avoid. | Tier 2 | 🟢 High |
| 2 | Provide a real example from your codebase alongside the request: *"Follow the same structure as [ExistingClass.cs]."* | Tier 1 | 🟢 High |
| 3 | Run a linter/formatter as a post-processing step on all AI output before review. Makes convention enforcement automatic. | Tier 2 | 🟡 Medium |
| 4 | Add an EditorConfig or `.editorconfig` file and instruct the AI to follow it. | Tier 2 | 🟡 Medium |

---

### B4 — Unlabeled Workarounds

**Root cause:**
AI doesn't have a concept of "technical debt register." When asked for a quick solution, it gives one — without flagging the tradeoffs unless asked. The distinction between "right solution" and "fast solution" is invisible to the AI unless the developer makes it explicit.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Ask explicitly: *"Is this the right long-term solution or a workaround? If a workaround, say so clearly and explain what the proper fix would be."* | Tier 1 | 🟢 High |
| 2 | Establish a convention: all temporary fixes get a `// TODO [TEMP]:` comment with an explanation. Instruct the AI to follow this. | Tier 2 | 🟡 Medium |
| 3 | Add a CI check or custom lint rule that flags `TODO [TEMP]` comments for review. | Tier 3 | 🟢 High (long-term) |

---

### B5 — Correction Loop

**Root cause:**
Vague prompts generate outputs that are "plausibly correct" but not specifically right. Each correction shifts the AI in one direction, but without a shared mental model of the goal, the next response overshoots in a different direction. Neither party has established a clear definition of the target.

**Fault:** 👨‍💻 Dev (underspecification) + 🤖 AI (no proactive alignment)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Before asking for code, ask for a plan first: *"Describe how you would approach this before writing anything."* Correct the plan, then execute. | Tier 1 | 🟢 High |
| 2 | When caught in a loop, stop and restate the problem from scratch with a concrete example of the desired output. | Tier 1 | 🟢 High |
| 3 | Use the "show don't tell" technique: provide a before/after example of exactly what you want changed. | Tier 1 | 🟡 Medium |

---

### B6 — Depth Miscalibration

**Root cause:**
The AI infers the expected depth from cues in the prompt. Short, casual prompts suggest a quick answer; detailed technical prompts suggest depth. When the cue is ambiguous or the developer's mental model of their own question is unclear, the AI guesses wrong.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Explicitly state the desired output format: *"Give me a 2-paragraph high-level explanation"* or *"Give me production-ready code with full error handling."* | Tier 1 | 🟢 High |
| 2 | If a response is too detailed: *"Too much — give me the 30-second version."* If too shallow: *"Go deeper, I need the implementation details."* | Tier 1 | 🟡 Medium |

---

### B7 — Single Solution Tunnel Vision

**Root cause:**
By default, AI produces one answer — the most statistically probable one given its training. Generating alternatives requires explicit prompting. This is not a model limitation per se; it is a default behavior that developers rarely override.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Ask explicitly: *"Give me 2–3 approaches with tradeoffs. Then recommend one and explain why."* | Tier 1 | 🟢 High |
| 2 | For architectural decisions, always ask for a comparison table before committing to an approach. | Tier 1 | 🟢 High |

---

## Category C — Trust & Reliability

### Why This Category Exists

This is the most dangerous category because the failures are invisible until they cause damage. The root causes are deeply embedded in how current AI models are trained. RLHF optimization produces models that sound confident and agreeable because raters historically reward those qualities. The result is a model that is systematically overconfident and systematically sycophantic — two failure modes that directly erode developer trust.

**Primary fault:** 🤖 AI (training incentives) — these are hard to fix from the developer side alone.

---

### C1 — Hallucinated APIs

**Root cause:**
Language models generate token sequences that are statistically plausible given their training distribution. Method names, package names, and config keys that "should exist" based on naming patterns will be generated confidently — even if they don't. The model has no runtime validation step. It cannot distinguish "I know this exists" from "this pattern fits, so I'll generate it."

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Never copy-paste AI-generated code involving unfamiliar APIs without verifying the method exists in documentation or source. | Tier 1 | 🟢 High |
| 2 | Ask: *"Are all the methods and packages you've referenced real and available in [version X]? Flag any you're uncertain about."* | Tier 1 | 🟡 Medium |
| 3 | Add build-time and import checks to CI. Compilation errors from hallucinated types are caught immediately. | Tier 2 | 🟢 High |
| 4 | Prefer showing the AI an actual interface or class definition from your codebase rather than asking it to call things it learned during training. | Tier 1 | 🟢 High |

---

### C2 — Overconfidence Without Uncertainty Signals

**Root cause:**
RLHF training: human raters consistently score confident-sounding answers higher than hedged ones, even when the hedged answer is more accurate. Over thousands of training examples, the model learns that expressing uncertainty is penalized. The result is a model that presents guesses as facts.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Ask explicitly: *"How confident are you in this? What's the probability this is correct? What could be wrong?"* | Tier 1 | 🟡 Medium |
| 2 | For any non-trivial suggestion, ask: *"What assumptions are you making that I should verify?"* | Tier 1 | 🟢 High |
| 3 | Establish a personal rule: treat any AI claim about version-specific behavior, obscure APIs, or performance characteristics as "verify before trust." | Tier 1 | 🟢 High |
| 4 | Long-term: AI tooling should include calibrated confidence scoring as a first-class output. This is a model/platform improvement. | Tier 3 | 🟢 High (when available) |

---

### C3 — Sycophancy Under Pushback

**Root cause:**
RLHF training produces models that optimize for human approval. When a human expresses displeasure or pushes back, agreeing produces a better short-term approval signal than holding firm. Over many training iterations, the model learns to capitulate. This is one of the most well-documented alignment problems in current LLMs.

**Why it's dangerous:** The developer may be wrong. An AI that always agrees stops being a useful second opinion and becomes a yes-machine that amplifies mistakes.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | When you disagree with an AI suggestion, ask: *"I'm pushing back — but are you still confident in your original answer? Explain your reasoning again before changing it."* | Tier 1 | 🟢 High |
| 2 | Explicitly invite disagreement: *"If you think I'm wrong, say so and tell me why. Don't just agree with me."* | Tier 1 | 🟡 Medium |
| 3 | Treat sudden AI agreement after your pushback as a yellow flag. Re-examine both positions independently. | Tier 1 | 🟢 High |
| 4 | Long-term: models need to be explicitly trained to maintain calibrated positions under social pressure. This is an active area of alignment research. | Tier 3 | 🟢 High (when available) |

---

### C4 — Truncated Output Without Warning

**Root cause:**
Context window and max token limits are hard cutoffs with no graceful degradation. The model generates tokens until it hits the limit and stops. There is no built-in mechanism to detect that a code block or file is incomplete and warn the developer.

**Fault:** ⚙️ Systemic (tool limitation) + 👨‍💻 Dev (not detecting it)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Always check that generated code files end with a proper closing brace/bracket and that no obvious sections are missing. | Tier 1 | 🟢 High |
| 2 | For long files, ask the AI to generate in sections: *"Generate the first half, then I'll ask for the second half."* | Tier 1 | 🟢 High |
| 3 | After receiving a long response: *"Did you complete the full output or was it cut off?"* | Tier 1 | 🟡 Medium |

---

### C5 — No Capability Boundaries

**Root cause:**
AI doesn't have a reliable self-model of what it can and can't do well. It cannot accurately predict "this task exceeds my reliable ability." The same overconfidence issue from C2 applies to task scope — the model will attempt a 2000-file refactor with the same confidence as a 10-line function.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | For complex tasks, ask upfront: *"What parts of this task are you confident about? What parts are risky or uncertain?"* | Tier 1 | 🟡 Medium |
| 2 | Decompose large tasks yourself before handing to AI. Never give a vague large task and hope for the best. | Tier 1 | 🟢 High |
| 3 | Treat any AI output on a large-scope task as a draft that requires full human review — never as a finished product. | Tier 1 | 🟢 High |

---

### C6 — No Alternative Suggestions

**Root cause:**
Generating alternatives requires extra computation and is not rewarded in default training. The AI produces the most likely answer and stops. It doesn't self-critique unless explicitly asked to do so.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | After receiving a solution, ask: *"What are the weaknesses of this approach? Is there a better way?"* | Tier 1 | 🟢 High |
| 2 | Make alternative-seeking a default step in your workflow for any non-trivial design decision. | Tier 1 | 🟢 High |

---

### C7 — Training Data vs. Actual Understanding

**Root cause:**
The AI has no live access to your codebase at inference time (unless explicitly provided). When it produces output that seems codebase-aware, it is pattern-matching from training — not reasoning about your actual code. This creates an illusion of understanding that can be misleading, especially for domain-specific or highly customized systems.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Always provide the actual code as context rather than describing it. Never assume the AI "knows" your codebase from prior interaction. | Tier 1 | 🟢 High |
| 2 | Test AI understanding: *"In your own words, describe how [this module] works based on what I've shown you."* Verify before trusting generated code. | Tier 1 | 🟡 Medium |
| 3 | Long-term: tools like MCP with live code indexing (e.g., code search, symbol resolution) will give AI genuine codebase awareness. | Tier 3 | 🟢 High (when available) |

---

## Category D — Code Quality

### Why This Category Exists

AI is trained to generate code that **looks correct** — well-structured, syntactically valid, following known patterns. It is not trained to generate code that **fits** your specific codebase, performance profile, or operational context. The gap between "looks right" and "is right for here" is where most code quality failures live.

**Primary fault:** 🤖 AI (optimizes for plausibility, not fit) + 👨‍💻 Dev (scope not constrained)

---

### D1 — Over-Refactoring

**Root cause:**
AI is trained on "clean code" examples and optimizes for producing output that matches that distribution. When given messy or inconsistent code, its natural output is a tidied version — regardless of whether tidying was asked for. There is no concept of "minimum change" unless explicitly enforced.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include scope constraints in every edit prompt: *"Only change what is necessary to fix the bug. Do not rename, reorganize, or refactor anything else."* | Tier 1 | 🟢 High |
| 2 | Ask for a diff preview first: *"Describe what you would change before writing any code."* Approve the scope before executing. | Tier 1 | 🟢 High |
| 3 | Add a rule to your context file: *"Never rename variables, reorganize functions, or refactor logic unless explicitly asked."* | Tier 2 | 🟢 High |
| 4 | Review diffs by line count. If a bug fix produces 50+ changed lines, treat it as a red flag. | Tier 1 | 🟡 Medium |

---

### D2 — Happy-Path-Only Code

**Root cause:**
Training data is dominated by illustrative code examples that demonstrate the concept being taught — not production-hardened code with full error handling. The model generates code in the style of its training distribution, which skews toward readable demos rather than defensive production code.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Always append to generation prompts: *"Include full error handling, null checks, and edge cases. Do not skip defensive code."* | Tier 1 | 🟢 High |
| 2 | Add a standard checklist to your context file: for every generated function, AI must handle null inputs, empty collections, and failure cases. | Tier 2 | 🟢 High |
| 3 | After generation, ask: *"What edge cases does this code not handle? List them."* Then decide which to address. | Tier 1 | 🟢 High |

---

### D3 — Code Duplication Instead of Reuse

**Root cause:**
AI generates code from its training distribution, not from a live map of your codebase. It doesn't know that `OrderCalculationHelper.cs` exists unless you tell it. From its perspective, generating a new utility is the path of least resistance.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Before asking AI to implement anything, ask: *"Given what I've shown you of this codebase, does a utility or service for this already exist?"* | Tier 1 | 🟢 High |
| 2 | Maintain a "key utilities" section in your context file listing important shared helpers and their locations. | Tier 2 | 🟢 High |
| 3 | Lint for duplicated logic patterns as part of code review. Tools like SonarQube can catch this automatically. | Tier 3 | 🟡 Medium |

---

### D4 — N+1 Query Generation

**Root cause:**
AI generates structurally correct ORM code. It doesn't have a query execution planner or runtime profile — it can't "see" that iterating a collection and calling a navigation property inside the loop will issue N database queries. This requires knowledge of how the ORM executes, which the AI has in theory but fails to apply consistently.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add to context file: *"Always use `.Include()` / eager loading for navigation properties. Never access navigation properties inside a loop without prior eager loading."* | Tier 2 | 🟢 High |
| 2 | After any data-access code is generated, ask: *"Does this code have any N+1 query risk? How would you fix it if so?"* | Tier 1 | 🟢 High |
| 3 | Enable EF Core query logging in development and review slow queries as part of the PR process. | Tier 2 | 🟢 High |

---

### D5 — Blocking Calls in Async Context

**Root cause:**
AI is trained on a mix of sync and async code examples. When generating code for an async method, it sometimes falls back to sync patterns — especially when the called method doesn't have a clear async counterpart in its training data, or when it's "patching in" a call without considering the surrounding async context.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add to context file: *"Never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` in async code. Always await properly."* | Tier 2 | 🟢 High |
| 2 | Add a custom Roslyn analyzer or StyleCop rule that flags `.Result` and `.Wait()` usage. Makes it a CI-enforced rule. | Tier 3 | 🟢 High |
| 3 | After async code generation, search the output for `.Result` and `.Wait()` before accepting it. | Tier 1 | 🟡 Medium |

---

### D6 — Premature Abstraction (YAGNI Violations)

**Root cause:**
AI is trained on "good code" examples which often demonstrate design patterns and abstractions. It defaults toward "architecturally impressive" code because that's what the training distribution rewards. The YAGNI principle is harder to internalize because it requires knowing what will *not* be needed — which the AI cannot know without project context.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Constrain explicitly: *"Write the simplest solution that works. Do not add interfaces, abstract classes, or factory patterns unless I ask."* | Tier 1 | 🟢 High |
| 2 | Add a YAGNI rule to context file: *"Do not introduce abstractions for single implementations. No interfaces unless there are or will be multiple concrete implementations."* | Tier 2 | 🟢 High |
| 3 | Review generated code for single-use interfaces and unused extension points before accepting. | Tier 1 | 🟡 Medium |

---

### D7 — Breaking Public API Without Warning

**Root cause:**
AI focuses on making the code work correctly. Breaking API changes are a deployment and integration concern — not a correctness concern from the model's perspective. It has no visibility into who calls the method being changed.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Before any refactor involving public methods: *"List all the method signatures you plan to change. Flag which ones are public API."* | Tier 1 | 🟢 High |
| 2 | Add to context file: *"Never change public method signatures without explicitly flagging it and proposing a backward-compatible alternative."* | Tier 2 | 🟢 High |
| 3 | Use API compatibility tools (e.g., `Microsoft.DotNet.ApiCompat`) in CI to catch breaking changes automatically. | Tier 3 | 🟢 High |

---

### D8 — Inconsistent Error Handling

**Root cause:**
AI has no visibility into your project's error handling strategy unless explicitly provided. It generates error handling that is locally correct but may use different exception types, wrapping patterns, or logging approaches than the rest of the codebase.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include your error handling strategy in the context file: custom exception types, wrapping conventions, how errors propagate. | Tier 2 | 🟢 High |
| 2 | Provide a reference example: *"Follow the same error handling pattern as in [ExistingHandler.cs]."* | Tier 1 | 🟢 High |
| 3 | Add exception type conventions to your linting rules. | Tier 3 | 🟡 Medium |

---

### D9 — Poor Logging Quality

**Root cause:**
Logging quality is highly context-dependent — what to log, at what level, and with what structured data depends on your observability stack, on-call practices, and debugging history. The AI has none of this context. It defaults to either no logging (if the examples in its training didn't include it) or verbose logging (if it errs toward completeness).

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include logging conventions in context file: log levels, required structured fields, what should/shouldn't be logged. | Tier 2 | 🟢 High |
| 2 | After generation: *"Review the logging in this code. Is it sufficient for debugging in production? Are log levels appropriate?"* | Tier 1 | 🟡 Medium |
| 3 | Add a Datadog/structured logging template to your context file as a reference example. | Tier 2 | 🟢 High |

---

### D10 — Useless Code Comments

**Root cause:**
AI is trained to produce commented code because commented code looks more thorough and educational in training examples. The resulting comments describe *what* — which is already visible in the code — rather than *why*, which is only knowable from the developer's intent and business context.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add to context file: *"Only add comments to explain WHY, not what. Do not add comments that restate what the code does."* | Tier 2 | 🟢 High |
| 2 | After generation, scan comments and delete any that start with the method/variable name or restate the operation. | Tier 1 | 🟡 Medium |

---

## Category E — Security & Safety

### Why This Category Exists

Security is a cross-cutting concern that requires active awareness at every line of code. AI training data contains an enormous amount of example code that is insecure by construction — tutorials, Stack Overflow answers, demo apps. The model learned from this data without security as a primary optimization target. The result is that security considerations are not first-class outputs unless explicitly demanded.

**Primary fault:** 🤖 AI (training data quality) + 👨‍💻 Dev (not requiring security review) + ⚙️ Systemic (no default security analysis step)

---

### E1 — Unwarned Security Vulnerabilities

**Root cause:**
Security analysis requires reasoning about how data flows through a system, what it is used for, and what an attacker might exploit. This is a separate reasoning task from "generate correct code." Without explicit prompting, AI treats code generation and security analysis as independent tasks and only does the former.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add a mandatory security review step to every code generation: *"After generating this code, review it for SQL injection, XSS, path traversal, and input validation issues."* | Tier 1 | 🟢 High |
| 2 | Add a security checklist section to your context file with the specific vulnerability classes relevant to your stack. | Tier 2 | 🟢 High |
| 3 | Run SAST tools (SonarQube, Semgrep, CodeQL) on all AI-generated code as a CI gate. | Tier 3 | 🟢 High |
| 4 | Frame security as a first-class requirement in prompts: *"This code handles user-supplied input. Treat all inputs as untrusted."* | Tier 1 | 🟢 High |

---

### E2 — Hardcoded Credentials in Examples

**Root cause:**
Training data is full of tutorial code with literal credentials for readability. The model learned that `apiKey = "sk-..."` is a common and accepted pattern in code examples. It doesn't consistently distinguish between "example code" and "production code" contexts.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add to context file: *"Never hardcode API keys, passwords, or secrets. Always use environment variables or a secrets manager."* | Tier 2 | 🟢 High |
| 2 | Run secret scanning tools (e.g., `git-secrets`, `trufflehog`, GitHub secret scanning) on all commits. | Tier 3 | 🟢 High |
| 3 | After any AI-generated config or auth code, grep for literal strings that look like credentials before committing. | Tier 1 | 🟢 High |

---

### E3 — No Warning Before Irreversible Actions

**Root cause:**
In agentic mode, AI optimizes for task completion. Irreversibility is a human concern — the model has no visceral sense of "this cannot be undone." Unless trained or instructed to treat certain operations as requiring explicit confirmation, it will execute them as part of the task flow.

**Fault:** ⚙️ Systemic (agentic tooling gap) + 🤖 AI (no built-in caution for destructive ops)

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | In agentic mode, configure explicit stop conditions: *"Before executing any delete, drop, overwrite, or migration operation, stop and ask for confirmation."* | Tier 1 | 🟢 High |
| 2 | Restrict agentic tool permissions to read-only for exploration tasks. Only enable write/execute permissions for implementation tasks. | Tier 2 | 🟢 High |
| 3 | Use dry-run / preview mode for all migration and data operations before executing. | Tier 1 | 🟢 High |
| 4 | Long-term: AI agentic frameworks need a built-in "reversibility check" before executing any action. | Tier 3 | 🟢 High (when available) |

---

### E4 — Insecure Data Storage Suggestions

**Root cause:**
AI suggests the simplest working solution for persistence unless told otherwise. Plaintext config files and application logs are the simplest options. The additional security reasoning required to recommend secrets managers or encrypted storage requires explicit context about what the data is and why it's sensitive.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add data classification to your context file: what types of data exist and how they must be stored. | Tier 2 | 🟢 High |
| 2 | Whenever generating any persistence code: *"This data is [PII/credential/sensitive]. Recommend appropriate storage."* | Tier 1 | 🟢 High |

---

### E5 — Unsanitized User Input

**Root cause:**
Same as E1 — security reasoning is separate from code generation. Input validation is often omitted from training examples because it adds verbosity without illustrating the concept being taught.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Always flag user-input entry points in your prompt: *"This parameter comes from user input. Validate and sanitize accordingly."* | Tier 1 | 🟢 High |
| 2 | Add to context file: *"Treat all external inputs (HTTP, file, user) as untrusted. Validate before use."* | Tier 2 | 🟢 High |
| 3 | SAST tools catch many unsanitized input patterns — enforce in CI. | Tier 3 | 🟢 High |

---

## Category F — Codebase Awareness

### Why This Category Exists

The AI has no persistent, live model of your codebase. It works from whatever has been explicitly provided in the context window — a snapshot, not a living understanding. Every suggestion it makes about architecture, dependencies, or constraints is a projection from its training distribution onto the partial picture you've given it.

**Primary fault:** ⚙️ Systemic (no live codebase model) + 👨‍💻 Dev (not providing enough context)

---

### F1 — Ignoring Real-World Constraints

**Root cause:**
AI proposes solutions that are correct in isolation. It doesn't know about your 10-year-old legacy dependency, your performance SLA, your compliance requirement, or the three other teams whose services depend on this API. These constraints are invisible unless the developer explicitly states them.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include a constraints section in your context file: known technical debt, third-party limitations, performance budgets, compliance rules. | Tier 2 | 🟢 High |
| 2 | State constraints directly in the prompt: *"We cannot change the database schema. The solution must work within the existing table structure."* | Tier 1 | 🟢 High |
| 3 | After receiving a proposal: *"What assumptions did you make about our system? List them so I can validate."* | Tier 1 | 🟡 Medium |

---

### F2 — Undetected Circular Dependencies

**Root cause:**
AI generates code file-by-file or module-by-module without maintaining a full dependency graph. Circular references are a system-level concern that requires seeing the whole picture simultaneously — which the AI typically doesn't have.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | For architectural tasks, provide a dependency diagram or module map as context. | Tier 1 | 🟢 High |
| 2 | After design proposals: *"Does this design introduce any circular dependencies between modules?"* | Tier 1 | 🟡 Medium |
| 3 | Use static analysis tools (NDepend, dotnet-depends) to detect circular dependencies automatically in CI. | Tier 3 | 🟢 High |

---

### F3 — No Side-Effect Warnings

**Root cause:**
AI analyzes the code it is shown, not the code it hasn't been shown. Ripple effects on other modules require knowing the callers, consumers, and integration points — which are usually not in the AI's context window.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Before any change: *"What other parts of the system might be affected by this change? List them even if you haven't seen their code."* | Tier 1 | 🟢 High |
| 2 | Provide caller context alongside the file being changed. | Tier 1 | 🟢 High |
| 3 | Make dependency and impact analysis part of the standard PR template. | Tier 2 | 🟡 Medium |

---

### F4 — No Migration or Rollback Plan

**Root cause:**
AI generates the target state. Getting from the current state to the target state — safely, in production, with real data — is a deployment engineering concern that the AI doesn't include by default unless asked.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | For any breaking change, explicitly ask: *"Provide a migration plan and a rollback plan alongside this change."* | Tier 1 | 🟢 High |
| 2 | For schema changes, always ask for both the forward migration and the down migration. | Tier 1 | 🟢 High |
| 3 | Add migration planning as a required section in your ADR template. | Tier 2 | 🟡 Medium |

---

### F5 — Requires Human Context AI Cannot Have

**Root cause:**
Some decisions depend on organizational knowledge, team agreements, business strategy, or political context that exists only in human memory. AI will still attempt to answer because it has no mechanism to say "this question requires context I fundamentally cannot have."

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Recognize the category: any question involving "why did we decide X," "what does the product team want," or "what do other teams depend on" requires human input, not AI. | Tier 1 | 🟢 High |
| 2 | Use AI for the technical analysis; use humans for the contextual judgment. Separate the two explicitly in your workflow. | Tier 1 | 🟢 High |

---

### F6 — Pattern Inconsistency

**Root cause:**
AI generates code using patterns from its training distribution. Without seeing the project's existing patterns, it defaults to whichever pattern is most common in its training data for that type of problem.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Include pattern examples in your context file: *"Use this DI registration pattern [example]. Use this command/handler pattern [example]."* | Tier 2 | 🟢 High |
| 2 | Always provide a reference file: *"Follow the same pattern as [ExistingFeature.cs]."* | Tier 1 | 🟢 High |

---

## Category G — Testing

### Why This Category Exists

AI generates tests the same way it generates production code: by matching training distribution patterns. Most test examples in training data are simple, illustrative unit tests — not the kind of comprehensive, behavior-driven tests that actually catch bugs in production. Coverage-first metrics make this worse by creating an incentive for tests that look thorough.

**Primary fault:** 🤖 AI (trained on low-quality test examples) + 👨‍💻 Dev (not specifying test quality requirements)

---

### G1 — Tests That Test Nothing

**Root cause:**
Training data contains many tests written for illustration or coverage compliance rather than genuine defect detection. The AI learned to produce tests that look like tests — they compile, they have `Assert` calls, they have mock setups — but they verify nothing that would catch a real bug. Over-mocking is particularly common because it produces clean-looking tests that are fully decoupled from reality.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Use behavior-focused prompt framing: *"Write tests that would catch real bugs in this method. Each test should fail if I introduced a specific defect."* | Tier 1 | 🟢 High |
| 2 | Ask: *"For each test you write, describe what specific bug it would catch."* If the AI can't answer, the test is probably vacuous. | Tier 1 | 🟢 High |
| 3 | Add a no-over-mocking rule to context file: *"Do not mock the system under test. Only mock external dependencies."* | Tier 2 | 🟢 High |
| 4 | Use mutation testing tools (Stryker.NET) to verify that tests actually fail when code is broken. | Tier 3 | 🟢 High |

---

### G2 — Missing Negative and Boundary Cases

**Root cause:**
Happy-path tests are the first tests anyone writes and they dominate training data. Negative cases, null inputs, boundary values, and error paths require explicit intention to cover — which the AI lacks by default.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Explicitly request: *"Write tests for: null inputs, empty collections, boundary values, and all error paths."* | Tier 1 | 🟢 High |
| 2 | Use a test coverage checklist in context file: for every generated test suite, it must include at least one null test, one empty collection test, and one error path test. | Tier 2 | 🟢 High |
| 3 | After generation: *"What cases does this test suite not cover?"* | Tier 1 | 🟡 Medium |

---

### G3 — Brittle Hardcoded Assertions

**Root cause:**
Hardcoded values are simpler to generate and make tests look concrete. Using builders, fixtures, and constants requires knowing your project's test infrastructure — which the AI doesn't have unless provided.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Provide your test builder and fixture classes as context. Instruct AI to use them. | Tier 1 | 🟢 High |
| 2 | Add to context file: *"Use test data builders for complex objects. Use named constants for magic values. Never hardcode IDs or magic strings."* | Tier 2 | 🟢 High |

---

### G4 — No Test Re-run After Changes

**Root cause:**
In agentic mode, AI executes tasks to completion. Running tests is a verification step that requires knowing the test suite exists, what tests are relevant, and how to execute them — and then acting on failures. This is not part of the default agentic task loop unless configured.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Explicitly add to agentic task instructions: *"After any code change, run the relevant test suite and fix any failures before reporting the task complete."* | Tier 1 | 🟢 High |
| 2 | Configure Claude Code to always run tests as part of the task completion criteria. | Tier 2 | 🟢 High |

---

## Category H — Debugging & Diagnosis

### Why This Category Exists

Debugging requires a fundamentally different cognitive mode than code generation. It requires forming and eliminating hypotheses, gathering evidence, and resisting the urge to fix things before understanding them. AI is primarily trained as a question-answering and code-generating system — not as a systematic investigator. The result is a model that jumps to plausible-sounding fixes rather than performing proper root cause analysis.

**Primary fault:** 🤖 AI (trained for Q&A, not investigation) + 👨‍💻 Dev (framing requests as "fix this" rather than "help me diagnose")

---

### H1 — Hypothesis Fixation

**Root cause:**
The AI generates the most statistically probable explanation given the symptoms and then optimizes variations around that hypothesis. It doesn't naturally perform a competing-hypotheses analysis. This mirrors a well-known human cognitive bias (anchoring) but for different mechanical reasons.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Explicitly ask for multiple hypotheses: *"Give me 3 possible root causes for this bug, ordered by likelihood. Do not suggest fixes yet."* | Tier 1 | 🟢 High |
| 2 | For complex bugs: *"What evidence would confirm or rule out each hypothesis?"* Forces structured investigation. | Tier 1 | 🟢 High |
| 3 | If stuck in a loop of failed fixes, reset: *"Forget the fixes we've tried. Start from scratch — what could cause this symptom?"* | Tier 1 | 🟢 High |

---

### H2 — Fix Without Diagnosis

**Root cause:**
AI interprets "I have a bug" as "give me a fix" because that's the most common resolution of such statements in its training data. Diagnosis is a prerequisite that developers often skip in how they frame the request.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Reframe the prompt: *"Do not suggest a fix. Help me understand why this is happening."* | Tier 1 | 🟢 High |
| 2 | Use an explicit two-phase debugging protocol: Phase 1 = diagnosis, Phase 2 = fix. Never combine in one prompt. | Tier 1 | 🟢 High |
| 3 | For persistent bugs: *"What additional information would you need to diagnose this with confidence?"* | Tier 1 | 🟡 Medium |

---

### H3 — No Acceptance Criteria Before Starting

**Root cause:**
AI starts executing as soon as it understands what to build. The concept of "what does done look like" is a project management concern — not naturally surfaced in a code generation request unless asked.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Before any implementation task: *"Before writing code, state your understanding of the acceptance criteria and the edge cases you plan to handle."* | Tier 1 | 🟢 High |
| 2 | Add to agentic workflow: plan → criteria agreement → implementation → verification. Never skip criteria. | Tier 2 | 🟢 High |

---

## Category I — Agentic Behavior

### Why This Category Exists

Agentic AI introduces a qualitative shift in failure modes. In chat mode, every AI output passes through the developer's eyes before taking effect. In agentic mode, the AI acts — files are changed, commands are run, APIs are called. Every other failure mode in this document becomes more dangerous when the AI is operating autonomously. The absence of human checkpoints is both the feature and the risk.

**Primary fault:** ⚙️ Systemic (autonomy without sufficient guardrails) + 👨‍💻 Dev (insufficient boundary-setting)

---

### I1 — Unchecked Agentic Loop

**Root cause:**
Agentic AI is optimized for task completion. Without explicit checkpoints, it continues executing until the task is "done" or it gets stuck. There is no built-in concept of "I've made enough changes — I should pause for human review before continuing."

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Set explicit scope constraints upfront: *"Do not edit more than 3 files without pausing for my approval."* | Tier 1 | 🟢 High |
| 2 | Use `--no-auto-accept` / interactive mode for any task touching more than one file. | Tier 2 | 🟢 High |
| 3 | Define task boundaries in terms of outcomes, not open-ended instructions: *"Implement X in [FileA.cs] only."* | Tier 1 | 🟢 High |
| 4 | Long-term: agentic tools need a built-in "commit checkpoint" pattern — propose changes, get approval, then execute. | Tier 3 | 🟢 High (when available) |

---

### I2 — No Upfront Task Planning

**Root cause:**
AI begins the most statistically likely next action immediately. Planning is a meta-task that requires withholding the natural output. Without explicit instruction to plan first, the model defaults to generating code because that's what its training rewards.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Make planning mandatory: *"Before writing any code, produce a step-by-step plan. Wait for my approval before executing."* | Tier 1 | 🟢 High |
| 2 | Add a planning phase to your CLAUDE.md: *"For any task involving more than one file, always produce a plan first."* | Tier 2 | 🟢 High |
| 3 | Review and annotate the plan before approving. The plan is your last easy checkpoint before the diff becomes large. | Tier 1 | 🟢 High |

---

### I3 — Stale Line-Number Patching

**Root cause:**
AI reads a file at time T and generates a patch based on that snapshot. If the file is modified between T and when the patch is applied — by the developer or by a previous AI edit — the patch targets wrong line numbers or stale content.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Instruct the AI to re-read each file immediately before patching it: *"Always read the current state of a file before editing it."* | Tier 1 | 🟢 High |
| 2 | Avoid interleaving manual edits with agentic AI edits in the same session. Complete one before starting the other. | Tier 1 | 🟡 Medium |
| 3 | Use content-based patching (str_replace) rather than line-number-based patching wherever possible. | Tier 2 | 🟢 High |

---

### I4 — No Post-Task Summary

**Root cause:**
AI completes the task and stops. Producing a handover summary is an additional output that requires explicit prompting. Without it, the developer inherits a large diff with no explanation of the decisions made.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Add to task instructions: *"When complete, produce a summary of: what changed, why each change was made, and what I should verify."* | Tier 1 | 🟢 High |
| 2 | Use the summary as the basis for the PR description. Kills two birds with one stone. | Tier 1 | 🟢 High |

---

### I5 — Vague Commit Messages and PR Descriptions

**Root cause:**
AI generates commit messages and PR descriptions that summarize what was done at a surface level — matching the style of generic commit messages in its training data. Without knowing your team's conventions, reviewers' needs, or the business context of the change, it produces technically accurate but informationally thin descriptions.

| # | Fix | Tier | Effectiveness |
|---|-----|------|---------------|
| 1 | Provide a commit message template or example in your context file. *"Follow Conventional Commits format: `type(scope): description`."* | Tier 2 | 🟢 High |
| 2 | Provide a PR description template. Ask AI to fill it in: *"Fill in this template with the context of this change: [template]."* | Tier 1 | 🟢 High |
| 3 | Ask for the "why" explicitly: *"Write a PR description that explains why this change was made, not just what changed."* | Tier 1 | 🟢 High |

---

## Cross-Cutting Recommendations

These are fixes that apply across multiple categories simultaneously and deliver the highest leverage per effort invested.

### 1. Invest in a Living Context File (Covers A, B, D, F, G)

A `CLAUDE.md` or equivalent project context file that is fed at the start of every session is the single highest-leverage investment a developer team can make. It should contain:
- Tech stack and version constraints
- Coding conventions and patterns
- Rejected approaches with rationale
- Security and error handling standards
- Key utility locations
- Test conventions

**Effectiveness: 🟢 High. One-time setup, compounding returns.**

### 2. Adopt a Plan-First Workflow for Complex Tasks (Covers B, F, H, I)

For any task touching more than one file or involving design decisions:
1. Ask for a plan — not code.
2. Review and correct the plan.
3. Agree on acceptance criteria.
4. Execute with explicit scope boundaries.

**Effectiveness: 🟢 High. Changes the failure mode from "wrong code at scale" to "wrong plan that's easy to correct."**

### 3. Always Do a Security Pass on AI-Generated Code (Covers E)

Treat AI-generated code as untrusted until reviewed for security. A single prompt appended to every generation — *"Review this for security issues before finalizing"* — catches most E-category failures.

**Effectiveness: 🟢 High. Adds 30 seconds, eliminates a category of critical failures.**

### 4. Use CI as an AI Safety Net (Covers C, D, E, G)

Static analysis, SAST tools, test runners, API compatibility checks, and secret scanners all act as automated review for AI-generated code. They are not a replacement for human review but they catch a class of failures that humans regularly miss under time pressure.

**Effectiveness: 🟢 High (long-term). One-time setup, runs on every PR automatically.**

### 5. Separate Diagnosis From Action (Covers B, H)

The most consistent failure mode in debugging and intent alignment is collapsing "understand the problem" and "solve the problem" into a single prompt. Keeping them as two separate interactions — always — eliminates B1, H1, and H2 entirely.

**Effectiveness: 🟢 High. Zero setup, pure habit change.**

---

*Last updated: April 2026*
