---
name: pk-lint
description: "Kiểm tra sức khoẻ hệ thống .cockpit/: link hỏng, registry lệch, file mồ côi, schema cũ. 3 mode: check (rà soát read-only), fix (sửa + rebuild index), evolve (nâng cấp schema). Dùng khi user nói 'lint', 'kiểm tra hệ thống', 'dọn dẹp cockpit', 'audit', 'rebuild index', hoặc nghi ngờ dữ liệu cockpit bị lệch."
---

# PK Lint: Rà soát + sửa chữa hệ thống

3 mode: **check** (mặc định, read-only), **fix** (consolidation + rebuild + restore), **evolve** (schema-review).

## Mode check (health check, read-only)

Ranh giới: pk-analyze đếm số liệu tổng cho dashboard. pk-lint soi chi tiết từng file.

### Hạng mục kiểm tra

**1. Registry sync**
- knowledge/index.md: mỗi entry → file tồn tại. File trong knowledge/ không trong index → báo.
- skills/registry.md, workflows/registry.md: tương tự.

**2. Link integrity**
- Quét mọi cross-reference nội bộ cockpit: `[[slug]]`, `[text](path)`, frontmatter `related`, `source_file`
- Resolve: tìm file `{type}-{slug}.md` trong knowledge/, skills/, workflows/. Kiểm tra path đích tồn tại.
- Link hỏng → báo cụ thể (file nguồn, dòng, link hỏng, lý do)

**3. Frontmatter check** (spec: ../pk-shared/references/schemas.md)
- Mọi page: `type`, `title`, `status`, `updated`
- Knowledge: thêm `confidence`
- Skill: thêm `version`, `trigger`, `input`, `output`
- Workflow: thêm `version`, `trigger`
- Thiếu → báo cụ thể

**4. Inbox backlog**
- Tuổi theo `captured_at`, ngưỡng theo Inbox Aging trong ../pk-shared/references/schemas.md. Trong check report:
  - 7-30 ngày → liệt kê tồn đọng
  - > 30 ngày → cảnh báo
- Phân execution vs knowledge.

**5. Reachability audit**
- File trong `.cockpit/` không neo vào SOT → báo mồ côi.

**6. Knowledge health metrics**
- Page count, freshness, orphans, usage distribution theo ../pk-shared/references/metrics.md, KHÔNG chép công thức

### Output

```markdown
## Check Report - YYYY-MM-DD

### Registry lệch (N)
### Link hỏng (N)
### Thiếu frontmatter (N)
### Inbox tồn đọng (N)
### Reachability (N)
### Knowledge metrics

✅ Không có vấn đề (nếu clean)
```

## Mode fix

### Sub-mode: consolidation (deep audit)

Đọc NỘI DUNG tất cả pages. Rà soát:
- Mâu thuẫn nội dung giữa pages
- Pages chồng chéo → đề xuất gộp
- Pages lỗi thời (> 90 ngày không update) → hạ confidence hoặc archive
- Nhất quán thuật ngữ
- Workflow dependency outdated

Trình report → user duyệt → thực thi thay đổi.

### Sub-mode: rebuild-index

Sinh lại file dẫn xuất từ nguồn gốc (frontmatter + log), format theo spec ../pk-shared/references/schemas.md:
1. Quét knowledge/*.md → sinh knowledge/index.md
2. Quét skills/*.md → sinh skills/registry.md
3. Quét workflows/*.md → sinh workflows/registry.md

Cột Usage tái dẫn xuất từ log theo công thức ../pk-shared/references/metrics.md mục "Usage count (canonical)". Fallback: log thiếu thì giữ giá trị Usage cũ trong index. Cả frontmatter lẫn log đều thiếu thì để 0.

### Sub-mode: restore

Khôi phục page/bó từ `archive/`:
1. Xác định mục tiêu (slug hoặc liệt kê archive)
2. Move file về đích. KHÔNG ghi đè nếu xung đột.
3. Đổi `status` → `active`
4. Rebuild index
5. Log

### Fix tự động vs cần duyệt

**Tự fix (sổ sách dẫn xuất)**: registry thiếu/thừa entry so với file thật.
**Cần duyệt**: link hỏng, frontmatter thiếu, inbox tồn, gộp pages.

## Mode evolve (schema-review)

Tổng hợp tín hiệu "khuôn không vừa" → đề xuất diff lên SCHEMA.md.

### Flow

1. Đọc `schema-signals.md` (BỎ QUA promote-candidate)
2. Quét sống: đếm page/type, cụm tag, file phình (KHÔNG dùng cho glossary, xem ghi chú bên dưới)
3. Áp ngưỡng (bảo thủ: thà bỏ sót hơn báo nhiễu). Ngưỡng canonical: `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act".
4. Sinh đề xuất diff
5. User duyệt
6. Migrate: bump version, rewrite link, rebuild index
7. Move tín hiệu đã batch sang "Đã xử lý" trong schema-signals.md kèm kết quả (duyệt hoặc bác). Chạy cả khi user bác đề xuất.

### 4 loại thay đổi

Ngưỡng đầy đủ (emit-threshold + act-threshold) tại `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act".

| Thay đổi | Tín hiệu đọc |
| --- | --- |
| Thêm page type | `no-fit-type` |
| Đổi layout (tách subfolder) | (đếm trực tiếp page active) |
| Đổi format (template hoá section) | `adhoc-section` |
| Thêm glossary | `term-repeat` |

Ghi chú:
- query-miss là bằng chứng phụ, không tự kích hoạt thay đổi.
- Dưới ngưỡng: GIỮ tín hiệu, không xoá.
- **Glossary**: đọc tín hiệu `term-repeat` từ schema-signals.md thay cho "quét sống". pk-distill emit `term-repeat` khi xử lý nội dung (đáng tin hơn quét tĩnh).

## Log

```
---
timestamp: YYYY-MM-DDTHH:mm
type: knowledge-activity
source_skill: pk-lint
---
Lint [mode]: N vấn đề, M fixed
```

## Quy tắc

- Check mode thuần read-only.
- Fix: tách phát hiện khỏi quyết định. User duyệt nội dung, tự fix sổ sách.
- Evolve: không auto-migrate. Mọi batch gate qua user.
- Reversibility: không hard delete. Archive.
