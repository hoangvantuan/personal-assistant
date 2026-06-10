# Shared Schemas

Schema dùng chung giữa nhiều skill. Canonical duy nhất.

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
effort: s|m|l|xl
notes: "..."
---
```

Body: `## DoD`, `## Output/Deliverable`, `## Checkpoints` (effort xl).

## Log format

Mỗi ngày 1 file: `log/YYYY-MM-DD.md`. Mỗi entry:

```yaml
---
timestamp: 2026-06-10T14:30
type: tracking|review|knowledge-activity
source_skill: pk-track|pk-consult|pk-distill|pk-lint
---
```

- `tracking`: cập nhật KR/action
- `review`: deep review, root cause
- `knowledge-activity`: create/update page, query, run skill, lint fix

## Knowledge page format

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

Lesson bổ sung: `## Kỳ vọng vs Thực tế`, `## Nguyên nhân gốc`, `## Hành động hệ thống`.

## Knowledge index format

```markdown
# Knowledge Index

| Slug | Type | Title | Tags | Status | Usage | Updated |
|------|------|-------|------|--------|-------|---------|
```

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

## raw/ convention

`raw/` CHỈ chứa file thô chưa xử lý: drafts, inbox overflow, file chờ phân loại, nguyên văn nguồn từ pk-capture.

KHÔNG thuộc raw/:
- Template tái sử dụng → `knowledge/resource-*.md`
- Tài liệu tham khảo → `knowledge/resource-*.md`
- File đã phân loại xong → inbox/ hoặc knowledge/
