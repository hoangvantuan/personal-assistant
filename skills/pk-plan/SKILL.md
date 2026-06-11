---
name: pk-plan
description: "Tạo hoặc sửa kế hoạch hành động: plan, milestones, actions, dependencies, deadline. Dùng khi user cần lập kế hoạch, phân rã mục tiêu thành bước cụ thể, thêm/sửa/xoá action, điều chỉnh milestone. Thay đổi CẤU TRÚC kế hoạch dùng skill này. Thay đổi TIẾN ĐỘ dùng pk-track."
---

# PK Plan: Tạo + cập nhật plan & actions

SOT chính: `plan.md` và `actions/`.

## Modes

| Mode | Trigger | Mô tả |
| --- | --- | --- |
| `new` | Chưa có plan.md | Tạo plan + milestones + actions |
| `update` | Sửa plan/action/milestone | Cập nhật cấu trúc |
| `pre-confirmed` | Áp dụng thay đổi đã duyệt từ pk-track | Delegate protocol |

## SOT quyền ghi

| File | Fields |
| --- | --- |
| plan.md | Milestones, counters (khi tạo), re-render Roadmap (view dẫn xuất) |
| actions/*.md | title, due_date, deps, effort, priority, DoD, notes |

> Subset của `../pk-shared/references/sot-ownership.md`.

## Điều kiện tiên quyết

- `objective.md` tồn tại và có nội dung. Thiếu → "Cần init objective trước."
- `tools.md` nên tồn tại (cross-check tools khả dụng).

## Nguyên tắc

- Mỗi action BẮT BUỘC: DoD rõ ràng, Output/Deliverable, tiêu chí chất lượng.
- Action mơ hồ ("Nghiên cứu thêm" không output) → CẤM.
- Effort xl → BẮT BUỘC Checkpoints hoặc tách.
- Confirm bảng trước ghi. Render Roadmap sau ghi.
- Snapshot Contract (`../pk-shared/references/snapshot-contract.md`): idempotent.
- Quality Gate (`../pk-shared/references/quality-gate.md`): 3 câu trước mỗi follow-up.

## Tích hợp tri thức (Plan → Consult)

Khi tạo action, gọi pk-consult (cross-call lớp 2 → lớp 3) tìm pattern, decision, troubleshooting liên quan. Nếu có skill/workflow phù hợp, link vào action notes.

Thực hiện:
1. Rút 3-5 keyword từ action title/description
2. Đối chiếu knowledge/index.md + skills/registry.md + workflows/registry.md (đã có trong snapshot)
3. Match → đọc page detail, trích phần liên quan vào notes
4. Có skill/workflow khớp → ghi `related_skill: [[slug]]` hoặc `related_workflow: [[slug]]` trong action notes

## Flow: mode new

1. Đọc objective.md (KR/KI targets)
2. Đọc tools.md (tools khả dụng)
3. Hỏi milestones + timeline
4. Mỗi milestone: hỏi actions, Quality Gate mỗi follow-up
5. Tích hợp tri thức: pk-consult tìm knowledge liên quan
6. Confirm bảng tổng
7. Ghi plan.md + actions/*.md
8. Render Roadmap

## Flow: mode update

1. Đọc plan.md + actions hiện tại
2. Hỏi user thay đổi gì
3. Quality Gate
4. Tích hợp tri thức cho action mới
5. Confirm diff
6. Ghi + re-render Roadmap

## Flow: mode pre-confirmed

Nhận payload từ pk-track (mọi mode, user đã duyệt trong flow track). Theo delegate protocol tại `../pk-shared/references/cross-call-rules.md`:
1. Hiển thị block lý do + diff
2. Ghi ngay (skip confirm)
3. Kèm `(Đã được confirm tại track. Ghi ngay.)`

## Quy tắc

- KHÔNG ghi file trước confirm (trừ pre-confirmed).
- Không sửa KR/KI targets (việc pk-init).
- Không sửa action status (việc pk-track).
- Không tạo/sửa knowledge (việc pk-distill).
