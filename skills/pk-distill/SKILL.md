---
name: pk-distill
description: "Đúc kết inbox thành tri thức có cấu trúc: knowledge page, skill, hoặc workflow. Ưu tiên cập nhật cái có sẵn hơn tạo mới. Dùng khi user nói 'đúc kết', 'xử lý inbox', 'gộp bài học thành wiki', 'promote', hoặc khi inbox knowledge đã tích đủ cần chưng cất thành tài liệu chính thức."
---

# PK Distill: Đúc kết inbox knowledge → tri thức có cấu trúc

> **QUY TẮC SỐ 1: LUÔN đọc `knowledge/index.md` + `skills/registry.md` + `workflows/registry.md` TRƯỚC KHI xử lý. Ưu tiên cải tiến cái cũ hơn tạo mới.**

## Precondition

1. `.cockpit/` tồn tại.
2. `inbox/` có ít nhất 1 item `domain: knowledge`, `status: pending`.
3. Inbox rỗng → "Không có gì để đúc kết." Dừng.

## Bảng quyết định hành động

| Tình huống | Hành động |
| --- | --- |
| Chưa có gì liên quan | **TẠO MỚI** page |
| Đã có page, bổ sung | **CẬP NHẬT** page cũ |
| Đã có page, thay thế | **SỬA** page cũ, deprecated cách cũ |
| Mâu thuẫn chưa rõ | **CẬP NHẬT** thêm `## Mâu thuẫn đang mở`, hạ `confidence: low` |
| Đã có skill/workflow, thiếu/thừa bước | **REFINE** skill/workflow |
| Nhiều page nhỏ cùng chủ đề | **GỘP** thành 1 page |
| Không đáng lưu | **BỎ QUA**: move inbox → `archive/inbox/` |

## Flow

### Bước 1: Đọc registries

- `knowledge/index.md` (đã có trong snapshot)
- `skills/registry.md` (đã có trong snapshot)
- `workflows/registry.md` (đã có trong snapshot)

### Bước 1.5: Quét tín hiệu promote

Đọc `schema-signals.md`, lấy dòng `promote-candidate` trong "Đang chờ xử lý".
- Gom theo slug, đếm phiếu
- **Ngưỡng: >= 3 phiếu cùng slug** → ứng viên promote
- Dưới ngưỡng → để lại

### Bước 2: Đọc inbox

Liệt kê inbox `domain: knowledge`, `status: pending`. Đọc nội dung.

### Bước 3: Phân tích + đề xuất

Mỗi item:
1. **Phân loại**: knowledge page (6 types) hay skill hay workflow?
2. **Tìm trùng** (grep nội dung thật):
   ```bash
   grep -ril "<từ khoá>" .cockpit/knowledge .cockpit/skills .cockpit/workflows
   ```
3. **Quyết định hành động** (bảng trên)
4. **Item type lesson**: "Hành động hệ thống" phải sinh đề xuất KÈM

Trình đề xuất:
```
📥 inbox/[tên-file].md
   → Hành động: TẠO MỚI / CẬP NHẬT / ...
   → Page đích: knowledge/[type]-[slug].md
   → Preview: [1-2 câu]
```

### Bước 3.5: Phát tín hiệu strain

Phát hiện "khuôn không vừa" → ghi `schema-signals.md`:
- `no-fit-type`: item không vừa 6 wiki types
- `adhoc-section`: section tự chế lặp >= 2 page cùng type

### Bước 4: User duyệt

Từng item: đồng ý / sửa / bác bỏ.

### Bước 5: Thực thi

1. **Tạo/cập nhật page** theo format (`../pk-shared/references/schemas.md`)
2. **Cập nhật registry**: knowledge/index.md, skills/registry.md, workflows/registry.md
3. **Cross-referencing**: `related: [...]` + `[[slug]]` links
4. **Rewrite inbound link** khi GỘP/deprecate
5. **Lifecycle metadata**: status, confidence
6. **Cache usage_count**: đọc log, cập nhật Usage trong knowledge/index.md

### Bước 5.5: Thực thi promote (nếu được duyệt)

1. Tạo skill/workflow từ wiki page nguồn
2. Frontmatter: `promoted_from: [[slug-nguồn]]`
3. Page nguồn: thêm "Đã nâng thành skill: [[slug-mới]]", giữ page (stub redirect)
4. Cập nhật registry
5. Cắt promote-candidate từ "Đang chờ xử lý" → "Đã xử lý"

### Bước 6: Dọn inbox

Move items đã xử lý → `archive/inbox/` (KHÔNG xoá cứng).

### Bước 7: Log

Append `log/YYYY-MM-DD.md`:
```
---
timestamp: YYYY-MM-DDTHH:mm
type: knowledge-activity
source_skill: pk-distill
---
Distill: tạo N, cập nhật M, gộp K, bỏ qua L
```

## Promote lifecycle

```
Phát hiện (reflect/track) → inbox/ (domain=knowledge, type=lesson)
→ pk-distill xử lý:
    ├── Domain fact → knowledge/lesson-*.md
    ├── Process pattern → knowledge/pattern-*.md
    └── Tool capability → knowledge/pattern-*.md
```

Wiki → Skill promote:
1. pk-consult ghi usage vào log mỗi khi match page
2. pk-distill cache usage_count trong knowledge/index.md
3. usage_count >= 3 + type pattern/lesson → gợi ý promote
4. Promote: nội dung → skills/ hoặc workflows/, page gốc thành stub redirect
5. pk-consult tìm stub → tự redirect sang skill

## Quy tắc cứng

1. LUÔN đọc registry TRƯỚC xử lý.
2. Ưu tiên cập nhật hơn tạo mới.
3. Không xoá inbox chưa duyệt.
4. Giữ tất cả thông tin có giá trị khi gộp.
5. Archive, không hard delete.
6. **Tách monolithic**: Khi page > 150 dòng hoặc chứa > 3 chủ đề không liên quan, đề xuất tách thành nhiều page. Mỗi page 1 chủ đề, link qua `related`.
7. **Lightweight entry cho tài liệu ngoài cockpit**: Khi tài liệu lớn nằm ở root workspace (được nhiều nơi tham chiếu), tạo concept page tóm tắt (1-2 câu) + link trỏ về file gốc. Không copy, không di chuyển.
