# Shared Schemas

Schema dùng chung giữa nhiều skill. File này là bản canonical duy nhất. SKILL.md các skill chỉ trích subset liên quan. Khi subset lệch file này thì file này thắng. Sửa file này trước, subset theo sau.

## Phân vùng schemas.md và SCHEMA.md

Hai file phục vụ hai mục đích không giao nhau:

- `schemas.md` (file này): **bất biến lõi**, không project nào được thay đổi. Gồm: frontmatter bắt buộc, format inbox/action/log/objective/plan, 6 wiki type gốc, promote criteria, ngưỡng tín hiệu. Read-only ở runtime.
- `SCHEMA.md` (nằm trong `.cockpit/` của từng project, KHÔNG tồn tại trong repo này): **phần mở rộng project-specific**, do `pk-lint` evolve sinh ra. Gồm: type wiki mới ngoài 6 type gốc, section đã template hoá, glossary, ngưỡng tuỳ biến.

Hai vùng **không giao**: không có luật thắng-thua giữa chúng. Writer đọc lõi từ `schemas.md`, đọc mở rộng từ `SCHEMA.md`; `pk-lint` evolve chỉ ghi `SCHEMA.md` nên có hiệu lực ngay trong project.

> **`SCHEMA.md` = `schemas.md` + delta evolve; trong phạm vi project, vùng mở rộng do `SCHEMA.md` sở hữu.**

## Inbox item thống nhất

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

### Phân luồng domain tự động

- `action`, `blocker` → domain=execution
- `decision`, `pattern`, `concept`, `troubleshooting`, `lesson`, `candidate-skill`, `candidate-workflow` → domain=knowledge
- `resource`, `thought` → hỏi user

### domain=both

pk-capture tách ngay lúc tạo thành 2 items riêng biệt, link nhau qua `related_inbox`. Item execution focus vào action/status. Item knowledge focus vào nội dung tri thức.

**Tiêu chí phát hiện domain=both**: item là domain=both khi CÓ ĐỒNG THỜI (a) **động từ hành động có output** (tính execution) VÀ (b) **tri thức tái dùng được** (tính knowledge). Thiếu một trong hai thì chọn domain duy nhất phù hợp.

**Hợp đồng 2 chiều**: khi tách cặp domain=both, pk-capture ghi `related_inbox` ở CẢ HAI item (item A trỏ item B, item B trỏ item A). Khi xử lý 1 item trong cặp, đọc `related_inbox` để biết item cặp; xử lý nốt hoặc đánh dấu liên đới tránh bỏ rơi nửa cặp.

## Objective file format

SOT của pk-init. Frontmatter KHÔNG chứa `last_track_date` / `last_review_date` (2 field này thuộc `plan.md`).

```markdown
---
type: project|ongoing
status: empty|active|paused
period: "YYYY-MM-DD → YYYY-MM-DD"
review_cycle: 14
constraints: "capacity, budget, gaps/risks tóm tắt"
---

## Mục tiêu
(1-2 câu objective text.)

## Key Results

| ID | Mô tả | Baseline | Target | Current | Status |
|----|-------|----------|--------|---------|--------|

## Key Indicators

| ID | Mô tả | Ngưỡng | Current | Status |
|----|-------|--------|---------|--------|
```

- `type`: bỏ trống khi `status: empty` (knowledge-only mode).
- `review_cycle`: chỉ type ongoing. Đơn vị ngày, đề xuất mặc định 14.
- KR/KI Status: auto-compute theo `metrics.md`.

## tools.md format

SOT của pk-init. Snapshot preload TOÀN BỘ body.

```markdown
# Tools

| Name | Purpose | Interface | Usage |
|------|---------|-----------|-------|
```

`Interface`: CLI, MCP, API, hoặc manual. Keyword match với tool → auto-load concept page liên quan (xem `snapshot-contract.md`).

## plan.md format

pk-plan tạo milestones + Roadmap. pk-track ghi counters, `last_track_date`, `last_review_date`.

```markdown
---
counters:
  total_actions: 12
  completed: 4
last_track_date: YYYY-MM-DD | null
last_review_date: YYYY-MM-DD | null
---

## Milestones

| ID | Tên | Target |
|----|-----|--------|
| M1 | Research | 2026-05-20 |

## Roadmap

(Khối render theo Roadmap format bên dưới. Bảng dẫn xuất từ actions/.)
```

## Roadmap format

Mỗi milestone heading chứa bảng action:

````markdown
## Roadmap

### M1: Research (target: 2026-05-20)

