---
name: pk-analyze
description: "Xem tình hình dự án, đo tiến độ, phân tích vấn đề. Read-only, không sửa file. Dùng khi user hỏi 'tình hình thế nào?', muốn xem dashboard, tổng hợp metrics, hoặc chẩn đoán 'có vấn đề gì không?'. Mọi câu hỏi mang tính quan sát, đo lường, chẩn đoán về dự án đều dùng skill này."
---

# PK Analyze: Phân tích trạng thái (read-only)

Đọc `.cockpit/`, tính metrics, phát hiện vấn đề, xếp ưu tiên. **KHÔNG BAO GIỜ ghi file.**

## Nguyên tắc

1. **Read-only**: không tạo, sửa, xoá file nào.
2. **Song song**: đọc nhiều file cùng lúc.
3. **Chính xác**: dùng công thức chuẩn (`../pk-shared/references/metrics.md`). Số liệu trích từ file.
4. **Root cause** (mode deep): "tại sao?" >= 3 lần trước khi kết luận.
5. **Cụ thể**: mọi nhận định kèm evidence.

## Đọc state

Snapshot Contract (`../pk-shared/references/snapshot-contract.md`): idempotent. Chạy qua harness đã có, chạy lẻ tự nạp.

Đọc song song:

| File | Đọc gì |
| --- | --- |
| `objective.md` | Frontmatter + KR/KI bảng |
| `tools.md` | TOÀN BỘ (đã có trong snapshot) |
| `plan.md` | Frontmatter + Roadmap body |
| `actions/*.md` | Frontmatter only |
| `inbox/` | Count + frontmatter pending |
| `knowledge/index.md` | TOÀN BỘ (đã có trong snapshot) |
| `skills/registry.md` | TOÀN BỘ (đã có trong snapshot) |
| `workflows/registry.md` | TOÀN BỘ (đã có trong snapshot) |

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
8. Knowledge health: page orphaned, index lệch, freshness thấp
9. Reachability audit

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
| focus | full | "full", "progress", "issues", "priority", "knowledge" |
| max_items | 2 (dashboard), 3 (track) | Số priority items |
| horizon_days | 3 (dashboard), 7 (track) | Cửa sổ deadline |
