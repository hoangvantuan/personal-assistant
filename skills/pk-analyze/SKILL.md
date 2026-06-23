---
name: pk-analyze
description: "Xem tình hình dự án, đo tiến độ, phân tích vấn đề. Read-only, không sửa file. Dùng khi user hỏi 'tình hình thế nào?', muốn xem dashboard, tổng hợp metrics, hoặc chẩn đoán 'có vấn đề gì không?'. Mọi câu hỏi mang tính quan sát, đo lường, chẩn đoán về dự án đều dùng skill này."
---

# PK Analyze: Phân tích trạng thái (read-only)

## Ensure snapshot (bắt buộc, đầu flow)

1. **Precondition**: `.cockpit/` tồn tại. Thiếu → route `pk-init` (mode new), KHÔNG nạp gì thêm, dừng.
2. **Idempotent qua marker**: phiên CHƯA có marker `SNAPSHOT_LOADED` → tự nạp full theo Snapshot Contract
   (`../pk-shared/references/snapshot-contract.md`) rồi đặt marker `SNAPSHOT_LOADED`. Đã có marker (vd
   harness Phase 1 đã đặt) → skip, không đọc lại.

Đọc `.cockpit/`, tính metrics, phát hiện vấn đề, xếp ưu tiên. **KHÔNG BAO GIỜ ghi file.**

## Nguyên tắc

1. **Read-only**: không tạo, sửa, xoá file nào.
2. **Song song**: đọc nhiều file cùng lúc.
3. **Chính xác**: dùng công thức chuẩn (`../pk-shared/references/metrics.md`). Số liệu trích từ file.
4. **Root cause** (mode deep): "tại sao?" >= 3 lần trước khi kết luận.
5. **Cụ thể**: mọi nhận định kèm evidence.

## Đọc state

Snapshot Contract (`../pk-shared/references/snapshot-contract.md`, mục "Context Snapshot") đã preload đủ phần chỉ mục/định danh. KHÔNG đọc lại các nguồn này.

Đọc thêm (DELTA) ngoài snapshot, song song:

| File | Đọc gì | Dùng để |
| --- | --- | --- |
| `objective.md` body | Bảng KR/KI targets + current | Tính metrics |
| `plan.md` body | Roadmap section | Đọc milestone/action structure (focus track, full) |
| `log/` | Entries gần nhất | Chỉ mode deep: tốc độ hoàn thành, trends |

## Tính metrics

Mọi công thức ở `../pk-shared/references/metrics.md`. KHÔNG chép lại.

## Phát hiện issues

Quét theo nghiêm trọng:

1. Period overdue
2. Action overdue (ghi số ngày)
3. Action blocked (liệt kê blocker)
4. KR at-risk/off-track
5. Checkpoint slip (effort xl)
6. Inbox aging (> 30 ngày, phân execution vs knowledge)
7. Capacity xung đột
8. Knowledge health (mức số liệu, công thức tại `../pk-shared/references/metrics.md`): orphan count, index lệch, freshness thấp. Bất thường thì gợi ý chạy `pk-lint check`.

## Xếp priority (top N actions)

First-match: overdue → block action khác → deadline trong horizon → priority cao doing → KR at-risk.

## Dashboard hợp nhất

Dashboard hiển thị CẢ HAI module:

### Layout: Execution

```
Dashboard: [Tên Objective]
[Câu tổng quan]
[Nhắc review nếu có]
Period: [start] > [end] ([X]% thời gian)

Key Results
  KR1: ████░░░░░░ 40/100 (40%) > on-track
    Active: 2 doing | 1 blocked | 0 pending

Actions: N tổng | X done | Y doing | Z blocked
Inbox execution: [N] pending
```

### Layout: Knowledge

```
Knowledge
  Wiki: [N] pages ([M]% fresh)
  Skills: [N] | Workflows: [N]
  Inbox knowledge: [N] pending

Cần chú ý
  - [issues]
```

### Knowledge-only mode (không có objective)

Chỉ hiện Knowledge section. Skip execution hoàn toàn.

## Mode deep

Bổ sung:
- Đọc log history
- Root cause mỗi issue (>= 3 "tại sao?")
- Tốc độ hoàn thành (done/tuần)
- Output thêm "Root cause" section

## Tham số

| Param | Mặc định | Mô tả |
| --- | --- | --- |
| mode | light | `light`: dashboard tiêu chuẩn (không đọc log, không root cause). `mode=deep` bật phần "Mode deep" (đọc log, root cause >= 3 "tại sao?", tốc độ hoàn thành). |
| focus | full | "full", "progress", "issues", "priority", "knowledge", "track" |
| max_items | 2 (dashboard), 3 (track) | Số priority items |
| horizon_days | 3 (dashboard), 7 (track) | Cửa sổ deadline |

Ghi chú: `focus=track` (tracking nhẹ) chỉ tính progress + quét issues loại 1-3. Ngữ cảnh track dùng `max_items=3`, `horizon_days=7`. `mode` và `focus` là hai chiều độc lập: `mode` kiểm soát độ sâu phân tích, `focus` kiểm soát phạm vi hiển thị.