| ID | Task | Deadline | Priority | Notes |
|----|------|----------|----------|-------|
| [A001](actions/A001-research-market.md) | Nghiên cứu thị trường | 2026-05-15 | high | |
````

Cột: ID (link), Task, Deadline, Priority, Notes. Sắp theo Priority → Deadline. Chỉ hiện action chưa done.

Roadmap là bảng dẫn xuất từ `actions/`, không phải SOT. Không sửa tay từng dòng. pk-plan và pk-track re-render sau khi ghi `actions/`.

Cột Deadline render từ field `due_date`.

## Action file format

Naming: `AXXX-slug.md`. Frontmatter:

```yaml
---
id: A001
title: "..."
key_result: KR1 | null
milestone: M1 | null
status: pending|doing|blocked|done
priority: critical|high|medium|low
due_date: YYYY-MM-DD
completed_date: YYYY-MM-DD | null
deps: [A002]
effort: s|m|l|xl
notes: "..."
---
```

- `deps`: danh sách action ID phụ thuộc. Mặc định rỗng `[]`.
- `completed_date`: set khi done (xem Archive Rules). Chưa done thì `null`.

Body: `## DoD`, `## Output/Deliverable`, `## Checkpoints` (effort xl).

## Log format

Mỗi ngày 1 file: `log/YYYY-MM-DD.md`. Mỗi entry:

```yaml
---
timestamp: 2026-06-10T14:30
type: tracking|review|knowledge-activity
source_skill: pk-track|pk-consult|pk-capture|pk-distill|pk-lint
---
```

- `tracking`: cập nhật KR/action
- `review`: deep review, root cause
- `knowledge-activity`: capture items, create/update page, query, run skill, lint fix

pk-init và pk-plan KHÔNG ghi log. Chủ ý: output của chúng tự là SOT. Enum `source_skill` phải đồng bộ với hàng `log/` trong `sot-ownership.md`.

## Knowledge page format

```yaml
---
type: concept|decision|pattern|troubleshooting|lesson|resource
title: "..."
tags: [...]
related: [...]
status: active|deprecated|archived|stub
redirect_to: "[[đích]]"
confidence: low|medium|high
pinned: false
updated: YYYY-MM-DD
---

## Tóm tắt
(BẮT BUỘC. 1-2 câu.)

## Khi nào dùng
(Optional.)

## Cách dùng
(Optional.)

## Ví dụ
(Optional.)

## Nguồn
(Optional.)
```

`pinned: true` → snapshot preload full body.

`status: stub` + `redirect_to: "[[đích]]"`: page đã được promote thành skill hoặc workflow. `redirect_to` trỏ tới slug đích. Snapshot đọc được ngay từ index mà không cần mở body page. `redirect_to` bắt buộc khi `status: stub`; bỏ trống khi status khác.

Lesson bổ sung: `## Kỳ vọng vs Thực tế`, `## Nguyên nhân gốc`, `## Hành động hệ thống`.

Mọi type có thể có section optional `## Mâu thuẫn đang mở`. Section này thuộc format chuẩn, KHÔNG tính là adhoc-section. Gỡ khi mâu thuẫn được giải quyết.

## schema-signals.md format

File `schema-signals.md` lưu tín hiệu "khuôn không vừa" từ quá trình xử lý. Bản ghi key-value cố định. Ai được ghi/đọc xem `sot-ownership.md`.

### Cấu trúc file

```markdown
# Schema Signals

## Đang chờ xử lý

<!-- Mỗi phiếu một block: -->
- type: <loại-phiếu>
  slug: <slug-của-page-hoặc-mô-tả-ngắn>
  source_skill: <pk-distill|pk-consult>
  timestamp: "YYYY-MM-DDTHH:mm"
  [trường bổ sung tuỳ loại phiếu]

## Đã xử lý

<!-- Phiếu đã batch hoặc bị bác, kèm kết quả: -->
- type: <loại-phiếu>
  slug: <slug>
  source_skill: <skill>
  timestamp: "YYYY-MM-DDTHH:mm"
  result: approved|rejected
  processed_at: "YYYY-MM-DDTHH:mm"
```

### 5 loại phiếu

| Loại phiếu | Emit bởi | Ý nghĩa |
| --- | --- | --- |
| `promote-candidate` | pk-consult, pk-distill | Page đủ điều kiện nâng thành skill/workflow |
| `query-miss` | pk-consult | Câu hỏi không có page nào trả lời trực tiếp |
| `no-fit-type` | pk-distill | Item không vừa 6 wiki type gốc |
| `adhoc-section` | pk-distill | Section ngoài format chuẩn, lặp >= 2 page cùng type |
| `term-repeat` | pk-distill | Thuật ngữ lặp >= 2 page (dữ liệu cho evolve glossary) |

