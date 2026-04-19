# CLAUDE.md & Rules — Best Practices

> **Nguồn tài liệu:** Anthropic official documentation ([code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)), [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices), [code.claude.com/docs/en/claude-directory](https://code.claude.com/docs/en/claude-directory), và các tài liệu cộng đồng uy tín (ClaudeFast Rules Directory Guide, GitHub: shanraisshan/claude-code-best-practice).

---

## CLAUDE.md là gì?

`CLAUDE.md` là file markdown cung cấp cho Claude các instructions liên tục xuyên suốt các sessions. Vì mỗi session Claude Code bắt đầu với một context window mới hoàn toàn, CLAUDE.md là cách bạn mang kiến thức sang session tiếp theo — build commands, coding standards, cấu trúc project, và các quy tắc "luôn phải làm X".

> **Anthropic hướng dẫn:** Xem CLAUDE.md là nơi bạn ghi lại những gì bạn phải giải thích lại. Thêm vào khi Claude mắc cùng một lỗi lần thứ hai, hoặc khi bạn gõ cùng một correction vào chat mà bạn đã gõ ở session trước.

CLAUDE.md là **context**, không phải configuration bắt buộc. Claude đọc nó, coi nó là instructions ưu tiên cao, và hành động phù hợp. Instructions càng cụ thể và ngắn gọn, Claude càng tuân theo nhất quán.

---

## Hai hệ thống memory bổ sung cho nhau

Claude Code cung cấp hai hệ thống cho persistent memory. Cả hai đều load ở đầu mỗi session.

| | CLAUDE.md files | Auto memory |
|---|---|---|
| **Ai viết** | Bạn | Claude |
| **Nội dung** | Instructions và rules | Learnings và patterns Claude tự khám phá |
| **Scope** | Project, user, hoặc org | Per working tree |
| **Load vào** | Mỗi session, toàn bộ | Mỗi session (200 dòng đầu hoặc 25 KB) |
| **Dùng cho** | Coding standards, workflows, architecture decisions | Build commands, debugging insights, preferences Claude tự học |

Dùng CLAUDE.md khi bạn muốn hướng dẫn tường minh hành vi của Claude. Auto memory cho phép Claude học từ corrections của bạn mà không cần effort thủ công.

---

## Các vị trí đặt CLAUDE.md

CLAUDE.md có thể tồn tại ở nhiều scope khác nhau. Scope cụ thể hơn có ưu tiên cao hơn.

| Scope | Vị trí | Mục đích | Chia sẻ với |
|---|---|---|---|
| **Managed policy** | System-level (tùy OS) | Standards toàn tổ chức do IT/DevOps quản lý | Tất cả users trong org |
| **Project** | `./CLAUDE.md` hoặc `./.claude/CLAUDE.md` | Instructions chia sẻ trong team | Team qua version control |
| **User** | `~/.claude/CLAUDE.md` | Preferences cá nhân cho tất cả projects | Chỉ bạn |
| **Local** | `./CLAUDE.local.md` (gitignored) | Preferences cá nhân, không commit | Chỉ bạn, không commit |

CLAUDE.md và CLAUDE.local.md được tìm kiếm bằng cách đi ngược lên directory tree từ working directory. Tất cả files tìm được đều được **nối vào context** — không override nhau. Trong cùng thư mục, `CLAUDE.local.md` được append sau `CLAUDE.md`, nên notes cá nhân luôn thắng khi xung đột ở cùng level.

---

## Thư mục `.claude/rules/`

Với các projects lớn hơn, `.claude/rules/` là cách được khuyến nghị để tổ chức instructions thành các files modular, dễ bảo trì thay vì một CLAUDE.md khổng lồ.

```
.claude/
├── CLAUDE.md             # Instructions toàn cục — luôn load
└── rules/
    ├── code-style.md     # Load mọi lúc (không có paths frontmatter)
    ├── testing.md        # Load mọi lúc
    ├── security.md       # Load mọi lúc
    └── api/
        └── endpoints.md  # Chỉ load khi làm việc với files matching
```

Mọi file `.md` trong `.claude/rules/` được tự động tìm thấy — không cần đăng ký thêm. Files load với **cùng mức độ ưu tiên cao như CLAUDE.md**.

### Tại sao Rules tồn tại: Vấn đề của Monolithic CLAUDE.md

Khi tất cả đều nằm trong một file CLAUDE.md, tất cả đều nhận được high-priority attention cùng lúc. React patterns cạnh tranh với API guidelines cạnh tranh với database rules — kể cả khi bạn chỉ đang làm việc với migration files.

> **Phát hiện từ cộng đồng chất lượng cao:** High priority ở khắp nơi = không có gì là priority. Khi mọi thứ đều được đánh dấu quan trọng, Claude khó xác định điều gì thực sự liên quan đến task hiện tại. Instructions bị bỏ qua, context trở nên nhiễu, hành vi trở nên khó đoán.

Rules giải quyết điều này bằng cách phân phối high-priority instructions qua các files có mục tiêu cụ thể. API rules vẫn có high priority — nhưng chỉ khi bạn làm việc với API files.

### Path-Specific Rules

Rules có thể được scoped theo file patterns bằng YAML frontmatter. Các rules này **chỉ load khi Claude làm việc với files matching**.

```yaml
---
paths:
  - src/api/**/*.ts
  - src/api/**/*.cs
---

# API Development Rules

- Mọi endpoint phải validate input trước khi xử lý
- Trả về error format nhất quán: `{ "error": { "code": "...", "message": "..." } }`
- Bao gồm structured logging ở entry và exit của mỗi handler
- Không bao giờ expose internal stack traces trong API responses
```

Rules không có `paths` field load vô điều kiện mỗi session, giống như CLAUDE.md.

### Hệ thống ưu tiên

| Nguồn | Ưu tiên | Load khi nào |
|---|---|---|
| **CLAUDE.md** | Cao | Mỗi session, luôn luôn |
| **Rules (không có paths)** | Cao | Mỗi session, luôn luôn |
| **Rules (có paths)** | Cao | Chỉ khi làm việc với matching files |
| **Skills** | Trung bình (on-demand) | Khi được trigger theo relevance hoặc `/skill-name` |
| **Conversation history** | Biến đổi | Tích lũy trong session, giảm sau compaction |
| **File contents (Read tool)** | Thông thường | Context weight bình thường |

---

## ✅ NÊN — Design Guidelines

### 1. Bắt đầu với `/init` và tinh chỉnh dần

Chạy `/init` để generate CLAUDE.md starter dựa trên codebase thực tế của bạn. Claude phân tích build systems, test frameworks, và code patterns để tạo nền tảng vững chắc. Từ đó tinh chỉnh với những instructions Claude không thể tự khám phá.

Đặt `CLAUDE_CODE_NEW_INIT=1` để có interactive multi-phase flow hỏi bạn muốn setup artifacts nào (CLAUDE.md, skills, hooks) và trình bày proposal để review trước khi ghi bất kỳ file nào.

### 2. Giữ CLAUDE.md dưới 200 dòng

> **Official Anthropic guidance:** Mục tiêu dưới 200 dòng mỗi file CLAUDE.md. Files dài hơn tiêu tốn nhiều context hơn và giảm độ tuân thủ. Các rules quan trọng bị lạc trong nhiễu.

Nếu file đang phình to, giải pháp không phải là cắt ngẫu nhiên — mà là **extract procedures ra skills và domain-specific rules ra `.claude/rules/`**.

### 3. Viết instructions cụ thể, có thể kiểm chứng

Instructions mơ hồ bị bỏ qua. Instructions cụ thể được tuân theo.

| ❌ Mơ hồ | ✅ Cụ thể |
|---|---|
| "Format code đúng cách" | "Dùng 2-space indentation, không trailing spaces" |
| "Test thay đổi của bạn" | "Chạy `dotnet test` trước mỗi commit" |
| "Giữ files có tổ chức" | "API handlers nằm trong `src/api/handlers/`" |
| "Xử lý errors đúng cách" | "Dùng `ThrowError()` cho validation failures, không throw raw exceptions" |

### 4. Dùng markdown headers và bullets

Claude scan structure giống như người đọc. Các sections có headers rõ ràng dễ theo dõi hơn các đoạn văn dày đặc. Nhóm các instructions liên quan dưới cùng một header.

```markdown
# Build & Test Commands
- Build: `dotnet build`
- Test: `dotnet test --collect:"XPlat Code Coverage"`
- Format: `dotnet format`

# Code Conventions
- Dùng `var` cho local variables với types hiển nhiên
- Ưu tiên expression-bodied members cho single-line methods
- Không bao giờ dùng `dynamic`
```

### 5. Dùng `@import` syntax cho external files

CLAUDE.md có thể import thêm files bằng cú pháp `@path/to/file`. Imported files được expand và load vào context lúc khởi động. Dùng cách này để reference README, package specs, hoặc shared guides mà không cần duplicate.

```markdown
Xem @README.md cho project overview.
Xem @package.json cho available scripts.

# Git workflow
@docs/git-workflow.md
```

### 6. Dùng path-specific rules cho domain concerns

Extract instructions domain-specific chỉ áp dụng cho một số file types nhất định vào path-targeted rules. Điều này giữ chúng ra khỏi context khi không liên quan.

```yaml
# .claude/rules/security.md
---
paths:
  - src/auth/**/*
  - src/payments/**/*
---

# Security-Critical Code Rules

- Không bao giờ log dữ liệu nhạy cảm: passwords, tokens, card numbers, PII
- Validate mọi inputs tại function boundaries
- Dùng parameterized queries — không bao giờ string interpolation
- Yêu cầu explicit authorization check trước mọi data access
```

### 7. Dùng HTML comments cho maintainer notes

Block-level HTML comments trong CLAUDE.md bị strip trước khi inject vào context của Claude. Dùng chúng để lại notes cho maintainers mà không tốn tokens.

```markdown
<!-- Đã review: 2026-01 — rules cho legacy WPF stack nằm trong rules/wpf-legacy.md -->

# Project Overview
...
```

### 8. Dùng `CLAUDE.local.md` cho preferences cá nhân không commit

Giữ sandbox URLs cá nhân, test data preferences, và local tool paths trong `CLAUDE.local.md`. Thêm vào `.gitignore`. File này load cùng với `CLAUDE.md` ở cùng scope và notes cá nhân thắng khi xung đột.

### 9. Định kỳ audit và prune

Review CLAUDE.md và rules files để tìm:
- **Mâu thuẫn** — nếu hai rules xung đột, Claude có thể chọn ngẫu nhiên
- **Dư thừa** — nếu Claude đã làm đúng mà không cần instruction đó, xóa đi hoặc chuyển thành hook
- **Scope creep** — nếu một entry đã trở thành multi-step procedure, chuyển nó sang skill

### 10. Commit project files vào version control

Project CLAUDE.md (`.claude/CLAUDE.md` hoặc `./CLAUDE.md`) và `.claude/rules/` files nên được commit để cả team cùng hưởng lợi từ cùng một context. Xem chúng như code — review changes, track history, roll back khi cần.

---

## ❌ KHÔNG NÊN — Các lỗi phổ biến

### 1. Không viết CLAUDE.md quá 200 dòng

> **Official Anthropic guidance:** "CLAUDE.md quá dài là anti-pattern. Nếu CLAUDE.md quá dài, Claude bỏ qua một nửa vì các rules quan trọng bị lạc trong nhiễu. Fix: Ruthlessly prune."

Fix đúng là extraction — chuyển procedures sang skills, chuyển domain rules sang `.claude/rules/`.

### 2. Không nhồi tất cả vào một file monolithic

Một CLAUDE.md 400 dòng chứa React patterns, API conventions, DB migration rules, và security requirements lẫn lộn gây ra **priority saturation**. Mọi thứ đều high priority, nên không có gì là priority.

Extract thành rules modular:
```
# Trước: một CLAUDE.md 400 dòng
# Sau:
.claude/
├── CLAUDE.md               # ~50 dòng: chỉ facts toàn cục
└── rules/
    ├── testing.md           # load mọi lúc
    ├── security.md          # load mọi lúc
    └── api-endpoints.md     # paths: src/api/**
```

### 3. Không viết instructions mơ hồ

Instructions mơ hồ được xem như suggestions. Instructions cụ thể, có thể kiểm chứng được xem như rules. "Xử lý errors đúng cách" không có tác dụng gì. "Dùng `ThrowError()` cho 4xx validation failures, không throw raw exceptions" thì có thể hành động được.

### 4. Không đặt multi-step procedures trong CLAUDE.md

CLAUDE.md dành cho facts và rules — context luôn áp dụng, ở session level. Nếu một instruction là multi-step procedure (deploy, chạy coverage, generate migrations), nó thuộc về **skill**. Skills load khi cần; CLAUDE.md load mỗi session dù procedure đó có liên quan hay không.

### 5. Không dùng MUST/CRITICAL/IMPORTANT thay cho specificity

Viết hoa không làm tăng độ tuân thủ một cách đáng kể. Specificity mới có tác dụng. Thay vì `PHẢI LUÔN chạy tests`, viết `Chạy dotnet test trước mỗi commit. Không tiến hành bước tiếp theo nếu tests fail.`

### 6. Không bỏ qua conflicting rules giữa các files

Trong monorepo, CLAUDE.md files từ thư mục của team khác có thể được pick up và tạo conflicts. Dùng `claudeMdExcludes` trong settings để bỏ qua các files không liên quan, và chạy `/memory` để audit những gì thực sự load trong session hiện tại.

### 7. Không duplicate instructions giữa CLAUDE.md và rules files

Nếu một instruction nằm ở cả CLAUDE.md và một rules file, nó tiêu tốn double tokens và có thể tạo ra inconsistencies theo thời gian. Chọn một vị trí duy nhất cho mỗi instruction.

---

## Decision Matrix — CLAUDE.md vs Rules vs Skills

| Loại instruction | Nên đặt ở đâu |
|---|---|
| Build commands, test commands | `CLAUDE.md` |
| Conventions toàn project, luôn áp dụng | `CLAUDE.md` |
| Domain-specific rules cho file types cụ thể | `.claude/rules/` với `paths` frontmatter |
| Rules theo topic nhưng không cần path scope | `.claude/rules/` (không có frontmatter) |
| Multi-step procedures (deploy, migrate, review) | Skills |
| Tài liệu tham khảo ít dùng | Skills (load on demand) |
| Preferences cá nhân không chia sẻ với team | `CLAUDE.local.md` |
| Preferences cá nhân cho tất cả projects | `~/.claude/CLAUDE.md` |

---

## Common Patterns

### Pattern 1: CLAUDE.md Gọn Nhẹ + Rules Cho Domain Concerns

```markdown
# CLAUDE.md (~50 dòng)

## Commands
- Build: `dotnet build`
- Test: `dotnet test`
- Lint: `dotnet format --verify-no-changes`

## Architecture
- Dual-stack: .NET Framework 4.8 (legacy) và .NET 8+ (new API)
- Tất cả features mới vào layer .NET 8+
- Không thêm code mới vào layer .NET Framework 4.8

## Source Control
- Không commit trực tiếp vào main
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
- Dùng FastEndpoints conventions cho endpoints mới
- Annotate route params với [FromRoute] khi client dùng Orval-generated TypeScript
- Dùng ThrowError() cho validation failures, không throw raw exceptions
- Bao gồm Datadog structured logging ở entry và exit
```

```yaml
# .claude/rules/payments.md
---
paths:
  - src/**/Payments/**/*
  - src/**/Billing/**/*
---

# Payment Code Rules
- Không bao giờ log card numbers, tokens, hoặc PII
- Stripe operations phải idempotent — luôn truyền idempotency key
- Mọi thay đổi payment state phải emit domain event qua MassTransit
```

### Pattern 2: Import AGENTS.md cho Multi-Tool Repos

Nếu repo đã dùng `AGENTS.md` cho các AI tools khác, import nó từ CLAUDE.md thay vì duplicate:

```markdown
# CLAUDE.md
@AGENTS.md

## Claude Code Specific
- Dùng plan mode trước khi thay đổi bất kỳ thứ gì trong `src/billing/`
- Ưu tiên subagents cho research tasks đọc 10+ files
```

---

## Các lệnh hữu ích

| Lệnh | Hiển thị gì |
|---|---|
| `/init` | Generate CLAUDE.md starter từ phân tích codebase |
| `/memory` | CLAUDE.md và rules files nào đã load + auto-memory entries |
| `/context` | Token usage breakdown: system prompt, memory files, skills, messages |
| `/compact` | Tóm tắt conversation để giải phóng context space |

---

## Checklist trước khi thêm instruction mới

```
□ Đây là fact hoặc rule áp dụng mọi session? → CLAUDE.md hoặc rules/
□ Chỉ áp dụng cho file types cụ thể? → rules/ với paths frontmatter
□ Đây là multi-step procedure? → Skill
□ Là preferences cá nhân không chia sẻ? → CLAUDE.local.md
□ CLAUDE.md đã quá 200 dòng? → Extract trước, rồi mới thêm
□ Đã có instruction tương tự chưa? → Cập nhật existing, không duplicate
□ Đủ cụ thể để kiểm chứng? ("Chạy dotnet test" vs "test thay đổi của bạn")
□ Xóa nó đi thì hành vi Claude có thay đổi không? Nếu không — xóa đi
```

---

## Tài liệu tham khảo

- [Anthropic — How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Anthropic — Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic — Explore the .claude directory](https://code.claude.com/docs/en/claude-directory)
- [ClaudeFast — Rules Directory: Modular Instructions That Scale](https://claudefa.st/blog/guide/mechanics/rules-directory)
- [GitHub — shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
