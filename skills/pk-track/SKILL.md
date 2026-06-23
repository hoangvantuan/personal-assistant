---
name: pk-track
description: "Xử lý inbox execution và cập nhật KR current (tiến độ thực tế). Light: ghi nhận tiến độ KR/action hàng ngày. Deep: review sâu với root cause analysis. Dùng khi user nói 'cập nhật tiến độ', 'xong task X', 'review sâu', 'tại sao chậm', hoặc cần xử lý inbox execution. KHÔNG dùng khi xử lý inbox knowledge (dùng pk-distill) hoặc sửa KR target/mục tiêu/cấu trúc kế hoạch (dùng pk-init hoặc pk-plan)."
---

# PK Track: Track + Review + Inbox Execution

## Ensure snapshot (bắt buộc, đầu flow)

1. **Precondition**: `.cockpit/` tồn tại. Thiếu → route `pk-init` (mode new), KHÔNG nạp gì thêm, dừng.
2. **Idempotent qua marker**: phiên CHƯA có marker `SNAPSHOT_LOADED` → tự nạp full theo Snapshot Contract
   (`../pk-shared/references/snapshot-contract.md`) rồi đặt marker `SNAPSHOT_LOADED`. Đã có marker (vd
   harness Phase 1 đã đặt) → skip, không đọc lại.

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
| plan.md | counters, last_track_date, last_review_date, re-render Roadmap (view dẫn xuất, sau khi ghi actions/) |
| actions/*.md | status, completed_date, `## Output/Deliverable` |
| log/*.md | Append entries |
| inbox/*.md (domain=execution) | Status transition |
| archive/actions/ | Move done actions |
| archive/inbox/ | Move items processed/discarded |

> Subset của `../pk-shared/references/sot-ownership.md`.

**KHÔNG được sửa**: Objective text, KR target, action title/due_date/deps. Đề xuất → delegate pk-init/pk-plan (pre-confirmed).

## Nguyên tắc

- Dashboard TRƯỚC khi hỏi update.
- Confirm trước ghi. <= 2 field → 1 dòng, >= 3 → bảng.
- Metrics: `../pk-shared/references/metrics.md` (canonical).
- Append-only log.
- Cuối flow đề xuất next action.

## Flow: mode light

### Phase 1: Snapshot + Dashboard

Dùng pk-analyze (focus: track) → thu analysis.

### Phase 2: Tương tác update

Hỏi user: KR.current? Action status?

### Phase 3: Confirm

Bảng thay đổi. Trước khi trình bảng confirm, áp Quality Gate (`../pk-shared/references/quality-gate.md`) cho input số liệu KR/KI current từ user (3 câu nội bộ: đủ cụ thể, giả định ẩn, mâu thuẫn). User confirm.

### Phase 4: Ghi

- **Đọc body action hiện tại trước khi ghi** `## Output/Deliverable` (bất biến canonical: `../pk-shared/references/snapshot-contract.md`, mục "Bất biến đọc body trước khi ghi body"). Snapshot chỉ có frontmatter; không đọc body trước = rủi ro ghi đè.
- Cập nhật KR/KI current trong objective.md
- Cập nhật action status trong actions/*.md
- Tính KR Status auto-compute (metrics.md)
- Archive done actions (schemas.md archive rules)
- Cập nhật plan.md counters + last_track_date
- Re-render Roadmap

### Phase 5: Xử lý inbox execution (nếu có)

Đọc inbox `domain=execution`, `status=pending`. Trình đề xuất, user duyệt rồi mới thực hiện. Mỗi item:

| Inbox type | Xử lý |
| --- | --- |
| `action` | Delegate pk-plan **pre_confirmed** (user đã duyệt trong flow track; theo Delegate Protocol tại `../pk-shared/references/cross-call-rules.md`) |
| `blocker` | Tự ghi (sửa action.status = blocked) |
| `resource` | Hỏi user: link vào action notes hoặc đổi domain=knowledge |
| `thought` | Hỏi user: chuyển thành action, đổi domain, hoặc discard |

**resource đổi domain=knowledge**: chỉ set field `domain: knowledge` ngay trong file, giữ nguyên file trong `inbox/`. **KHÔNG set `status: processed`, KHÔNG move sang archive.** pk-distill sẽ tự nhặt item này khi quét `inbox/ domain=knowledge, status=pending`.

**Item có `related_inbox`** (cặp domain=both): khi xử lý item execution, đọc `related_inbox` để biết item cặp (domain=knowledge). Thông báo cho user biết item cặp cần được pk-distill xử lý nốt, tránh bỏ rơi nửa cặp.

Set `status: processed` hoặc `discarded`. Move item sang `archive/inbox/` (TRỪ item resource vừa đổi sang domain=knowledge: không move, không set processed).

### Phase 6: Log + Tóm tắt

Append log. Đề xuất next action. Gợi ý rút bài học (pk-reflect) nếu flow có thực chất VÀ chạy standalone. Khi chạy qua pk-harness thì bỏ qua, harness Phase 3 đảm nhiệm.

## Flow: mode deep

### Phase 1: Deep analysis

pk-analyze deep: đọc toàn bộ + log + root cause (>= 3 "tại sao?").

### Phase 2: Trình bày + đề xuất

Root cause mỗi issue + đề xuất điều chỉnh (cấu trúc + progress).

### Phase 3: All-changes confirm

Gom proposals → trình user duyệt batch.

### Phase 4: Ghi progress

Cập nhật KR/action (giống light Phase 4). Ngoài ra, deep ghi thêm `last_review_date` vào plan.md (light chỉ ghi `last_track_date`, không ghi `last_review_date`).

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
5. Xử lý (blocker tự ghi; action và thought đã duyệt thành action: delegate pk-plan pre_confirmed; resource: hỏi user)
6. Move item đã xử lý sang `archive/inbox/`
7. Báo cáo

## Tích hợp: Track → Consult

Khi phát hiện action có skill/workflow liên quan (từ notes hoặc keyword match):
- Gọi pk-consult mode run (cross-call lớp 2 → lớp 3)
- Kết quả quay về cập nhật action status/output

## Quy tắc cứng

1. KHÔNG sửa cấu trúc action (title, due_date, deps). Delegate.
2. KHÔNG ghi vào knowledge/skills/workflows. Gọi pk-capture nếu phát hiện tri thức.
3. Confirm trước mọi ghi.
4. Archive rules theo `../pk-shared/references/schemas.md`.
5. **Verify sequence trước khi gán số thứ tự.** Khi gặp counter/sequence nghiệp vụ mà pk-track ghi qua KR/KI current (buổi coaching, sprint, version), kiểm tra gap giữa records hiện có. Nếu có khoảng trống (ví dụ: B3 → B5 thiếu B4), hỏi user xác nhận trước khi gán số tiếp. (Verify sequence cho action ID là việc của pk-plan, nơi sinh AXXX.)
