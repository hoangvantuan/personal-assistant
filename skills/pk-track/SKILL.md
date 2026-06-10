---
name: pk-track
description: "Cập nhật progress, review sâu, inbox execution. 3 mode: light (tiến độ), deep (review + root cause), inbox-only (xử lý inbox execution). Trigger: track, cập nhật, review sâu, inbox."
---

# PK Track: Track + Review + Inbox Execution

Ghi progress fields, log, archive. KHÔNG sửa cấu trúc (delegate sang pk-init/pk-plan).

## Modes

| Mode | Trigger | Mô tả |
| --- | --- | --- |
| `light` | Daily tracking | Cập nhật KR/action progress |
| `deep` | Review sâu, at-risk | Root cause + đề xuất điều chỉnh |
| `inbox-only` | Xử lý inbox execution | Chỉ inbox domain=execution |

## SOT quyền ghi

| File | Fields |
| --- | --- |
| objective.md | KR.current, KI.current |
| plan.md | counters, last_track_date, last_review_date |
| actions/*.md | status, `## Output/Deliverable` |
| log/*.md | Append entries |
| inbox/*.md (domain=execution) | Status transition |
| archive/actions/ | Move done actions |

> Subset của `../pk-shared/references/sot-ownership.md`.

**KHÔNG được sửa**: Objective text, KR target, action title/deadline/deps. Đề xuất → delegate pk-init/pk-plan (pre-confirmed).

## Nguyên tắc

- Dashboard TRƯỚC khi hỏi update.
- Confirm trước ghi. <= 2 field → 1 dòng, >= 3 → bảng.
- Snapshot Contract: idempotent.
- Metrics: `../pk-shared/references/metrics.md` (canonical).
- Append-only log.
- Cuối flow đề xuất next action.

## Flow: mode light

### Phase 1: Snapshot + Dashboard

Dùng pk-analyze (focus: progress, overdue, blocked) → thu analysis.

### Phase 2: Tương tác update

Hỏi user: KR.current? Action status?

### Phase 3: Confirm

Bảng thay đổi. User confirm.

### Phase 4: Ghi

- Cập nhật KR/KI current trong objective.md
- Cập nhật action status trong actions/*.md
- Tính KR Status auto-compute (metrics.md)
- Archive done actions (schemas.md archive rules)
- Cập nhật plan.md counters + last_track_date
- Re-render Roadmap

### Phase 5: Xử lý inbox execution (nếu có)

Đọc inbox `domain=execution`, `status=pending`. Mỗi item:

| Inbox type | Xử lý |
| --- | --- |
| `action` | Delegate pk-plan (tạo action) |
| `blocker` | Tự ghi (sửa action.status = blocked) |

Set `status: processed` hoặc `discarded`.

### Phase 6: Log + Tóm tắt

Append log. Đề xuất next action. Gợi ý rút bài học nếu flow có thực chất.

## Flow: mode deep

### Phase 1: Deep analysis

pk-analyze deep: đọc toàn bộ + log + root cause (>= 3 "tại sao?").

### Phase 2: Trình bày + đề xuất

Root cause mỗi issue + đề xuất điều chỉnh (cấu trúc + progress).

### Phase 3: All-changes confirm

Gom proposals → trình user duyệt batch.

### Phase 4: Ghi progress

Cập nhật KR/action (giống light Phase 4).

### Phase 5: Delegate thay đổi cấu trúc

Với mỗi thay đổi cấu trúc đã confirm, gọi pk-init/pk-plan với `pre_confirmed: true` (`../pk-shared/references/cross-call-rules.md` delegate protocol).

### Phase 6: Phát hiện tri thức mới

Deep review phát hiện pattern/lesson → gọi pk-capture (cross-call lớp 2 → lớp 3) tạo inbox domain=knowledge. pk-track KHÔNG tự ghi vào knowledge (giữ SOT).

### Phase 7: Log review

Ghi log type=review.

## Flow: mode inbox-only

1. Đọc inbox `domain=execution`, `status=pending`
2. Cảnh báo items > 30 ngày
3. Gợi ý xử lý per item
4. User quyết định
5. Xử lý (blocker tự ghi, action/resource delegate)
6. Báo cáo

## Tích hợp: Track → Consult

Khi phát hiện action có skill/workflow liên quan (từ notes hoặc keyword match):
- Gọi pk-consult mode run (cross-call lớp 2 → lớp 3)
- Kết quả quay về cập nhật action status/output

## Quy tắc cứng

1. KHÔNG sửa cấu trúc action (title, deadline, deps). Delegate.
2. KHÔNG ghi vào knowledge/skills/workflows. Gọi pk-capture nếu phát hiện tri thức.
3. Confirm trước mọi ghi.
4. Archive rules theo `../pk-shared/references/schemas.md`.
5. **Verify sequence trước khi gán số thứ tự.** Khi gặp counter/sequence (buổi coaching, sprint, version, action ID), kiểm tra gap giữa records hiện có. Nếu có khoảng trống (ví dụ: B3 → B5 thiếu B4), hỏi user xác nhận trước khi gán số tiếp.
