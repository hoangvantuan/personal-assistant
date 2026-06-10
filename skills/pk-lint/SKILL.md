---
name: pk-lint
description: "Rà soát sức khoẻ hệ thống .cockpit/. 3 mode: check (health check read-only), fix (sửa + rebuild index + restore), evolve (schema evolution). Trigger: lint, kiểm tra, dọn dẹp, audit, rebuild index, restore, schema review, metrics."
---

# PK Lint: Rà soát + sửa chữa hệ thống

3 mode: **check** (mặc định, read-only), **fix** (consolidation + rebuild + restore), **evolve** (schema-review).

## Mode check (health check, read-only)

### Hạng mục kiểm tra

**1. Registry sync**
- knowledge/index.md: mỗi entry → file tồn tại. File trong knowledge/ không trong index → báo.
- skills/registry.md, workflows/registry.md: tương tự.

**2. Link integrity**
- Tìm `[[...]]` trong body pages
- Resolve: tìm file `{type}-{slug}.md` trong knowledge/, skills/, workflows/
- Link hỏng → báo

**3. Frontmatter check**
- Mọi page: `type`, `title`, `status`, `updated`
- Knowledge: thêm `confidence`
- Skill: thêm `version`, `trigger`, `input`, `output`
- Workflow: thêm `version`, `trigger`
- Thiếu → báo cụ thể

**4. Inbox backlog**
- Tuổi theo `captured_at`. > 7 ngày → cảnh báo. Phân execution vs knowledge.

**5. Reachability audit**
- File trong `.cockpit/` không neo vào SOT → báo mồ côi.

**6. Knowledge health metrics**
- Page count, freshness, orphans, usage distribution

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

Sinh lại file dẫn xuất từ frontmatter:
1. Quét knowledge/*.md → sinh knowledge/index.md
2. Quét skills/*.md → sinh skills/registry.md
3. Quét workflows/*.md → sinh workflows/registry.md

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
2. Quét sống: đếm page/type, cụm tag, file phình
3. Áp ngưỡng (bảo thủ: thà bỏ sót hơn báo nhiễu)
4. Sinh đề xuất diff
5. User duyệt
6. Migrate: bump version, rewrite link, rebuild index

### 4 loại thay đổi

- Thêm page type (cụm tag >= N tín hiệu)
- Đổi layout (type phình → subfolder)
- Đổi format (section adhoc lặp → template)
- Thêm mục SCHEMA (thuật ngữ lặp → glossary)

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
