# Project Cockpit: Hợp nhất OKR + Knowledge Engine

## Tổng quan

Hệ thống hợp nhất 2 bộ skill: Objective Kit (quản lý mục tiêu, tracking công việc) và Knowledge Engine (quản lý knowhow, bài học). Kết quả là **1 hệ thống per-project cho cá nhân**, gồm 2 module ngang hàng:

- **Module Thực thi**: mục tiêu, kế hoạch, hành động, tracking, metrics (từ OKR)
- **Module Tri thức**: wiki, skills, workflows, distill, consult (từ Knowledge Engine)
- **Chia sẻ**: inbox thống nhất, archive, raw, schema, log

Mối quan hệ lõi: **tri thức hỗ trợ mục tiêu**. Tri thức tồn tại độc lập nhưng phục vụ việc theo đuổi mục tiêu. Khi lập kế hoạch, tra cứu tri thức. Khi xong việc, rút bài học vào kho.

Giữ nguyên cả hai điểm mạnh:

- Từ OKR: SOT ownership nghiêm ngặt, metrics chuẩn, preload contract
- Từ Knowhow: lifecycle (raw → inbox → wiki → skills), distill (ưu tiên cập nhật hơn tạo mới), schema tự tiến hóa

## Kiến trúc dữ liệu

Thư mục gốc: `.cockpit/`

```
.cockpit/
├── SCHEMA.md                    # Quy ước dữ liệu + schema version
│
├── ── MODULE THỰC THI ──
├── objective.md                 # Mục tiêu + KR/KI + Constraints (capacity, budget, gaps/risks)
├── tools.md                     # Công cụ tương tác (luôn load)
├── plan.md                      # Milestones + counters
├── actions/                     # 1 file/action: AXXX-slug.md
│
├── ── MODULE TRI THỨC ──
├── knowledge/
│   ├── index.md                 # Registry tất cả trang wiki (bảng đơn 7 cột)
│   ├── decision-*.md            # Quyết định chiến lược
│   ├── pattern-*.md             # Mẫu đã chứng minh
│   ├── concept-*.md             # Thuật ngữ, khái niệm, tài liệu tham khảo
│   ├── troubleshooting-*.md     # Vấn đề + cách xử lý
│   ├── lesson-*.md              # Bài học: kỳ vọng vs thực tế + hành động hệ thống
│   └── resource-profile.md      # Hồ sơ năng lực tổng hợp (pinned: true, luôn preload full body)
├── skills/                      # Quy trình tái sử dụng (1 file = 1 skill)
│   └── registry.md
├── workflows/                   # Chuỗi quy trình (gọi nhiều skill)
│   └── registry.md
│
├── ── CHIA SẺ ──
├── inbox/                       # Thống nhất: cả action items + knowledge items
├── raw/                         # Nguồn gốc ghi chép ban đầu (immutable)
├── log/                         # Append-only: 1 file/ngày (YYYY-MM-DD.md)
├── archive/                     # Xóa mềm toàn hệ thống (reversible)
│   ├── actions/
│   ├── inbox/
│   ├── knowledge/
│   ├── skills/
│   └── workflows/
└── schema-signals.md            # Log tín hiệu "khuôn không vừa"
```

### Mapping từ hệ thống cũ

| Khái niệm cũ | Hợp nhất thành |
|---------------|----------------|
| `.okr/lessons/skill/` | `skills/` (đầy đủ: trigger, input, output, steps) |
| `.okr/lessons/workflow/` | `workflows/` (có registry, dependency) |
| `.okr/lessons/project/` | `knowledge/lesson-*.md` + `knowledge/concept-*.md` |
| `.okr/context/` | `knowledge/concept-*.md` + `knowledge/pattern-*.md` |
| `.okr/inbox/` + `.knowhow/inbox/` | `inbox/` thống nhất |
| `.knowhow/wiki/log.md` + `.okr/log/` | `log/` thống nhất |
| `resources.md` công cụ | `tools.md` (root level) |
| `resources.md` capacity/budget/gaps | `objective.md` section Constraints |
| `resources.md` tài liệu/KB | `knowledge/concept-*.md` |

### tools.md

Công cụ tương tác, luôn load ở root level. Hệ thống dùng để biết có thể tương tác với gì trong quá trình làm việc.

