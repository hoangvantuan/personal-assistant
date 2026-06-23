---
name: pk-distill
description: "Đúc kết inbox thành tri thức có cấu trúc: knowledge page, skill, hoặc workflow. Ưu tiên cập nhật cái có sẵn hơn tạo mới. Dùng khi user nói 'đúc kết', 'xử lý inbox', 'gộp bài học thành wiki', 'promote', hoặc khi inbox knowledge đã tích đủ cần chưng cất thành tài liệu chính thức."
---

# PK Distill: Đúc kết inbox knowledge → tri thức có cấu trúc

> **QUY TẮC SỐ 1: LUÔN đọc `knowledge/index.md` + `skills/registry.md` + `workflows/registry.md` TRƯỚC KHI xử lý. Ưu tiên cải tiến cái cũ hơn tạo mới.**

## Ensure snapshot (bắt buộc, đầu flow)

1. **Precondition**: `.cockpit/` tồn tại. Thiếu → route `pk-init` (mode new), KHÔNG nạp gì thêm, dừng.
2. **Idempotent qua marker**: phiên CHƯA có marker `SNAPSHOT_LOADED` → tự nạp full theo Snapshot Contract
   (`../pk-shared/references/snapshot-contract.md`) rồi đặt marker `SNAPSHOT_LOADED`. Đã có marker (vd
   harness Phase 1 đã đặt) → skip, không đọc lại.
