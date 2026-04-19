# Claude Code Skills — Best Practices

> **Nguồn tài liệu:** Anthropic official documentation ([code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)), Anthropic API Docs ([platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)), Anthropic Engineering blog ([anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)), và các tài liệu cộng đồng uy tín.

---

## Skill là gì?

Một skill là một thư mục chứa file `SKILL.md` — bao gồm instructions, scripts, và resources có tổ chức giúp Claude có thêm capabilities. Skills mở rộng những gì Claude có thể làm mà không cần bạn phải paste lại cùng một playbook vào mỗi cuộc hội thoại.

> **Anthropic dùng ví dụ:** Xây dựng một skill cho agent giống như soạn tài liệu onboarding cho nhân viên mới. Thay vì xây dựng các agent riêng biệt cho từng use case, bạn có thể chuyên biệt hóa agent của mình bằng các composable capabilities bằng cách gói gọn và chia sẻ procedural knowledge.

### Cơ chế tải của Skills — Progressive Disclosure

Đây là nguyên tắc thiết kế cốt lõi giúp skills vừa linh hoạt vừa tiết kiệm token:

| Level | Nội dung tải | Khi nào |
|---|---|---|
| **Level 1** — Metadata | `name` + `description` từ frontmatter | Khi khởi động session, luôn luôn |
| **Level 2** — Instructions | Toàn bộ nội dung `SKILL.md` | Khi Claude quyết định skill này liên quan đến task |
| **Level 3** — Supporting files | `reference.md`, scripts, examples | Chỉ khi cần cho task cụ thể |

Điều này có nghĩa là bạn có thể cài nhiều skills mà không tốn context — Claude chỉ biết mỗi skill tồn tại và khi nào dùng nó. Nội dung thực sự chỉ load khi cần.

---

## Skills vs. Commands vs. Subagents

Hiểu rõ skills nằm ở đâu trong hệ sinh thái Claude Code để tránh dùng sai.

### Skills vs. Commands

Custom commands (`.claude/commands/`) đã được **gộp vào skills**. Một file `.claude/commands/deploy.md` và một skill `.claude/skills/deploy/SKILL.md` đều tạo ra `/deploy` và hoạt động như nhau. Skills là hướng được khuyến nghị vì hỗ trợ thêm: supporting files, auto-invocation control, và subagent execution.

### Skills vs. Subagents

| | Skills | Subagents |
|---|---|---|
| **Context** | Chạy inline trong main conversation | Context window riêng biệt |
| **Lịch sử hội thoại** | Có đầy đủ | Không có — fresh context |
| **Output** | Thêm vào main thread | Chỉ trả về summary |
| **Phù hợp cho** | Reference knowledge, reusable workflows, conventions | Heavy tasks, parallel work, verbose output |
| **Có thể spawn task khác?** | Không | Không (subagents không thể spawn subagents) |

**Dùng Skill khi** bạn muốn instructions tái sử dụng chạy trong context hội thoại hiện tại.
**Dùng Subagent khi** một task sẽ làm ngập main context với logs, file contents, hoặc output mà bạn không cần reference lại.

> **Lưu ý:** Hai tính năng này bổ sung cho nhau, không cạnh tranh. Bạn có thể preload skills vào context của subagent bằng `skills:` frontmatter field, và bạn có thể chạy một skill trong subagent bằng `context: fork`.

---

## Cấu trúc thư mục Skill

```
.claude/skills/
└── my-skill/
    ├── SKILL.md           # Instructions chính (bắt buộc)
    ├── reference.md       # Tài liệu tham khảo chi tiết, load khi cần
    ├── examples/
    │   └── sample.md      # Ví dụ output mẫu
    └── scripts/
        └── validate.sh    # Script thực thi Claude có thể chạy
```

`SKILL.md` là file duy nhất bắt buộc. Supporting files là optional và giúp giữ core skill gọn nhẹ trong khi vẫn bundle được tài liệu chi tiết — chỉ load khi thực sự cần.

---

## Hai loại Skills

### 1. Capability Uplift Skills

Cung cấp cho Claude những khả năng nó không có sẵn — tạo document, browser automation, web scraping, xử lý PDF, các CLI workflow đặc thù.