```markdown
# Tools

| Name | Purpose | Interface | Usage |
|------|---------|-----------|-------|
| Things3 | Quản lý task cá nhân | CLI: `things3-cli` | Sync actions ↔ Things3 |
| Google Drive | Lưu trữ file | MCP: `gws-drive` | Upload deliverables |
| Terraform | Provisioning infra | CLI: `terraform` | Tạo/sửa infrastructure |
```

Cột `Interface` cho hệ thống biết cách tương tác: CLI command, MCP tool, API, hay manual.

### knowledge/index.md

Bảng đơn, không phân nhóm theo type:

```markdown
# Knowledge Index

| Slug | Type | Title | Tags | Status | Usage | Updated |
|------|------|-------|------|--------|-------|---------|
| decision-tech-stack | decision | Chọn tech stack | [arch,infra] | active | 0 | 2026-06-10 |
| pattern-retry-backoff | pattern | Retry backoff strategy | [resilience] | active | 3 | 2026-06-08 |
| lesson-kr2-baseline | lesson | KR2 baseline sai | [metrics] | active | 1 | 2026-06-09 |
```

- `Usage`: cache count, pk-consult ghi usage vào log, pk-distill cập nhật count khi chạy
- `Status`: active / deprecated / archived

### Inbox item thống nhất

```yaml
---
type: action|blocker|resource|thought|decision|pattern|troubleshooting|concept|lesson|candidate-skill|candidate-workflow
domain: execution|knowledge
title: "..."
captured_at: "YYYY-MM-DDTHH:mm"
source_file: "raw/..."
related_kr: KR1 | null
related_action: A005 | null
related_inbox: "YYYY-MM-DD-HHmm-slug.md" | null
status: pending|processed|discarded
---
```

Trường `domain` phân luồng xử lý:

- `execution` → pk-track xử lý
- `knowledge` → pk-distill xử lý

Quy tắc phân loại tự động theo `type`:

- `action`, `blocker` → domain=execution
- `decision`, `pattern`, `concept`, `troubleshooting`, `lesson`, `candidate-skill`, `candidate-workflow` → domain=knowledge
- `resource`, `thought` → hỏi user

Trường hợp `domain=both`: pk-capture tách ngay lúc tạo thành 2 items riêng biệt, link nhau qua `related_inbox`. Item execution focus vào action status, item knowledge focus vào nội dung tri thức.

### Log thống nhất

Mỗi ngày 1 file: `log/YYYY-MM-DD.md`. Mỗi entry là 1 block markdown:

```yaml
---
timestamp: 2026-06-10T14:30
type: tracking|review|knowledge-activity
source_skill: pk-track|pk-consult|pk-distill|pk-lint
---
```

- `tracking`: cập nhật KR/action (từ pk-track light)
- `review`: deep review, root cause analysis (từ pk-track deep)
- `knowledge-activity`: create/update page, query, run skill, lint fix (từ pk-consult, pk-distill, pk-lint)

### Knowledge page format

Minimal bắt buộc + optional sections. Chỉ giữ section có nội dung, không tạo section rỗng.

```yaml
---
type: concept|decision|pattern|troubleshooting|lesson|resource
title: "..."
tags: [...]
related: [...]
status: active|deprecated|archived
confidence: low|medium|high
pinned: false
updated: YYYY-MM-DD
---

## Tóm tắt
(BẮT BUỘC. 1-2 câu: cái này là gì, tại sao quan trọng.)

## Khi nào dùng
(Optional.)

## Cách dùng
(Optional.)

## Ví dụ
(Optional.)

## Nguồn
(Optional. Link gốc, reference.)
```

Trường `pinned: true` → snapshot preload full body (không chỉ frontmatter). Mặc định false.

Wiki types ban đầu: decision, pattern, concept, troubleshooting, lesson, resource (6 types).

## Bản đồ Skill

15 skill gốc → 11 skill hợp nhất. Prefix: `pk-` (project-knowledge).

### Bảng hợp nhất

