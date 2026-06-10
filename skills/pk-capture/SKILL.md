---
name: pk-capture
description: "Ghi nhanh vào inbox thống nhất. Phân loại domain tự động (execution vs knowledge). Trigger: capture, ghi nhanh, note, lưu lại, ghi vào inbox, nhặt knowhow, lưu bài học."
---

# PK Capture: Ghi nhanh vào inbox thống nhất

Chạy inline. Phân loại type + domain, ghi `inbox/`. CHỈ ghi inbox, KHÔNG ghi thẳng vào actions, knowledge, skills, workflows.

## Precondition

Kiểm tra `.cockpit/` tồn tại. Thiếu → route pk-init.

## Tiêu chí phân loại

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

Khi item vừa có tính execution vừa có tính knowledge: tách ngay thành 2 items riêng biệt, link qua `related_inbox`. Item execution focus action/status. Item knowledge focus nội dung tri thức.

## Chế độ capture

### 1. Từ cuộc trao đổi (mặc định)

Quét conversation history. Tìm khoảnh khắc có tín hiệu knowhow hoặc action item.

### 2. Từ nguồn ngoài

User cung cấp file/link. Đọc, áp dụng cùng tiêu chí. Nguồn lớn (>500 dòng hoặc >= 3 file): xem `references/batch-ingest.md`.

### 3. Từ phiên reflect

pk-reflect phỏng vấn xong gọi flow này từ Bước 2 (trình candidates), với `captured_from: reflect`.

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

1. Capture CHỈ ghi `inbox/` và `raw/`. KHÔNG ghi actions, knowledge, skills, workflows.
2. Luôn trình candidates cho user duyệt trước ghi.
3. Luôn lưu raw (cả khi nguồn là hội thoại).
4. domain=both tách ngay thành 2 items, link qua `related_inbox`.
5. Khi không có objective, domain mặc định = knowledge.