```yaml
---
name: processing-pdfs
description: Trích xuất text và bảng từ PDF, điền form, merge documents.
  Dùng khi làm việc với file PDF hoặc khi user đề cập PDFs, forms, hoặc document extraction.
---

Dùng pdfplumber để extract text:

```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

Để điền form, xem [forms.md](forms.md).
Để extract nâng cao, xem [reference.md](reference.md).
```

### 2. Encoded Preference Skills

Hướng dẫn Claude theo đúng workflow của team cho những thứ Claude đã biết cách làm — NDA review, commit message format, API design conventions, incident report template.

```yaml
---
name: writing-fastendpoints
description: Conventions cho FastEndpoints endpoints trong codebase này.
  Dùng khi viết endpoint mới hoặc review API changes.
---

Khi viết FastEndpoints endpoint:
- Dùng class cho Request DTOs, không dùng record (tránh source generator fallback)
- Annotate route params với [FromRoute] khi Orval-generated client tách route/body
- Dùng ThrowError() cho validation failures (4xx), không throw exception thô
- Log structured với Datadog tại entry và exit của mỗi endpoint
```

---

## Frontmatter Reference

```yaml
---
name: my-skill                       # Bắt buộc. Chữ thường, gạch ngang, tối đa 64 ký tự
description: Skill này làm gì        # Rất nên có. Tối đa 1024 ký tự
when_to_use: Context trigger bổ sung # Tùy chọn. Mở rộng description
disable-model-invocation: true       # Tùy chọn. Ngăn auto-invocation — chỉ dùng /skill-name
allowed-tools: Read Grep             # Tùy chọn. Pre-approve tools cho skill này
context: fork                        # Tùy chọn. Chạy trong isolated subagent context
---
```

### Giải thích các field quan trọng

**`description`** — Field quan trọng nhất. Claude dùng field này để quyết định khi nào auto-invoke. Đặt use case chính lên đầu. Combined text của `description` + `when_to_use` bị truncate ở 1.536 ký tự trong skill listing.

**`disable-model-invocation: true`** — Dùng cho action skills (deploy, migrate, release) chỉ nên chạy khi bạn gõ `/skill-name` tường minh. Ngăn auto-invocation ngẫu nhiên.

**`context: fork`** — Chạy skill trong isolated subagent context. Nội dung skill trở thành prompt của subagent đó. Subagent **không** có quyền truy cập conversation history. Dùng cho phân tích phức tạp tạo ra verbose output mà bạn không cần trong main thread.

**`allowed-tools`** — Pre-approve các tools cụ thể để Claude không hỏi permission giữa chừng khi skill đang chạy.

---

## ✅ NÊN — Design Guidelines

### 1. Viết ngắn gọn — tin vào intelligence của Claude

Chỉ thêm context mà Claude chưa có. Tự hỏi với từng đoạn: "Claude có thực sự cần giải thích này không? Tôi có thể assume Claude đã biết không?"

```yaml
# ✅ Tốt — ~50 tokens, assume Claude biết PDF là gì
## Extract PDF text

Dùng pdfplumber:
```python
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

# ❌ Xấu — ~150 tokens, giải thích những thứ hiển nhiên
## Extract PDF text

PDF (Portable Document Format) là định dạng file phổ biến chứa text và hình ảnh.
Để extract text từ PDF, bạn cần dùng thư viện. Có nhiều thư viện khác nhau nhưng
pdfplumber được khuyến nghị vì dễ dùng và xử lý được hầu hết các trường hợp...
```

### 2. Match mức độ specificity với độ rủi ro của task

Dùng mô hình **degree of freedom** từ official Anthropic best practices:

| Loại task | Mức freedom | Áp dụng |
|---|---|---|
| Code review, phân tích mở | Cao — natural language guidance | Context quyết định cách tiếp cận tốt nhất |
| Tạo report, output có template | Trung bình — pseudocode với parameters | Pattern ưu tiên tồn tại nhưng cho phép biến thể |
| Database migration, deployment, release | Thấp — exact scripts, không được lệch | Consistency là quan trọng; lỗi gây hậu quả nghiêm trọng |