| Skill | Nguồn gốc | Vai trò |
|-------|-----------|---------|
| **pk-harness** | okr-harness | Điểm vào duy nhất. Đọc snapshot, phân luồng execution vs knowledge. Tối đa 1 câu clarify nếu intent mơ hồ. |
| **pk-init** | okr-init + knowhow-init | Tạo `.cockpit/`, SCHEMA.md, registries. Hỏi tools + objective + KR/KI + constraints. Objective optional (knowledge-only mode cho phép). |
| **pk-analyze** | okr-analyze | Read-only thuần, KHÔNG ghi bất kỳ file nào. Metrics, dashboard hợp nhất, phát hiện vấn đề, reachability audit. |
| **pk-plan** | okr-plan | Tạo/sửa plan, milestones, actions. Gọi pk-consult tìm knowledge liên quan khi tạo action. |
| **pk-track** | okr-track | 3 mode: light (cập nhật tiến độ), deep (review sâu + root cause), inbox-only (xử lý inbox domain=execution). Deep review dùng `pre_confirmed` batch. |
| **pk-capture** | okr-capture + knowhow-capture | Inbox thống nhất. Phân loại domain tự động. domain=both tách ngay thành 2 items link nhau. |
| **pk-reflect** | okr-retro + knowhow-reflect | Default light (rút nhanh từ session). Deep phải chỉ định rõ (phỏng vấn AAR 5 câu, gọi pk-analyze lấy data). Output → inbox. |
| **pk-distill** | knowhow-distill | Xử lý inbox domain=knowledge → wiki/skills/workflows. Ưu tiên cập nhật hơn tạo mới. Cache usage_count từ log vào index.md. |
| **pk-consult** | knowhow-query + knowhow-run | Tham vấn tri thức. Auto-detect mode dựa trên ngữ cảnh: cần thông tin → query (read-only), cần hành động → run (thực thi skill/workflow). Ghi usage vào log. Tự load concept pages liên quan từ tools.md inventory khi match, không hỏi user. |
| **pk-lint** | knowhow-lint | 3 mode: check (health check + metrics, read-only), fix (consolidation + rebuild-index + restore, có ghi), evolve (schema-review, hiếm dùng). |
| **pk-shared** | okr-shared + knowhow schema | Không chạy độc lập. Hub canonical: 6 reference files. |

### Skill bị gộp hoặc loại

| Skill gốc | Số phận | Lý do |
|-----------|---------|-------|
| okr-retro | Gộp vào pk-reflect (mode light) | Reflect bao hàm retro |
| knowhow-init | Gộp vào pk-init | 1 lần init |
| knowhow-capture | Gộp vào pk-capture | Inbox thống nhất |
| knowhow-query | Gộp vào pk-consult (mode query) | Cùng nhu cầu "tham vấn tri thức" |
| knowhow-run | Gộp vào pk-consult (mode run) | Cùng nhu cầu, khác đầu ra |

### Cross-call Rules

Skill được phép gọi nhau, 1 chiều theo lớp:

```
Lớp 1 (orchestrator):  pk-harness
Lớp 2 (core):          pk-plan, pk-track, pk-reflect
Lớp 3 (support):       pk-capture, pk-consult, pk-distill, pk-lint
Lớp 4 (foundation):    pk-analyze, pk-init, pk-shared
```

Lớp trên gọi được lớp dưới hoặc cùng lớp. Lớp dưới KHÔNG gọi ngược lên.

Ví dụ hợp lệ:

- pk-plan (2) → pk-consult (3) ✓
- pk-track (2) → pk-capture (3) ✓
- pk-reflect (2) → pk-capture (3) ✓
- pk-reflect (2) → pk-analyze (4) ✓
- pk-distill (3) → pk-analyze (4) ✓

Ví dụ KHÔNG hợp lệ:

- pk-consult (3) → pk-track (2) ✗
- pk-analyze (4) → pk-plan (2) ✗

### SOT Ownership