### Quy tắc phiếu rejected

User bác bỏ promote → move các phiếu `promote-candidate` sang "Đã xử lý" kèm `result: rejected`. Chỉ gợi ý lại khi đủ 3 phiếu MỚI (không tính phiếu rejected cũ).

## Bảng ngưỡng emit/act (canonical)

Bảng duy nhất định nghĩa ngưỡng cho mọi loại tín hiệu. SKILL.md các skill chỉ **trỏ về đây**, không tự chép số.

| Loại tín hiệu | Emit-threshold (ai emit) | Act-threshold (ai act + hành động) |
| --- | --- | --- |
| `promote-candidate` | Mỗi lần pk-consult/pk-distill phát hiện điều kiện | >= 3 phiếu cùng slug (pk-distill propose promote) |
| `query-miss` | Mỗi lần query chắp >= 3 page hoặc không có page | Bằng chứng phụ, không tự kích hoạt thay đổi |
| `no-fit-type` | Mỗi lần pk-distill gặp item không vừa 6 type | >= 5 phiếu cùng cụm tag (pk-lint evolve: thêm page type) |
| `adhoc-section` | pk-distill: section ngoài chuẩn, lặp >= 2 page cùng type | >= 3 phiếu cùng section name (pk-lint evolve: template hoá section) |
| `term-repeat` | pk-distill: thuật ngữ lặp >= 2 page | >= 3 phiếu cùng thuật ngữ (pk-lint evolve: thêm glossary) |
| (layout) | (không có tín hiệu riêng, pk-lint đếm trực tiếp) | >= 15 page active cùng type (pk-lint evolve: tách subfolder) |

Ghi chú cột:
- **Emit-threshold**: ngưỡng để ghi phiếu vào schema-signals.md. Emit sớm để tích bằng chứng.
- **Act-threshold**: ngưỡng để pk-lint evolve đề xuất thay đổi SCHEMA.md. Act muộn khi đủ bằng chứng.
- Hai ngưỡng phục vụ hai giai đoạn khác nhau, **không ép chúng bằng nhau**.

## Promote criteria

Cơ chế nâng wiki page thành skill/workflow:

- Trigger DUY NHẤT: page đạt >= 3 phiếu `promote-candidate` trong `schema-signals.md`.
- `usage_count` KHÔNG phải trigger. Chỉ là bằng chứng phụ trình kèm khi duyệt.
- Type đủ điều kiện promote: `pattern`, `troubleshooting`, `lesson` (lesson chỉ khi mô tả quy trình lặp lại được).
- Promote chỉ là GỢI Ý. User duyệt mới thực hiện (pk-distill Bước 5.5).
- User bác bỏ → phiếu cũ bị cắt: move sang "Đã xử lý" kèm đánh dấu rejected. Chỉ gợi ý lại khi đủ 3 phiếu MỚI.

### Hai đường tạo skill/workflow

**Đường phiếu** (`promote-candidate`): NÂNG CẤP page **đã có**, đã chứng minh qua usage (đạt act-threshold, xem Bảng ngưỡng emit/act). Cơ chế: pk-distill Bước 1.5 phát hiện, Bước 5.5 thực thi.

**Đường candidate** (inbox type `candidate-skill` / `candidate-workflow`): ý định **tạo mới chủ động** (không phải nâng page đã có). pk-distill tạo trực tiếp (qua user duyệt) CHỈ KHI đủ cả ba: `trigger` + `input` + `output` trong nội dung; thiếu bất kỳ trường nào → tự hạ xuống wiki `pattern` page để chờ chín, có thể lên phiếu sau.

Hai đường KHÔNG giao: đường candidate không cần phiếu; đường phiếu không cần inbox candidate.

## Knowledge index format

```markdown
# Knowledge Index

| Slug | Type | Title | Tags | Status | Pinned | Usage | Updated | Redirect |
|------|------|-------|------|--------|--------|-------|---------|----------|
```

`Pinned` (true/false) và `Usage` là giá trị dẫn xuất. Pinned lấy từ frontmatter `pinned` của page. Usage tính từ log theo công thức tại `metrics.md`, mục "Usage count (canonical)". **Usage trong index là cache-hint, CÓ THỂ CŨ**; log là SOT thực (xem `sot-ownership.md` và `metrics.md`). Không dùng cột Usage trong index để ra quyết định promote: luôn recompute từ log.

