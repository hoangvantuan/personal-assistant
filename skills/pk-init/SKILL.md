---
name: pk-init
description: "Tạo/sửa objective, KR, KI, constraints, tools, khởi tạo .cockpit/. Trigger: init, khởi tạo dự án, tạo mục tiêu, sửa mục tiêu, thêm tool, sửa capacity."
---

# PK Init: Khởi tạo + cập nhật objective, tools, constraints

SOT chính: `objective.md` và `tools.md`.

## Modes

| Mode | Trigger | Mô tả |
| --- | --- | --- |
| `new` | Chưa có `.cockpit/` | Tạo toàn bộ cấu trúc + SCHEMA.md + registries |
| `update-objective` | Sửa objective/KR/KI/period/constraints | Sửa objective.md |
| `update-tools` | Sửa tools (thêm/bỏ công cụ) | Sửa tools.md |

## SOT quyền ghi

| File | Fields |
| --- | --- |
| objective.md | Objective text, KR/KI targets, period, status, constraints |
| tools.md | Bảng tools 4 cột |
| SCHEMA.md | Chỉ khi mode new (tạo lần đầu) |

> Subset của bảng canonical `../pk-shared/references/sot-ownership.md`.

## Nguyên tắc

- Hỏi từng câu một, không hàng loạt.
- BẮT BUỘC confirm bảng trước ghi.
- Quality Gate 3 câu trước mỗi follow-up (`../pk-shared/references/quality-gate.md`).
- Snapshot Contract (`../pk-shared/references/snapshot-contract.md`): idempotent, qua harness đã có, chạy lẻ tự nạp.
- Solo only: 1 user, 1 objective.
- Đề xuất + lý do, user quyết.

## Flow: mode new

### Bước 1: Tạo cấu trúc thư mục

```bash
mkdir -p .cockpit/{actions,inbox,raw,log,archive/{actions,inbox,knowledge,skills,workflows},knowledge,skills,workflows}
```

### Bước 2: Sinh SCHEMA.md

Chứa quy ước dữ liệu:
- Cấu trúc thư mục `.cockpit/`
- Quy ước naming (actions: AXXX-slug.md, inbox: YYYY-MM-DD-HHmm-slug.md, knowledge: {type}-{slug}.md, log: YYYY-MM-DD.md)
- Wiki page types (6 types: decision, pattern, concept, troubleshooting, lesson, resource)
- Frontmatter bắt buộc theo loại file

### Bước 3: Sinh registries rỗng

- `knowledge/index.md`: bảng 7 cột (Slug, Type, Title, Tags, Status, Usage, Updated)
- `skills/registry.md`: bảng 6 cột (Skill, Mô tả, Khi nào dùng, Version, Tags, Cập nhật)
- `workflows/registry.md`: bảng 7 cột (Workflow, Mô tả, Khi nào dùng, Skills dùng, Version, Tags, Cập nhật)
- `schema-signals.md`: header + 2 section (Đang chờ xử lý / Đã xử lý)

### Bước 4: Hỏi tools

Hỏi user: "Dự án dùng công cụ gì? (CLI, MCP, API, manual)". Ghi vào `tools.md`:

```markdown
# Tools

| Name | Purpose | Interface | Usage |
|------|---------|-----------|-------|
```

### Bước 5: Hỏi objective (OPTIONAL)

Hỏi user: "Mục tiêu dự án là gì? (Bỏ qua nếu chưa sẵn sàng, chạy knowledge-only mode)".

Nếu user cung cấp:
- Thu thập objective text + KR/KI + constraints (capacity, budget, gaps/risks)
- Ghi `objective.md`

Nếu user skip:
- Tạo `objective.md` rỗng (chỉ frontmatter `status: empty`)
- KHÔNG tạo `plan.md`
- Hệ thống hoạt động ở knowledge-only mode

### Bước 6: Attach hướng dẫn vào CLAUDE.md

Tìm CLAUDE.md tại root workspace. Idempotent: grep trước, chỉ append nếu chưa có.

Nội dung: hướng dẫn đọc `.cockpit/` khi bắt đầu session.

## Flow: mode update-objective

1. Đọc objective.md hiện tại
2. Hỏi user thay đổi gì
3. Quality Gate mỗi follow-up
4. Confirm bảng diff
5. Ghi file

## Flow: mode update-tools

1. Đọc tools.md hiện tại
2. Hỏi user thay đổi gì
3. Confirm
4. Ghi file

## Quy tắc

- KHÔNG ghi file trước confirm.
- KR không SMART → chỉ rõ thiếu gì + gợi ý sửa.
- Không tạo plan.md hay action files (việc pk-plan).
- Không sửa action status (việc pk-track).
- Không tạo/sửa knowledge pages (việc pk-distill).
