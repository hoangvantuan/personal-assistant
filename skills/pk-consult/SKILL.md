---
name: pk-consult
description: "Tham vấn tri thức. Auto-detect: cần thông tin → query (read-only), cần hành động → run (thực thi skill/workflow), cần truyền đạt → teach (soạn lộ trình đọc). Trigger: hỏi kho, tìm trong knowledge, chạy skill, áp dụng workflow, tra cứu knowhow, soạn onboarding, dạy lại X."
---

# PK Consult: Tham vấn tri thức

Auto-detect mode dựa ngữ cảnh:
- **Query** (read-only): tra cứu knowledge, trả lời câu hỏi từ kho
- **Run** (thực thi): chạy skill/workflow đã đúc kết
- **Teach** (soạn lộ trình): tạo reading curriculum từ knowledge base

## Auto-detect logic

| Tín hiệu | Mode |
| --- | --- |
| "hỏi kho", "kho có gì về X", "tra cứu", "tìm" | query |
| "chạy skill", "áp dụng workflow", "làm theo" | run |
| "soạn onboarding", "dạy lại X", "tổng hợp cho người mới", "lộ trình học" | teach |
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

## Mode Teach

Kích hoạt khi user yêu cầu kiểu "soạn onboarding", "dạy lại X cho người mới", "tổng hợp cho người khác học". Bản chất: query + sắp xếp output thành lộ trình đọc.

### Bước 1: Gom page theo chủ đề

Như Bước 1 query thường: đọc registries, grep từ khoá, đọc page hit.

### Bước 2: Xếp theo thang biết → hiểu → làm

1. **Mở đầu**: concept (từ vựng cần biết trước)
2. **Giữa**: decision + lesson (vì sao mọi thứ như hiện tại, bẫy đã gặp)
3. **Cuối**: pattern + skill/workflow (làm thế nào, theo bó nào)

### Bước 3: Soạn lộ trình

Mỗi mục gồm:
- Tóm tắt 2-3 câu
- Link `[[slug]]` đến bản đầy đủ
- 1 câu hỏi tự kiểm

KHÔNG copy nguyên văn page vào lộ trình. Wiki là bản gốc duy nhất, lộ trình chỉ là bản đồ đọc.

### Bước 4: Đầu ra

Trình cho user. Chỉ ghi ra file khi user chỉ định nơi lưu, lưu NGOÀI `.cockpit/`.

### Bước 5: Phát hiện lỗ hổng

Chủ đề quan trọng không có page, chuỗi học đứt quãng → báo user kèm danh sách thiếu, gợi ý capture/distill bổ sung. Emit `query-miss` nếu phải chắp >= 3 page.

### Usage log (mode teach)

```
---
timestamp: YYYY-MM-DDTHH:mm
type: knowledge-activity
source_skill: pk-consult
---
Teach: <chủ đề> | N page, thiếu: <danh sách ngắn hoặc "không">
```

## Auto-load concept pages từ tools.md

Khi task match keyword trong tools.md inventory → tự load concept page liên quan từ knowledge/. Read-only, không hỏi user.

## Quy tắc cứng

1. Query và Teach KHÔNG ghi vào knowledge. Đáng lưu → qua pk-capture → inbox. Teach chỉ ghi file khi user chỉ định, và NGOÀI `.cockpit/`.
2. Run KHÔNG sửa skill/workflow. Vướng → ghi usage log + gợi ý pk-distill refine.
3. Không bịa câu trả lời / bó không tồn tại.
4. Trích dẫn `[[slug]]` cho mọi tuyên bố từ page.
5. Run phát hiện làm theo wiki page → emit promote-candidate.
