# Claude Code Commands — Best Practices

> **Nguồn tài liệu:** Anthropic official documentation ([code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands)), [code.claude.com/docs/en/features-overview](https://code.claude.com/docs/en/features-overview), [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills), và các tài liệu cộng đồng uy tín (alexop.dev, MindStudio, Daniel Miessler).

---

## Commands là gì?

Commands điều khiển Claude Code từ bên trong một session. Có hai loại:

**Built-in commands** — được code sẵn vào CLI. Ví dụ: `/clear`, `/compact`, `/memory`, `/plan`, `/init`. Hành vi của chúng là cố định.

**Custom commands (skills)** — các file markdown bạn tự viết. Được lưu trong `.claude/commands/` hoặc `.claude/skills/`, chúng tạo ra các phím tắt `/command-name` cho các workflow lặp lại. Từ tháng 10/2025, custom commands đã được **gộp vào hệ thống Skills** — cả hai cơ chế đều giống hệt nhau về bản chất.

> **Official Anthropic guidance:** "Custom commands đã được gộp vào skills. Một file tại `.claude/commands/deploy.md` và một skill tại `.claude/skills/deploy/SKILL.md` đều tạo ra `/deploy` và hoạt động như nhau. Skills được khuyến nghị vì chúng hỗ trợ thêm nhiều tính năng hơn."

Gõ `/` trong bất kỳ session Claude Code nào để xem tất cả commands, hoặc gõ `/` theo sau là các ký tự để filter.

---

## Built-in Commands Reference

Các built-in commands quan trọng nhất, phân loại theo nhóm.

### Context & Session Management

| Command | Mục đích |
|---|---|
| `/clear` | Xóa conversation history và giải phóng context. Aliases: `/reset`, `/new` |
| `/compact [instructions]` | Tóm tắt conversation để giảm context usage. Truyền focus instructions để giữ lại thông tin cụ thể |
| `/context` | Visualize context usage dưới dạng lưới màu. Hiển thị gợi ý tối ưu hóa và cảnh báo capacity |
| `/branch [name]` | Fork conversation hiện tại tại điểm này, giữ nguyên bản gốc. Alias: `/fork` |
| `/resume [session]` | Resume một conversation trước đó theo ID hoặc tên |
| `/rewind` | Rewind conversation và/hoặc code về một checkpoint trước. Alias: `/checkpoint` |
| `/btw <question>` | Hỏi một câu nhanh mà không thêm vào conversation history |

### Planning & Workflow

| Command | Mục đích |
|---|---|
| `/init` | Generate `CLAUDE.md` starter từ phân tích codebase |
| `/plan [description]` | Vào plan mode. Truyền description để bắt đầu planning ngay |
| `/batch <instruction>` | **[Skill]** Orchestrate large-scale parallel changes. Phân rã công việc thành 5–30 units, spawn một background agent per unit trong isolated git worktrees |
| `/simplify [focus]` | **[Skill]** Review các files vừa thay đổi để tìm vấn đề chất lượng. Spawn ba review agents song song, tổng hợp findings, apply fixes |
| `/debug [description]` | **[Skill]** Bật debug logging và troubleshoot bằng cách đọc session debug log |

### Inspection & Diagnostics

| Command | Mục đích |
|---|---|
| `/memory` | Edit `CLAUDE.md` files, quản lý auto-memory, xem những gì đã load |
| `/hooks` | Xem hook configurations cho tool events |
| `/skills` | Liệt kê các skills từ project, user, và plugin sources |
| `/agents` | Quản lý subagent configurations |
| `/permissions` | Quản lý allow, ask, và deny rules cho tool permissions |
| `/doctor` | Chẩn đoán Claude Code installation và settings |
| `/cost` | Hiển thị token usage statistics cho session hiện tại |
| `/diff` | Mở interactive diff viewer hiển thị uncommitted changes và per-turn diffs |

### Automation & Scheduling

| Command | Mục đích |
|---|---|
| `/loop [interval] [prompt]` | **[Skill]** Chạy một prompt lặp lại trong khi session vẫn mở |
| `/schedule [description]` | Tạo, cập nhật, liệt kê, hoặc chạy các scheduled routines |
| `/autofix-pr [prompt]` | Spawn một web session theo dõi PR và push fixes khi CI fail hoặc reviewer comment |

### Utility

| Command | Mục đích |
|---|---|
| `/copy [N]` | Copy assistant response cuối vào clipboard. Truyền `N` để copy response thứ N từ cuối |
| `/export [filename]` | Export conversation hiện tại ra plain text |
| `/model [model]` | Chọn hoặc thay đổi AI model giữa chừng |
| `/effort [level]` | Đặt model effort: `low`, `medium`, `high`, `max`, hoặc `auto` |
| `/security-review` | Phân tích các pending changes để tìm security vulnerabilities |
| `/insights` | Generate báo cáo phân tích các session patterns và friction points |

---

## Bundled Skills (Được gọi như Commands)

Một số entries trong command list là **bundled skills** — prompts giao cho Claude, không phải hardcoded CLI behavior. Chúng dùng cùng cơ chế với skills bạn tự viết, và Claude cũng có thể tự invoke chúng khi liên quan.

| Command | Làm gì |
|---|---|
| `/batch` | Large-scale refactors song song với isolated git worktrees per unit |
| `/simplify` | Ba review agents song song tổng hợp findings và apply fixes |
| `/debug` | Phân tích session debug log |
| `/loop` | Thực thi prompt lặp lại, tự điều chỉnh tốc độ giữa các iterations |
| `/claude-api` | Load Claude API reference cho ngôn ngữ của project hiện tại |

---

## Custom Commands (Tự viết)

Custom commands là các file markdown tạo ra `/command-name` shortcuts cho các workflow của riêng bạn.

### Hai vị trí lưu trữ

```
# Single-file command (format cũ, vẫn hoạt động)
.claude/commands/deploy.md        →  gọi bằng /deploy

# Directory-based skill (khuyến nghị, nhiều tính năng hơn)
.claude/skills/deploy/SKILL.md   →  gọi bằng /deploy
```

Cả hai đều tạo cùng một lệnh `/deploy`. Dùng dạng directory khi bạn cần supporting files (scripts, templates, examples).

### Cấu trúc Custom Command cơ bản

```markdown
---
description: Deploy nhánh hiện tại lên staging
allowed-tools: Bash, Read
disable-model-invocation: true
---

Deploy lên staging:
1. Chạy tests — `dotnet test`
2. Build Docker image — `docker build -t app:staging .`
3. Push và deploy — `./scripts/deploy.sh staging`
4. Verify health endpoint — `./scripts/health-check.sh staging`
```

### Sử dụng `$ARGUMENTS`

Commands có thể nhận arguments được truyền vào sau tên command. Placeholder `$ARGUMENTS` được thay thế bằng text mà user gõ sau command.

```markdown
---
description: Tra cứu tài liệu cho một thư viện hoặc API
allowed-tools: WebFetch, Read
---

Fetch tài liệu hiện tại cho: $ARGUMENTS

1. Kiểm tra có llms.txt tại root URL không
2. Fetch các documentation pages liên quan
3. Trả lời câu hỏi dựa trên tài liệu vừa fetch, không dùng training data
```

Invoke bằng: `/lookup-docs Dexie.js liveQuery usage`

### Orchestrate Subagents từ một Command

Commands có thể explicitly spawn subagents song song bằng cách đưa Task tool instructions vào body:

```markdown
---
description: Research một vấn đề kỹ thuật bằng parallel agents
allowed-tools: Task, WebSearch, WebFetch, Grep, Glob, Read, Write
---

Research chủ đề sau: $ARGUMENTS

Spawn các subagents này SONG SONG bằng Task tool:

1. **Web Agent** — tìm kiếm official documentation và GitHub issues
2. **Codebase Agent** — tìm kiếm các patterns hiện có trong codebase này
3. **Stack Overflow Agent** — tìm community solutions và workarounds

Tổng hợp findings vào `docs/research/$ARGUMENTS.md`.
```

---

## Commands vs Skills vs CLAUDE.md — Khi nào dùng cái gì

Đây là quyết định quan trọng nhất cần hiểu rõ.

| Feature | Load khi nào | Ai invoke | Phù hợp nhất cho |
|---|---|---|---|
| **CLAUDE.md** | Mỗi session, luôn luôn | Tự động | Facts "luôn làm X", conventions, project structure |
| **Built-in command** | Khi cần | Bạn gõ `/command` | Session control: clear, compact, plan, review |
| **Custom command / skill** | Khi cần | Bạn gõ `/command` | Repeatable workflows bạn trigger thủ công |
| **Skill (auto-invoke)** | Khi cần | Claude phát hiện relevance | Reference knowledge, conventions Claude tự áp dụng |
| **Subagent** | Khi cần | Claude hoặc explicit delegation | Heavy tasks cô lập, parallel work, verbose output |

### Phân biệt then chốt: Commands vs Skills

Cả hai đều là markdown files. Sự khác biệt nằm ở **cách invoke và liệu Claude có thể tự invoke hay không**.

| | Custom Command | Skill (auto-invocable) |
|---|---|---|
| **Invocation** | Tường minh — bạn gõ `/name` | Tường minh (`/name`) **hoặc** Claude tự invoke |
| **Supporting files** | Chỉ một `.md` file | Directory với scripts, templates, examples |
| **Auto-invocation** | Không theo mặc định | Có, dựa trên description matching |
| **Phù hợp nhất** | Deterministic workflows bạn kiểm soát | Reference knowledge + workflows |

> **Rule từ official docs:** "Lưu nó thành user-invocable skill khi bạn cứ phải gõ cùng một prompt để bắt đầu task. Dùng CLAUDE.md khi Claude mắc cùng một convention error hai lần."

### Trigger Guide thực tế

| Bạn nhận ra điều này... | Thêm cái này |
|---|---|
| Claude mắc convention error hai lần | Thêm vào `CLAUDE.md` |
| Bạn gõ cùng một prompt để bắt đầu mỗi task | Lưu thành custom command / skill |
| Bạn paste cùng một multi-step procedure vào chat | Capture thành skill |
| Một side task làm ngập conversation với verbose output | Route qua subagent |
| Bạn muốn gì đó xảy ra tự động mỗi lần mà không cần hỏi | Viết hook |

---

## ✅ NÊN — Design Guidelines cho Custom Commands

### 1. Dùng `disable-model-invocation: true` cho action commands

Bất kỳ command nào thực hiện operation không thể hoàn tác (deploy, migrate, release, format) phải yêu cầu invoke tường minh. Không bao giờ để Claude tự trigger dựa trên keyword matching.

```yaml
---
name: run-migrations
description: Chạy pending EF Core database migrations
disable-model-invocation: true
allowed-tools: Bash
---

Chạy EF Core migrations:
1. `dotnet ef migrations list` — xác nhận pending migrations
2. `dotnet ef database update` — apply migrations
3. Verify: `dotnet ef migrations list` phải hiển thị tất cả là applied
```

### 2. Scope `allowed-tools` ở mức minimum cần thiết

Giống như subagents, commands chỉ nên có các tools cần thiết cho công việc của chúng.

```yaml
# ✅ Read-only research command
allowed-tools: Read, Grep, Glob, WebFetch

# ✅ Deployment command cần Bash
allowed-tools: Bash, Read

# ❌ Không restrict — kế thừa tất cả kể cả MCP servers
# (bỏ qua allowed-tools hoàn toàn)
```

### 3. Dùng `$ARGUMENTS` cho parameterizable commands

Làm commands linh hoạt bằng cách nhận arguments thay vì hardcode mọi thứ. Điều này tránh phải tạo nhiều commands gần giống nhau cho các tasks tương tự.

```yaml
---
description: Chạy tests cho một module hoặc path cụ thể
allowed-tools: Bash
---

Chạy tests cho: $ARGUMENTS

Execute: `dotnet test --filter "$ARGUMENTS" --collect:"XPlat Code Coverage"`
Báo cáo bất kỳ failures nào với full output.
```

### 4. Viết descriptions như trigger conditions

Field `description` được dùng cả cho autocomplete display khi gõ `/` và cho quyết định auto-invocation của Claude. Viết nó như một trigger condition, không chỉ là một label.

```yaml
# ✅ Trigger condition rõ ràng
description: Tạo changelog entry cho các thay đổi trên nhánh hiện tại. Dùng sau khi hoàn thành một feature hoặc fix.

# ❌ Chỉ là label
description: Changelog command
```

### 5. Commit project commands vào version control

Project-level commands (`.claude/commands/` hoặc `.claude/skills/`) nên được commit để cả team chia sẻ cùng workflows. Xem chúng như code.

### 6. Dùng dạng directory cho complex commands

Khi một command cần reference files, templates, hoặc scripts, dùng skill directory structure thay vì single `.md` file:

```
.claude/skills/
└── generate-pr-description/
    ├── SKILL.md          # Instructions chính
    ├── template.md       # PR description template
    └── examples/
        └── sample-pr.md  # Ví dụ output
```

---

## ❌ KHÔNG NÊN — Các lỗi phổ biến

### 1. Không dùng commands cho conventions luôn áp dụng

Nếu Claude nên biết một rule — "không bao giờ dùng `dynamic`", "luôn chạy tests trước khi commit" — đặt nó vào `CLAUDE.md`. Commands chỉ thực thi khi được invoke. Rules thuộc CLAUDE.md sẽ không áp dụng trừ khi command được chạy tường minh.

### 2. Không bỏ qua `disable-model-invocation` cho action commands

Một deploy hoặc migration command mà Claude auto-trigger dựa trên keyword trong prompt là nguy hiểm. Luôn thêm `disable-model-invocation: true` cho bất kỳ thứ gì không thể hoàn tác.

### 3. Không xây commands cho những thứ hooks xử lý tốt hơn

Nếu bạn muốn thứ gì đó tự động xảy ra khi một sự kiện cụ thể xảy ra (sau mỗi file edit, trước khi commit), dùng hook thay vì command. Commands yêu cầu bạn gõ `/command-name`. Hooks fire theo cách deterministic.

```
# Bạn muốn ESLint chạy sau mỗi file write
# ❌ Sai: /lint command bạn phải nhớ chạy
# ✅ Đúng: PostToolUse hook trên Write/Edit tools
```

### 4. Không tạo nhiều commands gần giống nhau

Nếu bạn có `/test-api`, `/test-db`, `/test-auth` đều làm cùng một việc với arguments khác nhau, hợp nhất thành một parameterized command: `/test $ARGUMENTS`.

### 5. Không nhầm `/btw` với custom command

`/btw` là built-in để hỏi nhanh mà không thêm vào conversation history. Nó không phải là custom command trigger. Dùng nó khi bạn muốn hỏi thứ gì đó mà không làm ô nhiễm session context.

---

## Các Built-in Commands Quan Trọng Trong Công Việc Hàng Ngày

### Context Management Workflow

```
# Đầu session dài
/init               # Generate CLAUDE.md nếu chưa có
/plan fix auth bug  # Vào plan mode trước khi code

# Giữa session khi context đang đầy
/context            # Kiểm tra usage
/compact            # Tóm tắt, tiếp tục làm

# Khi bắt đầu lại từ đầu
/clear              # Full reset
```

### Inspection Trước Khi Hành Động

```
/memory             # Xem CLAUDE.md content nào đã load
/context            # Xem token breakdown
/diff               # Review uncommitted changes trước khi commit
/security-review    # Kiểm tra vulnerabilities trong current diff
```

### Session Branching để Explore

```
/branch experiment-approach-a   # Fork session hiện tại
# ... thử một cách tiếp cận ...
/resume             # Quay lại session gốc
/branch experiment-approach-b   # Thử cách tiếp cận khác
```

---

## MCP Prompts dưới dạng Commands

MCP servers có thể expose prompts xuất hiện như commands theo format `/mcp__<server>__<prompt>`. Chúng được tự động discover từ các connected servers và hoạt động như built-in commands.

Ví dụ, nếu một GitHub MCP server expose prompt `create-pr`, nó xuất hiện như `/mcp__github__create-pr` trong command list.

---

## Checklist Trước Khi Tạo Custom Command

```
□ Đây là repeatable workflow bạn trigger thủ công? → Custom command / skill
□ Claude nên biết điều này mọi lúc? → CLAUDE.md thay vào đó
□ Điều này nên xảy ra tự động khi có event? → Hook thay vào đó
□ Command có thực hiện irreversible operations không? → Thêm disable-model-invocation: true
□ Tools đã được restrict ở mức minimum cần thiết chưa?
□ Có thể parameterize với $ARGUMENTS để thay thế nhiều commands tương tự không?
□ Description có đóng vai trò như trigger condition rõ ràng không?
□ Đã commit vào version control cho team chưa?
```

---

## Tài liệu tham khảo

- [Anthropic — Commands reference](https://code.claude.com/docs/en/commands)
- [Anthropic — Extend Claude Code (feature comparison)](https://code.claude.com/docs/en/features-overview)
- [Anthropic — Skills documentation](https://code.claude.com/docs/en/skills)
- [alexop.dev — Claude Code Customization: CLAUDE.md, Slash Commands, Skills, and Subagents](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/)
- [MindStudio — Claude Skills vs Slash Commands: When to Use Each](https://www.mindstudio.ai/blog/claude-skills-vs-slash-commands)
- [Daniel Miessler — When to Use Skills vs Workflows vs Agents](https://danielmiessler.com/blog/when-to-use-skills-vs-commands-vs-agents)
