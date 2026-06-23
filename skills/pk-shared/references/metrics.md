# Metrics: Công thức tính & tín hiệu chẩn đoán (canonical)

Logic ĐỌC dùng chung. `pk-analyze` đọc để render dashboard. `pk-track` đọc để compute trước khi ghi. Canonical duy nhất: skill khác link về đây, KHÔNG chép lại.

## Tiến độ Key Result (KR)

Công thức: `% = (Current - Baseline) / (Target - Baseline) * 100`

Cập nhật Current:
1. **User tự nhập** (ưu tiên, MỌI mode)
2. **Tính từ actions** (CHỈ pk-track mode deep): done/tổng thuộc KR

## KR Status (auto-compute)

Thứ tự ưu tiên (first match):

| # | Status | Điều kiện |
| --- | --- | --- |
| 1 | `achieved` | `current >= target` |
| 2 | `missed` | `current < target` AND `now > end_date` |
| 3 | `pending` | `current == baseline` AND `now <= end_date` |
| 4 | `in-progress` | Còn lại |

## Key Indicator Status (KI)

- `healthy`: current >= ngưỡng tối thiểu
- `warning`: 80% × ngưỡng <= current < ngưỡng
- `critical`: current < 80% × ngưỡng

## Trend (Project)

| Trend | Điều kiện |
| --- | --- |
| on-track | Tiến độ thực >= kỳ vọng theo timeline |
| at-risk | Chênh < 20% |
| off-track | Chênh >= 20% hoặc có actions blocked |

Tiến độ kỳ vọng = % thời gian đã trôi trong period.

## Trend (Ongoing)

So sánh KI status hiện tại vs lần review trước:
- `improving`: KI chuyển từ critical/warning → healthy
- `stable`: không đổi
- `declining`: KI chuyển từ healthy → warning/critical

## Period Overdue (Project only)

```
period_overdue_days = max(0, today - end_date)
overdue = (period_overdue_days > 0) AND (objective.status == "active")
```

Dashboard: render block cảnh báo ĐẦU (trước Key Results). Đề xuất extend hoặc đổi status. Không tự sửa.

Hết period: pk-reflect rút bài học, rồi pk-init update-objective (extend period hoặc đổi status). User quyết. Không có mode đóng period riêng.

## Nhắc review (canonical)

First match, 1 dòng tối đa:

**Project:**
| # | Điều kiện | Thông báo |
| --- | --- | --- |
| 1 | `last_track_date` is null | "Chưa track lần nào." |
| 2 | Qua nửa period, chưa review | "Đã qua nửa period, chưa review." |
| 3 | Chưa track > 14 ngày | "Chưa track 2 tuần." |

**Ongoing:**
| # | Điều kiện | Thông báo |
| --- | --- | --- |
| 1 | `last_track_date` is null | "Chưa track lần nào." |
| 2 | Chưa review, track > review_cycle × 2 | "Track nhiều lần nhưng chưa review." |
| 3 | Quá hạn review > review_cycle × 1.5 | "Quá hạn review N ngày." |
| 4 | Chưa track > 14 ngày | "Chưa track 2 tuần." |

## Action Health

- **Done rate**: `completed / total_actions` (từ counters `plan.md`)
- **Overdue**: `due_date < today` AND status ∈ {doing, blocked, pending}
- **Blocked**: status = blocked, liệt kê reason
- **Checkpoint slip**: action effort xl, mốc quá hạn chưa tick

## Knowledge Health (bổ sung cho dashboard hợp nhất)

- **Wiki count**: tổng page active trong knowledge/index.md
- **Skill/Workflow count**: tổng entry trong skills/registry.md + workflows/registry.md
- **Inbox knowledge pending**: đếm inbox domain=knowledge, status=pending
- **Page freshness**: % page active có `updated` < 90 ngày
- **Orphaned files**: file trong knowledge/ không có trong index.md

Ranh giới: số liệu mức dashboard thuộc pk-analyze. Soi chi tiết từng file thuộc pk-lint check.

### Usage count (canonical)

**Log là SOT của usage.** Cột `Usage` trong `knowledge/index.md` là cache-hint, CÓ THỂ CŨ. Khi cần giá trị chính xác (đặc biệt khi promote), bắt buộc recompute từ log, không tin cột index.

Cách đếm: đếm entry usage log của pk-consult trong `log/*.md` theo slug. Tính cả dạng `match: [[slug]]` lẫn `Run: [[slug]]`. pk-distill (cache vào index) và pk-lint (rebuild-index) dùng chung công thức này.

## Capacity / xung đột tài nguyên

Đọc constraints trong `objective.md` đối chiếu `actions/*.md`:

| Tín hiệu | Cảnh báo |
| --- | --- |
| Tổng giờ ước tính > capacity còn lại | Quá tải |
| >= 3 actions cùng deadline (±2 ngày) | Dồn việc |
| Action cần skill chưa có | Gap, đề xuất bổ sung |