| Field/File | Skill ghi |
|------------|-----------|
| objective.md (text, KR/KI targets, period, status, constraints) | pk-init |
| tools.md | pk-init |
| plan.md (milestones, roadmap) | pk-plan |
| actions/*.md (cấu trúc: title, deadline, deps) | pk-plan |
| actions/*.md (status, current values) | pk-track |
| KR/KI current, plan counters | pk-track |
| inbox/ (tạo) | pk-capture, pk-reflect |
| inbox/ domain=execution (xử lý) | pk-track |
| inbox/ domain=knowledge (xử lý) | pk-distill |
| knowledge/*.md, knowledge/index.md | pk-distill |
| knowledge/index.md usage_count | pk-distill (cache từ log) |
| skills/*.md, skills/registry.md | pk-distill |
| workflows/*.md, workflows/registry.md | pk-distill |
| log/ (append) | pk-track, pk-consult (usage), pk-lint |
| raw/ (tạo, immutable) | pk-capture, pk-reflect |
| archive/ | pk-track (actions), pk-distill (knowledge, skills, workflows), pk-lint (restore) |
| SCHEMA.md | pk-lint (evolve mode) |
| schema-signals.md | pk-distill, pk-consult (emit) |

### Preload Contract

**1 context snapshot duy nhất**, định nghĩa trong pk-shared, mọi skill đều đọc:

```
ls -1 .cockpit/                       # Detect cấu trúc, anomalies
objective.md        frontmatter       # type, status, period, KR/KI IDs, constraints tóm tắt
tools.md            toàn bộ           # Inventory công cụ (luôn load)
plan.md             frontmatter       # counters, milestones tóm tắt, dates
actions/            scan frontmatter   # count by status, active IDs
inbox/              scan frontmatter   # count by domain
knowledge/index.md  toàn bộ           # Registry tri thức + usage count
skills/registry.md  toàn bộ           # Registry skills
workflows/registry.md toàn bộ         # Registry workflows
Pinned pages        full body          # Pages có pinned: true trong frontmatter
```

**Mọi thứ ngoài snapshot: on-demand.** Skill cần body objective.md? Đọc lúc cần. Cần chi tiết 1 action? Đọc lúc cần.

Khi skill phát hiện keyword match giữa task hiện tại và tool/concept trong snapshot → tự load concept page chi tiết, không hỏi user.

Maintain 1 chỗ duy nhất: `pk-shared/references/snapshot-contract.md`.

### Graceful Degradation

Skill tự detect mode dựa trên file tồn tại:

| Tình trạng | Hành vi |
|------------|---------|
| Không có `.cockpit/` | pk-harness route sang pk-init |
| Có `.cockpit/`, không có objective.md | pk-harness chỉ route sang knowledge skills. pk-plan/pk-track/pk-analyze báo "chưa có mục tiêu." |
| Có objective.md, chưa có plan.md | pk-track báo "chưa có plan." pk-analyze chỉ hiện KR status. |
| Có cả hai | Đầy đủ chức năng |

pk-capture khi không có objective: `related_kr` và `related_action` để null, domain mặc định = knowledge.

## Luồng tích hợp

### Inbox thống nhất

```
pk-capture / pk-reflect
    ↓ tạo item (domain=both → tách ngay thành 2 items, link qua related_inbox)
inbox/
    ↓
domain=execution → pk-track xử lý → actions/, plan.md
domain=knowledge → pk-distill xử lý → knowledge/, skills/, workflows/
```

### Bốn điểm tích hợp chính

**1. Plan → Consult (tri thức hỗ trợ lập kế hoạch)**

pk-plan gọi pk-consult (cross-call lớp 2 → lớp 3) tìm pattern, decision, troubleshooting liên quan khi tạo action. Nếu có skill/workflow phù hợp, link vào action notes.

**2. Track → Capture (thực thi sinh tri thức)**

pk-track deep review phát hiện pattern mới hoặc lesson → gọi pk-capture (cross-call lớp 2 → lớp 3) tạo inbox item. pk-track KHÔNG tự ghi vào knowledge (giữ SOT).

**3. Reflect → Analyze (phân tích nuôi phản tư)**

pk-reflect deep mode gọi pk-analyze (cross-call lớp 2 → lớp 4) lấy metrics, trends làm dữ liệu đầu vào cho phỏng vấn AAR. Output → inbox domain=knowledge → distill.

**4. Track → Consult (thực thi skill hỗ trợ tracking)**

pk-track phát hiện action có skill/workflow liên quan → gọi pk-consult mode run (cross-call lớp 2 → lớp 3). Kết quả quay về pk-track cập nhật action status.

### Dashboard hợp nhất

pk-analyze (read-only) hiển thị sức khỏe CẢ HAI module:

- Execution: KR progress, action health, trend, issues
- Knowledge: wiki page count, skill/workflow count, inbox knowledge pending, page freshness, orphaned files

### Deep Review + pre_confirmed

pk-track deep review phát hiện cần sửa milestone/action structure:

1. pk-track gom proposals → trình user duyệt batch
2. User duyệt tất cả cùng lúc
3. pk-track gọi pk-plan/pk-init với payload `pre_confirmed: true`
4. pk-plan/pk-init thực thi ngay, không hỏi confirm lại

## Cơ chế bảo vệ

### Lesson lifecycle

Bài học chín dần theo lifecycle, không phân loại 3 tier ngay lúc extract:

```
Phát hiện (reflect/track)
    → inbox/ (domain=knowledge, type=lesson)
    → pk-distill xử lý:
        ├── Domain fact → knowledge/lesson-*.md
        ├── Process pattern → knowledge/pattern-*.md
        └── Tool capability → knowledge/pattern-*.md (hoặc troubleshooting)
```

Promote flow (wiki → skill/workflow):

1. pk-consult ghi usage vào log/ mỗi khi match 1 page
2. pk-distill đọc log, cập nhật `usage_count` trong knowledge/index.md
3. Khi usage_count ≥ 3 + type là pattern/lesson → pk-distill gợi ý promote
4. Promote: nội dung chuyển sang skills/ hoặc workflows/, page gốc thành stub chỉ link redirect
5. knowledge/index.md cập nhật, pk-consult tìm thấy stub → tự redirect sang skill

### Reachability

Mọi file phải reachable từ SOT:

- Execution: objective → plan → actions → output
- Knowledge: SCHEMA.md → knowledge/index.md → pages. skills/registry.md → skills. workflows/registry.md → workflows.
- Cross-module: knowledge page có `related_kr` / `related_action` field

### SCHEMA.md

Chỉ chứa quy ước dữ liệu (không chứa SOT ownership hay cross-call rules):

- Cấu trúc thư mục `.cockpit/`
- Quy ước naming (actions: AXXX-slug.md, inbox: YYYY-MM-DD-HHmm-slug.md, knowledge: {type}-{slug}.md, log: YYYY-MM-DD.md)
- Wiki page types (6 types ban đầu)
- Frontmatter bắt buộc theo loại file

SOT ownership và cross-call rules nằm trong pk-shared references.

### Schema tự tiến hóa

SCHEMA.md ban đầu có 6 wiki types + execution structure. pk-distill/pk-consult phát hiện "khuôn không vừa" → ghi schema-signals.md. pk-lint evolve mode kiểm tra ngưỡng → đề xuất thêm type hoặc tái cấu trúc. User duyệt → migrate.

### Reversibility

Không có hard delete. Mọi thao tác destructive đi qua archive/ (5 subfolder: actions, inbox, knowledge, skills, workflows). Khôi phục qua pk-lint fix mode (restore).

## pk-init flow

Khi chưa có `.cockpit/`:

1. Tạo toàn bộ thư mục `.cockpit/` + SCHEMA.md + registries rỗng + index rỗng
2. Hỏi tools (công cụ tương tác) → ghi tools.md
3. Hỏi objective + KR/KI + constraints → ghi objective.md (OPTIONAL, skip nếu chưa sẵn sàng)
4. Attach hướng dẫn vào CLAUDE.md (đọc `.cockpit/` khi bắt đầu session)

Nếu user skip objective: tạo objective.md rỗng, plan.md chưa tạo. Hệ thống hoạt động ở knowledge-only mode, thêm objective sau bằng pk-init update.

## pk-shared references

6 reference files, không chạy độc lập:

| File | Nội dung |
|------|----------|
| snapshot-contract.md | Định nghĩa context snapshot duy nhất mọi skill đều đọc |
| sot-ownership.md | Bảng field → skill ghi |
| metrics.md | Công thức metrics (KR%, KI status, trend, action health) |
| quality-gate.md | 3 check (cụ thể? giả định ẩn? mâu thuẫn?) |
| cross-call-rules.md | 4 lớp, hướng gọi cho phép |
| schemas.md | Format inbox, action, log, registries |

## Skill files cần viết

| Skill | Files |
|-------|-------|
| pk-harness | SKILL.md + references/flows.md |
| pk-init | SKILL.md |
| pk-analyze | SKILL.md |
| pk-plan | SKILL.md |
| pk-track | SKILL.md |
| pk-capture | SKILL.md |
| pk-reflect | SKILL.md |
| pk-distill | SKILL.md |
| pk-consult | SKILL.md |
| pk-lint | SKILL.md |
| pk-shared | 6 reference files |

Tổng: 10 SKILL.md + 1 flows.md + 6 references = **17 files**.

## Quyết định thiết kế

| # | Quyết định | Lý do |
|---|-----------|-------|
| 1 | `.cockpit/` thay vì `.project/` | Tránh xung đột với IDE/build tools. Gợi đúng ý "trung tâm điều khiển". |
| 2 | Per-project (không global) | Mục tiêu gắn liền dự án cụ thể. Tri thức per-project dễ quản lý hơn. |
| 3 | Inbox thống nhất | Giảm ma sát: 1 nơi capture, phân luồng tự động. |
| 4 | pk-query + pk-run → pk-consult | Cùng nhu cầu "tham vấn tri thức", chỉ khác đầu ra. Auto-detect mode dựa ngữ cảnh. |
| 5 | pk-retro gộp vào pk-reflect | Reflect (AAR deep) bao hàm retro (session extraction). Default light. |
| 6 | pk-analyze thuần read-only | User gọi "dashboard" kỳ vọng không side effect. Auto-fix thuộc pk-lint. |
| 7 | Cross-call 1 chiều 4 lớp | Cho phép skill gọi nhau, ngăn circular dependency. |
| 8 | 1 snapshot + on-demand | Maintain 1 chỗ thay vì 10 preload configs riêng. |
| 9 | Giữ SOT ownership nghiêm ngặt | 1 field = 1 skill ghi. Cross-module integration qua cross-call, không bypass. |
| 10 | Lesson chín dần (wiki → promote → skill) | Cho phép bài học tích lũy bằng chứng (≥3 usage) rồi promote. |
| 11 | Promote giữ stub + redirect | Không mất entry trong index, cross-link không gãy, pk-consult tự redirect. |
| 12 | Lesson usage: hybrid log + cache | pk-consult ghi log (SOT sạch), pk-distill cache count trong index.md (đọc nhanh). |
| 13 | resources.md → tools.md + objective constraints + knowledge concepts | Tools = công cụ tương tác (vận hành). Tài liệu = knowhow (tri thức). Capacity/budget = ràng buộc mục tiêu. |
| 14 | Tools auto-load concept pages | Khi match keyword → tự đọc chi tiết, không hỏi user. An toàn vì read-only. |
| 15 | Init không bắt buộc objective | Knowledge-only mode cho phép. Thêm objective sau. |
| 16 | Bỏ closure flow | Xong thì dừng. Dữ liệu vẫn nằm đó. Cần bài học thì chạy pk-reflect. |
| 17 | SCHEMA.md chỉ chứa quy ước dữ liệu | SOT ownership và cross-call rules thuộc pk-shared (ít thay đổi, quy tắc vận hành). |
| 18 | pk-lint 3 mode (check/fix/evolve) | Giảm cognitive load từ 6 mode. User chỉ cần nhớ 3. |
| 19 | pre_confirmed cho deep review | User duyệt batch 1 lần, không bị hỏi lại bởi pk-plan/pk-init. |
| 20 | pk-harness: snapshot → thu hẹp → 1 câu clarify | Không đoán mò, không chạy sai skill. |
| 21 | Concept page: minimal bắt buộc + optional | Không ép template cứng. Chỉ giữ section có nội dung. |
| 22 | domain=both: pk-capture tách ngay | Mỗi item tách viết lại cho đúng ngữ cảnh, link qua related_inbox. |
| 23 | Log theo ngày, 3 type, có source_skill | Gọn, dễ scan. 1 file/ngày không quá lớn. |
| 24 | archive/ 5 subfolder | Mọi thứ có thể archive đều có chỗ (actions, inbox, knowledge, skills, workflows). |
| 25 | pk-shared 6 reference files | Canonical 1 chỗ, không scatter. Sửa 1 chỗ, mọi skill cập nhật. |
| 26 | Graceful degradation theo file | Skill tự detect, không crash khi thiếu file. Hướng dẫn user bước tiếp theo. |
| 27 | tools.md bảng 4 cột với Interface | Hệ thống biết ngay cách tương tác: CLI, MCP, API, manual. |
