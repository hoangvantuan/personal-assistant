---
name: pk-capture
description: "Ghi nhanh 1 mẩu rời (ý tưởng, bài học, task) vào inbox. Tự phân loại execution hay knowledge. Dùng khi user nói 'ghi lại', 'note', 'nhớ cái này', 'lưu bài học', 'thêm vào inbox', 'ghi chú', hoặc bất kỳ lúc nào cần bắt một mẩu thông tin nhanh mà chưa cần xử lý ngay. KHÔNG dùng khi cần quét cả phiên làm việc (dùng pk-reflect) hoặc đúc kết cả inbox (dùng pk-distill)."
---

# PK Capture: Ghi nhanh vào inbox thống nhất

Chạy inline. Phân loại type + domain, ghi `inbox/`. CHỈ ghi `inbox/`, `raw/` và append `log/`. KHÔNG ghi thẳng vào actions, knowledge, skills, workflows.

## Ensure snapshot (bắt buộc, đầu flow)

1. **Precondition**: `.cockpit/` tồn tại. Thiếu → route `pk-init` (mode new), KHÔNG nạp gì thêm, dừng.
2. **Idempotent qua marker**: phiên CHƯA có marker `SNAPSHOT_LOADED` → tự nạp full theo Snapshot Contract
   (`../pk-shared/references/snapshot-contract.md`) rồi đặt marker `SNAPSHOT_LOADED`. Đã có marker (vd
   harness Phase 1 đã đặt) → skip, không đọc lại.

## Tiêu chí phân loại

Phần tiêu chí domain trích từ `../pk-shared/references/schemas.md`. Khi lệch, schemas.md thắng.

### Type

| Type | Tín hiệu | Domain |
| --- | --- | --- |
| `action` | Động từ hành động, có output | execution |
| `blocker` | "không thể", "đang chờ", "bị kẹt" | execution |
| `resource` | Tool/tài liệu mới | hỏi user |
| `thought` | Ý tưởng mở, chưa rõ scope | hỏi user |
| `decision` | "quyết định", "chọn X thay Y" | knowledge |
| `pattern` | Cách giải quyết lặp lại | knowledge |
| `troubleshooting` | "lỗi", "fix", "root cause" | knowledge |
| `concept` | Thuật ngữ, định nghĩa | knowledge |
| `lesson` | "bài học", "rút kinh nghiệm", kỳ vọng lệch thực tế | knowledge |
| `candidate-skill` | Thao tác cụ thể, tái dùng | knowledge |
| `candidate-workflow` | Chuỗi bước, quy trình | knowledge |

### Domain tự động

- `action`, `blocker` → domain=execution
- `decision`, `pattern`, `concept`, `troubleshooting`, `lesson`, `candidate-skill`, `candidate-workflow` → domain=knowledge
- `resource`, `thought` → hỏi user

### domain=both

**Tiêu chí phát hiện** (canonical: `../pk-shared/references/schemas.md`, mục "domain=both"): item là domain=both khi CÓ ĐỒNG THỜI (a) **động từ hành động có output** (tính execution) VÀ (b) **tri thức tái dùng được** (tính knowledge). Thiếu một trong hai thì chọn domain duy nhất phù hợp, không tách.

Khi xác nhận domain=both: tách ngay thành 2 items riêng biệt, link qua `related_inbox`. Item execution focus action/status. Item knowledge focus nội dung tri thức.

**Hợp đồng 2 chiều**: ghi `related_inbox` ở CẢ HAI item (item A điền tên file của item B, item B điền tên file của item A). Không để 1 chiều.

## Chế độ capture

### 1. Từ cuộc trao đổi (mặc định)

Quét conversation history. Tìm khoảnh khắc có tín hiệu knowhow hoặc action item.

### 2. Từ nguồn ngoài

User cung cấp file/link. Đọc, áp dụng cùng tiêu chí. Nguồn lớn (>500 dòng hoặc >= 3 file): xem `references/batch-ingest.md`.

### 3. Từ phiên reflect

pk-reflect bàn giao xong gọi flow này từ Bước 2 (trình candidates), với `captured_from: reflect-light` hoặc `reflect-deep` (xem Handoff Protocol trong `../pk-shared/references/cross-call-rules.md`).

## Flow

### Bước 1: Quét nguồn

Đọc nguồn, lọc phần có tín hiệu.

### Bước 2: Trình danh sách candidates

Mỗi item: emoji + type + domain + tóm tắt 1-2 câu.

```
1. 📋 [execution] action: Viết test cho module auth
2. 🔧 [knowledge] pattern: Retry với exponential backoff + jitter
3. 🎓 [knowledge] lesson: Estimate thiếu buffer kiểm thử
```

### Bước 3: User duyệt

Giữ / sửa / bỏ. Cho phép thay type, domain, thêm item bỏ sót.

### Bước 4: Ghi vào inbox

1. Ghi raw vào `raw/YYYY-MM-DD-slug.md` (nguyên văn nguồn, immutable)
2. Tạo `inbox/YYYY-MM-DD-HHmm-slug.md` theo schema (`../pk-shared/references/schemas.md`)
3. Set `source_file: raw/YYYY-MM-DD-slug.md`

Khi không có objective: `related_kr` và `related_action` để null, domain mặc định = knowledge.

### Bước 5: Ghi log

Append `log/YYYY-MM-DD.md`:
```
---
timestamp: YYYY-MM-DDTHH:mm
type: knowledge-activity
source_skill: pk-capture
---
Capture N items (E execution, K knowledge) từ [nguồn]
```

### Bước 6: Xác nhận

Báo user. Gợi ý:
- domain=execution: "Xử lý khi track tiếp."
- domain=knowledge: "Chạy pk-distill để đúc kết."

## Quy tắc cứng

1. Capture CHỈ ghi `inbox/`, `raw/` và append `log/`. KHÔNG ghi actions, knowledge, skills, workflows.
2. Luôn trình candidates cho user duyệt trước ghi.
3. Luôn lưu raw (cả khi nguồn là hội thoại).
4. domain=both tách ngay thành 2 items, link qua `related_inbox` 2 chiều (cả 2 item đều điền field này).
5. Khi không có objective, domain mặc định = knowledge.
