# Preload Contract: Context Snapshot duy nhất

Canonical. Định nghĩa context snapshot MỌI skill pk-* đều đọc ở đầu flow. Maintain 1 chỗ duy nhất.

**Vấn đề giải quyết**: khi gọi skill lẻ (`/pk-track`, `/pk-plan`...) KHÔNG qua `pk-harness`, skill thiếu snapshot mà orchestrator đáng lẽ làm. Contract bảo đảm: mọi entry point đều nạp đủ nền.

## Nguyên tắc idempotent (không đọc trùng)

Chỉ nạp cái **chưa có** trong context. Idempotency **kiểm được bằng marker `SNAPSHOT_LOADED`** (không dựa vào "agent tự nhớ"):

- Chạy QUA `pk-harness`: Phase 1 snapshot xong, harness **đặt marker `SNAPSHOT_LOADED`**. Skill thấy marker → KHÔNG đọc lại.
- Chạy LẺ (không qua harness): skill tự nạp full một lần ở đầu flow rồi **đặt marker `SNAPSHOT_LOADED`**.

Trước khi nạp: kiểm tra `.cockpit/` tồn tại. Không có → route `pk-init` (không nạp gì thêm).

Các skill trỏ về đây để triển khai block "Ensure snapshot" chuẩn.

## Context Snapshot (1 bản duy nhất)

**Bước 0 (bắt buộc):** `ls -1 .cockpit/` (cấp 1, KHÔNG đệ quy). Phát hiện cấu trúc + anomalies.

| Nguồn | Độ sâu nạp | Dùng để |
| --- | --- | --- |
| `objective.md` | Frontmatter: `type`, `status`, `period`, `review_cycle`, KR/KI IDs, constraints tóm tắt | Route, chọn metrics, guard paused |
| `tools.md` | **TOÀN BỘ body** | Inventory công cụ, auto-load concept pages khi match |
| `plan.md` | Frontmatter + counters | Có/không plan, done rate |
| `actions/*.md` | Scan frontmatter | Count by status, active IDs |
| `inbox/*.md` | Scan frontmatter `status: pending` + `captured_at` mỗi item (cho Inbox Aging) | Count by domain + dữ liệu aging |
| `schema-signals.md` | **TOÀN BỘ** (tối thiểu 2 section "Đang chờ xử lý" / "Đã xử lý") | Tín hiệu promote-candidate, strain; dùng bởi pk-distill, pk-lint |
| `SCHEMA.md` | **TOÀN BỘ body** | Quy ước dữ liệu project-specific; dùng bởi pk-distill (phân loại type) và pk-lint (evolve) |
| `knowledge/index.md` | **TOÀN BỘ** | Registry tri thức + usage count |
| `skills/registry.md` | **TOÀN BỘ** | Registry skills |
| `workflows/registry.md` | **TOÀN BỘ** | Registry workflows |
| Pinned pages | **Full body** | Lấy theo cột Pinned trong `knowledge/index.md`, KHÔNG scan frontmatter từng file |

## KHÔNG preload (on-demand)

| Nguồn | Ai đọc, khi nào |
| --- | --- |
| `objective.md` body (bảng KR/KI) | `pk-analyze` khi tính metrics |
| `plan.md` body (Roadmap) | `pk-track` re-render, `pk-plan` update |
| `actions/*.md` body | `pk-track` khi update |
| `knowledge/*.md` body | `pk-consult` khi query, `pk-distill` khi xử lý |
| `skills/*.md` body | `pk-consult` mode run |
| `workflows/*.md` body | `pk-consult` mode run |
| `log/**` | `pk-track` deep, `pk-analyze` deep, `pk-distill` (cache usage_count) |
| `archive/**` | `pk-lint` restore, `pk-lint` check (reachability) |
| `raw/**` | `pk-capture` provenance |

## Áp dụng tri thức (không chỉ load cho có)

`knowledge/index.md` đã nạp → dùng làm **context định hướng**. Trước khi đề xuất/ghi: đối chiếu knowledge page có nội dung liên quan việc đang làm. Cần detail → đọc body page tương ứng.

**Trước khi hỏi user bất kỳ thông tin gì**, kiểm tra snapshot đã nạp (tools.md, knowledge/, objective.md). Dữ liệu có trong cockpit thì dùng, không hỏi.

## Auto-load concept pages từ tools.md

Khi skill phát hiện keyword match giữa task hiện tại và tool/concept trong snapshot → tự load concept page chi tiết từ `knowledge/`. An toàn vì read-only.

## Bất biến "đọc body trước khi ghi body"

Snapshot chỉ preload chỉ mục và frontmatter, **KHÔNG preload body**. Do đó:

**Mỗi điểm GHI body file phải ĐỌC body hiện tại của file đó TRƯỚC khi ghi.** Ghi đè body mà không đọc trước = mất dữ liệu.

Áp dụng cho mọi skill ghi body (pk-track, pk-plan, pk-init, pk-distill, pk-lint). Không ngoại lệ.

## Reachability khi ghi (chống file mồ côi)

Mọi file trong `.cockpit/` phải reachable từ SOT:

| Loại file | Neo vào |
| --- | --- |
| Action | `actions/` + Roadmap link trong `plan.md` |
| Knowledge page | `knowledge/index.md` entry |
| Skill | `skills/registry.md` entry |
| Workflow | `workflows/registry.md` entry |
| Inbox item | `inbox/` (thư mục cấu trúc) |
| Log entry | `log/` (thư mục cấu trúc) |
| Raw source | `raw/` (thư mục cấu trúc) |

## Graceful Degradation

| Tình trạng | Hành vi |
| --- | --- |
| Không có `.cockpit/` | pk-harness route sang pk-init |
| Có `.cockpit/`, không có objective.md | Danh sách canonical các skill vẫn chạy được: pk-capture, pk-consult, pk-distill, pk-lint, pk-reflect (light). pk-analyze chạy knowledge-only mode. pk-reflect deep degraded. pk-plan và pk-track báo "chưa có mục tiêu." |
| Có objective.md, chưa có plan.md | pk-track báo "chưa có plan." pk-analyze chỉ hiện KR status. |
| Có cả hai | Đầy đủ chức năng |

pk-capture khi không có objective: `related_kr` và `related_action` để null, domain mặc định = knowledge.
