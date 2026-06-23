---
name: pk-reflect
description: "Rút bài học từ phiên làm việc hoặc sự kiện. Light (mặc định): quét nhanh phiên, trích bài học. Deep: phỏng vấn AAR 5 câu chuyên sâu. Dùng khi user nói 'rút bài học', 'retro', 'tổng kết phiên', 'rút kinh nghiệm', 'AAR', hoặc vừa kết thúc một giai đoạn và muốn nhìn lại. Output đi vào inbox qua pk-capture."
---

# PK Reflect: Rút bài học

## Ensure snapshot (bắt buộc, đầu flow)

1. **Precondition**: `.cockpit/` tồn tại. Thiếu → route `pk-init` (mode new), KHÔNG nạp gì thêm, dừng.
2. **Idempotent qua marker**: phiên CHƯA có marker `SNAPSHOT_LOADED` → tự nạp full theo Snapshot Contract
   (`../pk-shared/references/snapshot-contract.md`) rồi đặt marker `SNAPSHOT_LOADED`. Đã có marker (vd
   harness Phase 1 đã đặt) → skip, không đọc lại.

2 mode: **light** (mặc định, rút nhanh từ phiên) và **deep** (phỏng vấn AAR 5 câu). Output → inbox.

## Modes

| Mode | Trigger | Mô tả |
| --- | --- | --- |
| `light` | Mặc định, "rút bài học", "retro" | Quét phiên, trích candidates, bàn giao pk-capture |
| `deep` | "AAR", "phỏng vấn sâu", user chỉ định rõ | Phỏng vấn 5 câu + gọi pk-analyze lấy data |

## Nguyên tắc

- Reflect KHÔNG ghi file trực tiếp vào knowledge/skills/workflows. Mọi output đi qua pk-capture → inbox.
- Light rút nhanh. Deep phải chỉ định rõ.
- Bàn giao cho pk-capture (cross-call lớp 2 → lớp 3) để giữ bất biến "một cửa inbox".

## Flow: mode light (session extraction)

### Bước 1: Quét phiên

Nhìn lại TOÀN BỘ hội thoại phiên. Tìm:
- Chỗ user sửa/bác bỏ cách làm
- Chỗ vướng, làm lại, hiểu nhầm
- Quy ước/ràng buộc mới lộ ra
- Pattern được phát hiện

### Bước 2: Lọc chất lượng

Chỉ giữ bài học **tái dùng được + không hiển nhiên + actionable**.

### Bước 3: Bàn giao pk-capture

Gọi pk-capture flow từ Bước 2 (trình candidates). Set `captured_from: reflect-light`. Capture lo: duyệt, ghi inbox, ghi log.

## Flow: mode deep (AAR phỏng vấn)

### Bước 1: Lấy data

Gọi pk-analyze (cross-call lớp 2 → lớp 4) lấy metrics, trends. Đọc log gần nhất.

**Degraded** (khi `.cockpit/` có nhưng KHÔNG có objective hoặc plan, tức "pk-reflect deep degraded" theo `../pk-shared/references/snapshot-contract.md`, mục "Graceful Degradation"): phỏng vấn 5 câu VẪN chạy, nhưng bỏ phần metrics/trends (pk-analyze chỉ knowledge-only, data nghèo). Thông báo cho user biết mode degraded trước khi tiếp tục.

### Bước 2: Chọn bối cảnh

| Bối cảnh | Tín hiệu | Trọng tâm |
| --- | --- | --- |
| Sau sự cố | Vừa xử lý xong lỗi | Nguyên nhân gốc + phòng ngừa |
| Sau milestone / cuối dự án | Bàn giao xong | Kỳ vọng vs thực tế |
| Định kỳ (sprint) | User muốn retro | Việc lặp + lỗi lặp |

### Bước 3: Phỏng vấn 5 câu

Hỏi TỪNG CÂU MỘT, chờ trả lời:

1. **Kỳ vọng**: Ban đầu định đạt gì?
2. **Thực tế**: Kết quả thật ra sao? Lệch chỗ nào?
3. **Nguyên nhân gốc**: Tại sao lệch? ("tại sao?" >= 3 lớp)
4. **Bài học**: 1 câu cho người ngoài hiểu. Khi nào áp dụng, khi nào KHÔNG?
5. **Hành động hệ thống**: Cái gì trong hệ thống phải đổi?

### Bước 4: Tổng hợp candidates

Từ phỏng vấn, tách thành candidates:
- 1 item `lesson` (lõi phiên)
- 0-n item `decision` / `pattern` / `troubleshooting` / `concept`
- 0-n item `candidate-skill` / `candidate-workflow` (từ câu 5)

### Bước 5: Bàn giao pk-capture

Set `captured_from: reflect-deep`. Nguồn raw: nguyên văn hỏi-đáp phỏng vấn.

## Quy tắc phỏng vấn (mode deep)

- Gợi nhớ bằng dữ kiện, không hỏi chay.
- Không phán xét. "Do tôi quên" → "vì sao quên được, hệ thống thiếu chốt chặn nào?"
- Một lần phản tư, một chủ đề.
- Ngắn: 10-15 phút. Đủ 5 câu dừng.
- User trả lời nông → chấp nhận. Bài học nông hơn không gì.

## Quy tắc cứng

1. Reflect KHÔNG ghi vào knowledge, skills, workflows, inbox. Mọi kết quả qua pk-capture.
2. Hỏi từng câu. CẤM dán 5 câu thành bảng hỏi.
3. Câu 5 (hành động hệ thống) bắt buộc. Bài học không kèm thay đổi hệ thống sẽ bị quên.
4. Trích nguyên văn lời user trong raw. Không diễn đạt lại.