`Redirect`: giá trị `[[đích]]` khi `status=stub`, rỗng nếu không. Snapshot đọc cột này để biết đích redirect ngay mà không cần mở body. Cột này được pk-lint rebuild-index sinh từ frontmatter `redirect_to` của page.

## Procedure block

Khối bước chuẩn để **Run** được. Định nghĩa canonical duy nhất tại đây; pk-consult Run và Promote criteria đều trỏ về mục này.

### Cấu trúc

Procedure block gồm:
- Các **bước tuần tự đánh số** (ví dụ: `1. Bước đầu tiên`)
- **Điều kiện rẽ nhánh** khi có (ví dụ: `Nếu X → Bước 3; ngược lại → Bước 4`)
- Marker **`→ Skill: [[X]]`** khi một bước ủy thác sang skill con (dùng trong workflow)

### Ràng buộc bắt buộc

- **Workflow**: procedure block là **body BẮT BUỘC**. Workflow không có procedure block là **không hợp lệ**: không thể Run, không thể promote.
- **Skill**: procedure block KHÔNG bắt buộc trong mọi skill; tuy nhiên skill/workflow không có procedure block sẽ bị báo "mới mô tả, chưa có quy trình" khi Run (xem pk-consult Mode Run).
- **Wiki page** (pattern/troubleshooting/lesson): được phép xuất hiện procedure block trong `## Cách dùng`, không bắt buộc.

### Ràng buộc Run/promote

Chỉ Run hoặc promote được target **CÓ** procedure block. Thiếu → KHÔNG bịa quy trình, báo "mới mô tả, chưa có quy trình" và dừng.

## Skill/Workflow file format

File trong `skills/` và `workflows/`. pk-distill tạo/sửa.

```yaml
---
type: skill|workflow
title: "..."
status: active|deprecated|archived
version: 1
trigger: "..."
input: "..."
output: "..."
skills_used: [slug-1, slug-2]
tags: [...]
updated: YYYY-MM-DD
---
```

- `skills_used`: chỉ workflow.
- `promoted_from: [[slug-nguồn]]`: optional, khi promote từ wiki page.

Mapping frontmatter → cột registry:

| Frontmatter | Cột registry |
| --- | --- |
| trigger | Khi nào dùng |
| skills_used | Skills dùng (chỉ workflow registry) |

Cột Version, Tags, Cập nhật render từ `version`, `tags`, `updated`.

## Skills registry format

```markdown
# Skill Registry

| Skill | Mô tả | Khi nào dùng | Version | Tags | Cập nhật |
|-------|--------|--------------|---------|------|----------|
```

## Workflows registry format

```markdown
# Workflow Registry

| Workflow | Mô tả | Khi nào dùng | Skills dùng | Version | Tags | Cập nhật |
|----------|--------|--------------|-------------|---------|------|----------|
```

## Inbox Aging

`staleness_days = today - captured_at` (compute on-the-fly).

| Phạm vi | Hành vi |
| --- | --- |
| <= 7 ngày | Bình thường |
| 7-30 ngày | Sort lên đầu |
| > 30 ngày | Cảnh báo "Inbox cũ >= 30 ngày" |

## Archive Rules

Khi action done:
1. Set `completed_date`
2. Dời file → `archive/actions/`
3. Xóa khỏi Roadmap
4. Cập nhật counters

5 subfolder archive: `archive/actions/`, `archive/inbox/`, `archive/knowledge/`, `archive/skills/`, `archive/workflows/`.

Naming conventions:
- Actions: `AXXX-slug.md`
- Inbox: `YYYY-MM-DD-HHmm-slug.md`
- Knowledge: `{type}-{slug}.md`
- Log: `YYYY-MM-DD.md`

### Bất biến unique-slug

**Slug phải duy nhất across MỌI type** (knowledge, skill, workflow). Một `[[slug]]` phân giải TẤT ĐỊNH về đúng 1 file trong `.cockpit/`. Không dùng cú pháp `[[type/slug]]`, giữ `[[slug]]` trần. Khi tạo file mới, kiểm tra trùng slug trước (pk-lint check báo vi phạm). Nếu slug đã tồn tại ở type khác, chọn slug khác biệt hơn.

## raw/ convention

`raw/` CHỈ chứa file thô chưa xử lý: drafts, inbox overflow, file chờ phân loại, nguyên văn nguồn từ pk-capture.

KHÔNG thuộc raw/:
- Template tái sử dụng → `knowledge/resource-*.md`
- Tài liệu tham khảo → `knowledge/resource-*.md`
- File đã phân loại xong → inbox/ hoặc knowledge/
