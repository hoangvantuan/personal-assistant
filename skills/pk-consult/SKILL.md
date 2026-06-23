---
name: pk-consult
description: "Tra cứu và vận dụng tri thức đã đúc kết trong .cockpit/. 3 mode: tra cứu (hỏi kho có gì về X), thực thi (chạy skill hoặc workflow đã lưu), soạn lộ trình học (onboarding, dạy lại X cho người mới). Dùng khi user muốn tìm kiến thức cũ, áp dụng quy trình đã đúc kết, hoặc soạn tài liệu hướng dẫn từ knowledge base."
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
| Mô tả task + user muốn THỰC THI việc | run |
| Mô tả task + user chỉ muốn BIẾT CÁCH làm | query |

## Mode Query

### Bước 1: Tìm page liên quan

1. Đọc registries (đã có trong snapshot)
2. Rút 3-5 từ khoá. Grep:
   ```bash
   grep -ril "<từ khoá>" .cockpit/knowledge .cockpit/skills .cockpit/workflows
   ```
3. Đọc page hit

### Bước 2: Tổng hợp câu trả lời

- Gặp page có `status: stub` hoặc `redirect_to` trong frontmatter (đọc được từ cột Redirect trong index, snapshot đã preload) → đọc bản đích tại `redirect_to`. Đích có thể là skill hoặc workflow. Ưu tiên đọc frontmatter/index, không cần mở body page nguồn.
- Trả lời trực tiếp từ page
- Trích dẫn `[[slug]]`
- Không có → "chưa có knowhow về việc này"

### Bước 3: Phát tín hiệu

**query-miss** vào `schema-signals.md` khi đạt emit-threshold của `query-miss` (canonical: `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act"): phải chắp nhiều page mới trả lời được, HOẶC không page nào trả lời trực tiếp.

**promote-candidate** khi:
- Câu hỏi thao tác ("làm sao...", "các bước...") VÀ
- Trả lời dựa chủ yếu vào 1 page thuộc type đủ điều kiện promote (canonical: `../pk-shared/references/schemas.md`, mục "Promote criteria")

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
3. Đúng 1 khớp → Nhịp 2. Nhiều → hỏi user chọn. Không có → wiki-fallback (bước 4)
4. Wiki-fallback: grep `knowledge/` tìm page thuộc type đủ điều kiện promote (canonical: `../pk-shared/references/schemas.md`, mục "Promote criteria")
   - Có page → hỏi user có làm theo wiki không. Đồng ý → thực hiện như Nhịp 2 + emit promote-candidate vào `schema-signals.md`
   - Không có page nào → báo "chưa có skill/workflow"

### Nhịp 2: Load

Mở file skill/workflow. Đọc HẾT nội dung.

Gặp page có `status: stub` hoặc `redirect_to` trong frontmatter (phát hiện từ cột Redirect trong index) → đọc và dùng bản đích (`redirect_to`). Đích có thể là skill hoặc workflow.

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

Chủ đề quan trọng không có page, chuỗi học đứt quãng → báo user kèm danh sách thiếu, gợi ý capture/distill bổ sung. Emit `query-miss` khi đạt emit-threshold của `query-miss` (canonical: `../pk-shared/references/schemas.md`, mục "Bảng ngưỡng emit/act").

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
5. Run phát hiện làm theo wiki page → emit promote-candidate vào `schema-signals.md` (chỉ khi page thuộc type đủ điều kiện promote, canonical: `../pk-shared/references/schemas.md` mục "Promote criteria").