> **Ví dụ từ Anthropic:** Claude như một robot trên đường. Trên **cây cầu hẹp với vực hai bên** — đưa ra chỉ dẫn chính xác (low freedom). Trên **cánh đồng trống** — chỉ hướng tổng quát và để Claude tự tìm đường tốt nhất (high freedom).

### 3. Đặt tên theo dạng gerund (động từ + -ing)

```
# ✅ Ưu tiên — gerund form
processing-pdfs
reviewing-pull-requests
analyzing-connection-pool
writing-ef-migrations
migrating-mstest-to-xunit

# ✅ Chấp nhận được
pdf-processing
pr-review

# ❌ Tránh
helper, utils, tools     # Quá mơ hồ
documents, data          # Quá generic
anthropic-helper         # Reserved words
```

### 4. Luôn thêm `disable-model-invocation: true` cho action skills

Skills kích hoạt operations (deploy, migrate, release) không bao giờ được tự động chạy. Bắt buộc phải gọi tường minh.

```yaml
---
name: deploy-staging
description: Deploy nhánh hiện tại lên staging environment
disable-model-invocation: true
---

Deploy lên staging:
1. Chạy test suite — `dotnet test`
2. Build Docker image — `docker build -t app:staging .`
3. Push và deploy — `./scripts/deploy.sh staging`
```

### 5. Tách nội dung lớn thành supporting files

Khi `SKILL.md` trở nên cồng kềnh, chuyển các section vào file riêng và reference chúng. Claude đọc supporting files khi cần — không tốn token cho đến khi thực sự sử dụng.

```markdown
# Trong SKILL.md
Skill này xử lý EF Core patterns. Xem query optimization tại [query-patterns.md](query-patterns.md).
Xem migration conventions tại [migrations.md](migrations.md).
```

### 6. Dùng `context: fork` cho phân tích tạo nhiều output

Khi output của skill sẽ làm ngập main conversation với logs, traces, hay file contents, hãy chạy nó trong isolated context.

```yaml
---
name: analyzing-connection-pool
description: Phân tích usage của connection pool trên toàn codebase
context: fork
disable-model-invocation: true
---

Phân tích tất cả chỗ DbContext được instantiate. Tìm kiếm:
- Patterns dùng Parallel.ForEach với service locator
- Thiếu using statements
- Singleton-scoped DbContext registrations

Trả về danh sách ưu tiên các findings với file paths và line numbers.
```

### 7. Iterate với Claude để khám phá context thực sự cần thiết

> **Hướng dẫn từ Anthropic Engineering:** Khi làm việc với Claude, hãy yêu cầu Claude ghi lại các cách tiếp cận thành công và lỗi thường gặp vào skill. Nếu Claude đi lạc hướng khi dùng skill, hãy yêu cầu nó tự phân tích nguyên nhân. Cách này giúp bạn khám phá context Claude thực sự cần thay vì cố đoán trước.

### 8. Commit project skills vào version control

Project skills (`.claude/skills/`) nên được commit vào repository để cả team cùng dùng. Xem chúng như code — review changes, iterate, và cải thiện liên tục.

---

## ❌ KHÔNG NÊN — Các lỗi phổ biến

### 1. Không giải thích những thứ Claude đã biết

Mỗi token trong `SKILL.md` cạnh tranh với conversation history khi skill được load. Skills không phải documentation — chúng là những bổ sung có mục tiêu vào context của Claude.

### 2. Không đặt action skills mà thiếu `disable-model-invocation`

Một deployment hoặc migration skill tự kích hoạt dựa trên keyword match là nguy hiểm. Luôn thêm `disable-model-invocation: true` cho bất kỳ skill nào thực hiện operations không thể hoàn tác.

### 3. Không nhồi tất cả vào một `SKILL.md` khổng lồ

Dùng supporting files và progressive disclosure. Một `SKILL.md` chứa mọi edge case, mọi ví dụ, mọi tài liệu tham khảo sẽ load tất cả vào context mỗi lần — phá vỡ mục đích của progressive disclosure.

### 4. Không dùng `context: fork` cho reference/convention skills

`context: fork` tạo subagent **không có quyền truy cập conversation history**. Một skill như `writing-fastendpoints` cần thấy code hiện tại của bạn để áp dụng conventions — fork nó sẽ khiến nó vô dụng.

