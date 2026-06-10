---
name: pk-consult
description: "Tham vấn tri thức. Auto-detect: cần thông tin → query (read-only), cần hành động → run (thực thi skill/workflow). Trigger: hỏi kho, tìm trong knowledge, chạy skill, áp dụng workflow, tra cứu knowhow."
---

# PK Consult: Tham vấn tri thức

Auto-detect mode dựa ngữ cảnh:
- **Query** (read-only): tra cứu knowledge, trả lời câu hỏi từ kho
- **Run** (thực thi): chạy skill/workflow đã đúc kết

## Auto-detect logic

| Tín hiệu | Mode |
| --- | --- |
| "hỏi kho", "kho có gì về X", "tra cứu", "tìm" | query |
| "chạy skill", "áp dụng workflow", "làm theo" | run |
| Mô tả task cụ thể + có skill/workflow khớp | run |
| Mô tả task + không có skill/workflow | query (tìm knowledge liên quan) |

## Mode Query

### Bước 1: Tìm page liên quan

1. Đọc registries (đã có trong snapshot)
2. Rút 3-5 từ khoá. Grep:
   ```bash
   grep -ril "<từ khoá>" .cockpit/knowledge .cockpit/skills .cockpit/workflows
   ```
3. Đọc page hit

### Bước 2: Tổng hợp câu trả lời

- Trả lời trực tiếp từ page
- Trích dẫn `[[slug]]`
- Không có → "chưa có knowhow về việc này"

### Bước 3: Phát tín hiệu

**query-miss** vào `schema-signals.md` khi:
- Chắp >= 3 page mới trả lời được, HOẶC
- KHÔNG page nào trả lời trực tiếp

**promote-candidate** khi:
- Câu hỏi thao tác ("làm sao...", "các bước...") VÀ
- Trả lời dựa chủ yếu vào 1 page pattern/troubleshooting

### Bước 4: Ghi usage log

Append `log/YYYY-MM-DD.md`:
```
---
timestamp: YYYY-MM-DDTHH:mm
type: knowledge-activity
source_skill: pk-consult
---
Query: [câu hỏi rút gọn], match: [[slug-1]], [[slug-2]]
```

## Mode Run

### Nhịp 1: Tra (nếu input là mô tả task)

1. Đọc skills/registry.md + workflows/registry.md
2. Match task với cột "Khi nào dùng" (trigger) → "Mô tả" → "Tags"
3. Đúng 1 khớp → Nhịp 2. Nhiều → hỏi user chọn. Không có → "chưa có skill/workflow"

### Nhịp 2: Load

Mở file skill/workflow. Đọc HẾT nội dung.

### Nhịp 3: Làm theo

1. Thực hiện lần lượt các bước
2. Workflow gặp `→ Skill: [[X]]` → load skill con, đệ quy
3. Tôn trọng điều kiện rẽ nhánh

### Usage log (mode run)

```
---
timestamp: YYYY-MM-DDTHH:mm
type: knowledge-activity
source_skill: pk-consult
---
Run: [[slug-bó]] | ok / vướng: [1 câu]
```

## Auto-load concept pages từ tools.md

Khi task match keyword trong tools.md inventory → tự load concept page liên quan từ knowledge/. Read-only, không hỏi user.

## Quy tắc cứng

1. Query KHÔNG ghi vào knowledge. Đáng lưu → qua pk-capture → inbox.
2. Run KHÔNG sửa skill/workflow. Vướng → ghi usage log + gợi ý pk-distill refine.
3. Không bịa câu trả lời / bó không tồn tại.
4. Trích dẫn `[[slug]]` cho mọi tuyên bố từ page.
5. Run phát hiện làm theo wiki page → emit promote-candidate.