3. **Precondition nguồn việc** (đặc thù pk-distill): cần ít nhất 1 trong 2 nguồn:
   - (1) `inbox/` có item `domain: knowledge`, `status: pending`.
   - (2) Có page đạt act-threshold `promote-candidate` trong `schema-signals.md` (ngưỡng canonical: `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act").
   Cả hai rỗng → "Không có gì để đúc kết." Dừng.
4. Chỉ có (2) → chạy nhánh promote-only: Bước 1 → 1.5 → 4 → 5.5 → 7.

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
- Chạy cả khi inbox rỗng (nhánh promote-only)
- Gom theo slug, đếm phiếu
- Đạt act-threshold của `promote-candidate` (canonical: `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act") → ứng viên promote
- Dưới ngưỡng → để lại
- Trình ứng viên kèm **Usage recompute từ log** (công thức: `../pk-shared/references/metrics.md`, mục "Usage count (canonical)") làm BẰNG CHỨNG PHỤ. KHÔNG dùng cột Usage trong index (cache-hint, có thể cũ).

### Bước 2: Đọc inbox

Liệt kê inbox `domain: knowledge`, `status: pending`. Đọc nội dung.

Khi item có `related_inbox` (cặp domain=both): đọc item cặp để hiểu đủ ngữ cảnh. Khi xử lý item này, đánh dấu liên đới lên item cặp (tránh bỏ rơi nửa cặp). Item cặp thuộc domain=execution → thông báo cho pk-track biết để xử lý nốt khi track.

### Bước 3: Phân tích + đề xuất

Mỗi item:
1. **Phân loại** (theo thứ tự ưu tiên bên dưới)
2. **Tìm trùng** (grep nội dung thật):
   ```bash
   grep -ril "<từ khoá>" .cockpit/knowledge .cockpit/skills .cockpit/workflows
   ```
3. **Quyết định hành động** (bảng trên)
4. **Item type lesson**: "Hành động hệ thống" phải sinh đề xuất KÈM

**Phân loại chi tiết theo type:**

- **Type `thought`**: KHÔNG phải type lưu trữ cuối. Re-classify vào 1 trong 6 wiki type (`decision` / `pattern` / `concept` / `troubleshooting` / `lesson` / `resource`) hoặc discard nếu không đủ giá trị. KHÔNG tạo knowledge page type=thought.

- **Type `candidate-skill` / `candidate-workflow`** (đường candidate, canonical: `../pk-shared/references/schemas.md`, mục "Hai đường tạo skill/workflow"): đi đường candidate độc lập, KHÔNG cần phiếu `promote-candidate`. Kiểm: có đủ `trigger` + `input` + `output` không?
  - Đủ cả ba → đề xuất tạo skill/workflow trực tiếp kèm procedure block (canonical: `../pk-shared/references/schemas.md`, mục "Procedure block"). Cần user duyệt trước khi tạo.
  - Thiếu bất kỳ trường nào → hạ xuống tạo wiki `pattern` page thay thế. Ghi chú "chờ chín, có thể lên phiếu promote-candidate sau".

  **Phân biệt rõ với đường phiếu**: đường phiếu (`promote-candidate`, Bước 1.5/5.5) nâng page đã có, đã chứng minh qua usage. Đường candidate tạo mới từ ý định, chưa có page gốc.

Trình đề xuất:
```
📥 inbox/[tên-file].md
   → Hành động: TẠO MỚI / CẬP NHẬT / ...
   → Page đích: knowledge/[type]-[slug].md
   → Preview: [1-2 câu]
```

### Bước 3.5: Phát tín hiệu strain

Phát hiện "khuôn không vừa" → ghi `schema-signals.md`. Ngưỡng emit/act canonical: `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act".

- `no-fit-type`: item không vừa 6 wiki types
- `adhoc-section`: section ngoài format chuẩn (`../pk-shared/references/schemas.md`) VÀ ngoài `SCHEMA.md` (template đã evolve), lặp đạt emit-threshold cùng type. Section `## Mâu thuẫn đang mở` thuộc format chuẩn, KHÔNG tính.
- `term-repeat`: thuật ngữ lặp đạt emit-threshold (cung cấp dữ liệu cho pk-lint evolve thêm glossary thay cho "quét sống"). Ghi thêm trường `term: "<thuật-ngữ>"` vào phiếu.

### Bước 4: User duyệt

Từng item: đồng ý / sửa / bác bỏ.

### Bước 5: Thực thi

1. **Tạo/cập nhật page** theo format (`../pk-shared/references/schemas.md`)
2. **Cập nhật registry**: knowledge/index.md, skills/registry.md, workflows/registry.md
3. **Cross-referencing**: `related: [...]` + `[[slug]]` links
4. **Rewrite inbound link khi GỘP/deprecate**: grep ngược bắt buộc trước khi gộp/deprecate:
   ```bash
   grep -rl "[[slug-cũ]]" .cockpit/
   ```
   Rewrite mọi link tìm được. KHÔNG duy trì back-link index/danh sách inbound (tránh drift). pk-lint link-integrity là lưới an toàn bắt link mồ côi còn sót.
5. **Lifecycle metadata**: status, confidence
6. **Cache usage_count**: đọc log, cập nhật Usage trong knowledge/index.md (công thức: `../pk-shared/references/metrics.md`, mục "Usage count (canonical)"). Đây là cache-hint; log là SOT.

### Bước 5.5: Thực thi promote (nếu được duyệt)

1. **Recompute usage từ log** trước khi quyết định promote (công thức: `../pk-shared/references/metrics.md`, mục "Usage count (canonical)"). KHÔNG tin cột Usage trong index (cache-hint, có thể cũ).
2. **Kiểm procedure block của page nguồn** TRƯỚC khi tạo file (canonical: `../pk-shared/references/schemas.md`, mục "Procedure block" và "Promote criteria"). Page nguồn KHÔNG có procedure block (trong `## Cách dùng`) → KHÔNG tạo skill/workflow, báo "cần bổ sung quy trình (procedure block) trước khi promote". Dừng nhánh promote cho page này.
3. Tạo skill/workflow từ wiki page nguồn
4. Frontmatter theo Skill/Workflow file format (`../pk-shared/references/schemas.md`), gồm `promoted_from: [[slug-nguồn]]`
5. **Page nguồn: set frontmatter `status: stub` + `redirect_to: "[[slug-mới]]"`** (slug-mới là skill hoặc workflow đích). Đây là nguồn tất định cho snapshot và pk-consult. Có thể GIỮ thêm dòng marker body "Đã nâng thành skill: [[slug-mới]]" cho người đọc, nhưng frontmatter là nguồn tất định.
6. Cập nhật registry (bao gồm cột Redirect trong knowledge/index.md)
7. Cắt promote-candidate từ "Đang chờ xử lý" → "Đã xử lý"
8. User bác bỏ → cắt phiếu: move các phiếu đó sang "Đã xử lý" kèm đánh dấu rejected. Chỉ gợi ý lại khi đủ act-threshold phiếu MỚI (không tính phiếu rejected cũ).

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
Distill: tạo N, cập nhật M, gộp K, bỏ qua L, promote P
```

## Promote lifecycle

```
Phát hiện (reflect/track) → inbox/ (domain=knowledge, type=lesson)
→ pk-distill xử lý:
    ├── Domain fact → knowledge/lesson-*.md
    ├── Process pattern → knowledge/pattern-*.md
    └── Tool capability → knowledge/pattern-*.md
```

Wiki → Skill/Workflow promote (pipeline promote-candidate hợp nhất):
1. Trigger DUY NHẤT: page đạt act-threshold `promote-candidate` trong `schema-signals.md`
2. Tiêu chí + ngưỡng canonical: `../pk-shared/references/schemas.md`, mục "Promote criteria" và "Bảng ngưỡng emit/act"
3. `usage_count` KHÔNG phải trigger. Chỉ là bằng chứng phụ; recompute từ log trước khi trình, KHÔNG dùng cột index.
4. User duyệt tại Bước 5.5 → nội dung sang skills/ hoặc workflows/, page gốc set `status: stub` + `redirect_to: "[[slug-mới]]"` ở frontmatter (bao cả đích là skill lẫn workflow)
5. pk-consult phát hiện stub qua frontmatter `status: stub` / `redirect_to` → tự redirect sang đích

## Quy tắc cứng

1. LUÔN đọc registry TRƯỚC xử lý.
2. Ưu tiên cập nhật hơn tạo mới.
3. Không xoá inbox chưa duyệt.
4. Giữ tất cả thông tin có giá trị khi gộp.
5. Archive, không hard delete.
6. **Tách monolithic**: Khi page > 150 dòng hoặc chứa > 3 chủ đề không liên quan, đề xuất tách thành nhiều page. Mỗi page 1 chủ đề, link qua `related`.
7. **Lightweight entry cho tài liệu ngoài cockpit**: Khi tài liệu lớn nằm ở root workspace (được nhiều nơi tham chiếu), tạo concept page tóm tắt (1-2 câu) + link trỏ về file gốc. Không copy, không di chuyển.