```yaml
# ❌ Sai — reference skills cần conversation context
---
name: writing-fastendpoints
context: fork   # Subagent này không thấy code hiện tại của bạn
---

# ✅ Đúng — chạy inline
---
name: writing-fastendpoints
description: FastEndpoints conventions. Dùng khi viết hoặc review endpoints.
---
```

### 5. Không đặt tên mơ hồ hoặc quá generic

```
# ❌ Tránh
helper
utils
do-stuff
my-skill

# ✅ Cụ thể
reviewing-fastendpoints
migrating-mstest-to-xunit
generating-ef-migrations
writing-masstransit-consumers
```

### 6. Không bỏ qua testing với nhiều models

Skills hoạt động khác nhau trên Haiku, Sonnet, và Opus. Một skill hoạt động tốt trên Opus có thể cần hướng dẫn chi tiết hơn cho Haiku.

### 7. Không cài skills từ nguồn không tin cậy mà không audit

Skills có thể chứa executable scripts. Luôn đọc toàn bộ nội dung của third-party skill trước khi cài — chú ý đến external network calls, data exfiltration patterns, và unsafe shell commands.

---

## Decision Matrix — Skills vs. Subagents

| Câu hỏi | Nếu CÓ → | Nếu KHÔNG → |
|---|---|---|
| Task tạo verbose output (logs, file dumps, test results)? | Subagent | Skill (inline) |
| Task cần toàn bộ conversation history? | Skill (inline) | Cả hai đều được |
| Đây là reusable workflow/convention áp dụng cho công việc hiện tại? | Skill (inline) | Subagent |
| Task cần context isolation để output gọn sạch? | Skill với `context: fork` | Skill (inline) |
| Cùng instructions được tái dùng cho nhiều subagents? | Skill preload qua `skills:` frontmatter | — |
| Task hoàn toàn self-contained với summary output rõ ràng? | Subagent | Skill |

---

## Khi nào dùng Skills vs. CLAUDE.md

Cả `CLAUDE.md` và skills đều lưu instructions cho Claude — nhưng phục vụ mục đích khác nhau.

| | `CLAUDE.md` | Skills |
|---|---|---|
| **Khi nào load** | Luôn luôn, ở mỗi session start | Theo yêu cầu, khi liên quan |
| **Token cost** | Luôn được tính | Gần như 0 cho đến khi được trigger |
| **Phù hợp nhất** | Facts ngắn, project-wide rules, context luôn áp dụng | Procedures dài, workflows tùy chọn, tài liệu tham khảo |
| **Trigger** | Tự động, không điều kiện | Theo relevance (auto) hoặc `/skill-name` (explicit) |

> **Hướng dẫn từ Anthropic:** Nếu một section trong `CLAUDE.md` đã phát triển thành một procedure thay vì một fact, hãy chuyển nó sang skill.

---

## Checklist trước khi tạo Skill mới

```
□ Đây có phải instructions Claude chưa có? (không chỉ lặp lại những gì Claude đã biết)
□ Description có action-oriented với trigger condition rõ ràng?
□ Nội dung có ngắn gọn — chỉ những gì Claude thực sự cần?
□ Có nên tách supporting files ra để tránh load tất cả cùng lúc?
□ Đây là action skill? → Thêm disable-model-invocation: true
□ Output sẽ làm ngập main context? → Thêm context: fork
□ Đây là project-specific? → Lưu vào .claude/skills/ và commit vào version control
□ Đây là fact/rule luôn áp dụng? → Cân nhắc dùng CLAUDE.md thay thế
```

---

## Tài liệu tham khảo

- [Anthropic — Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Anthropic API Docs — Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Anthropic Engineering — Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Agent Skills open standard](https://agentskills.io)
- [Level Up Coding — A Mental Model for Claude Code: Skills, Subagents, and Plugins](https://levelup.gitconnected.com/a-mental-model-for-claude-code-skills-subagents-and-plugins-3dea9924bf05)
- [Firecrawl — Best Claude Code Skills to Try in 2026](https://www.firecrawl.dev/blog/best-claude-code-skills)
