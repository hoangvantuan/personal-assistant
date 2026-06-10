<div align="center">

# Project Cockpit

**Workspace per-project hợp nhất quản lý mục tiêu với quản lý tri thức.**

Vận hành bằng [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills.

</div>

## Tổng quan

Project Cockpit gộp hai hệ thống thành một workspace duy nhất cho mỗi dự án (`.cockpit/`):

- **Module Thực thi**: mục tiêu, key results, kế hoạch, hành động, tracking tiến độ, metrics
- **Module Tri thức**: wiki, skills tái sử dụng, workflows, đúc kết, tham vấn

Mối quan hệ lõi: **tri thức hỗ trợ mục tiêu**. Khi lập kế hoạch, tra cứu kho tri thức. Khi xong việc, rút bài học ngược lại vào kho.

## Kiến trúc dữ liệu

```
.cockpit/
├── objective.md                 # Mục tiêu, KR/KI, ràng buộc
├── tools.md                     # Danh mục công cụ khả dụng
├── plan.md                      # Milestones + roadmap
├── actions/                     # Mỗi action một file (AXXX-slug.md)
├── knowledge/                   # Trang wiki (decision, pattern, concept, lesson, ...)
│   └── index.md                 # Bảng registry
├── skills/                      # Quy trình tái sử dụng
├── workflows/                   # Chuỗi nhiều skill
├── inbox/                       # Capture thống nhất (thực thi + tri thức)
├── raw/                         # Tài liệu gốc (immutable)
├── log/                         # Nhật ký hàng ngày (append-only)
└── archive/                     # Xóa mềm (khôi phục được)
```

## Hệ thống Skill

Project Cockpit gồm 11 skill, điều phối qua một điểm vào duy nhất.

| Skill | Lớp | Vai trò |
| --- | --- | --- |
| **pk-harness** | Orchestrator | Điểm vào. Đọc state, route intent, chạy inline |
| **pk-init** | Foundation | Tạo `.cockpit/`, thiết lập mục tiêu, công cụ, ràng buộc |
| **pk-analyze** | Foundation | Dashboard read-only: metrics, sức khỏe, phân tích gốc rễ |
| **pk-plan** | Core | Tạo/sửa kế hoạch, milestones, actions, phụ thuộc |
| **pk-track** | Core | Tracking tiến độ, review sâu, xử lý inbox thực thi |
| **pk-reflect** | Core | Rút bài học: recap nhanh hoặc phỏng vấn AAR sâu |
| **pk-capture** | Support | Ghi nhanh vào inbox thống nhất, tự phân loại |
| **pk-distill** | Support | Đúc kết inbox tri thức thành wiki/skills/workflows |
| **pk-consult** | Support | Tra cứu kho tri thức hoặc chạy skills/workflows |
| **pk-lint** | Support | Kiểm tra sức khỏe, gộp dữ liệu, tiến hóa schema |
| **pk-shared** | Foundation | 6 file tham chiếu chuẩn (không chạy độc lập) |

### Phân cấp gọi skill

Các skill tuân theo phân cấp 4 lớp nghiêm ngặt (Orchestrator > Core > Support > Foundation), chỉ gọi một chiều từ trên xuống. Mỗi trường trong `.cockpit/` thuộc đúng một skill sở hữu (SOT ownership), tránh xung đột ghi.

## Khái niệm chính

- **Inbox thống nhất**: Một điểm capture duy nhất. Tự phân luồng sang execution hoặc knowledge dựa trên loại item.
- **Snapshot contract**: Một lần đọc context duy nhất, idempotent, dùng chung cho mọi skill. Không đọc file trùng lặp.
- **Promote lifecycle**: Trang wiki có 3+ lần sử dụng được thăng cấp thành skill/workflow.
- **Graceful degradation**: Skill tự thích ứng khi thiếu file. Chưa có objective? Chạy chế độ tri thức. Chưa có plan? Gợi ý tạo mới.
- **Quality gate**: Kiểm tra nội bộ 3 điểm (cụ thể, giả định ẩn, mâu thuẫn) trước mỗi bước tiếp theo.

## Cách sử dụng

Gọi qua Claude Code:

```
/pk-harness
```

Harness đọc state `.cockpit/` rồi route tới skill phù hợp theo ý định của bạn:

| Bạn nói gì | Skill được gọi |
| --- | --- |
| *(mặc định / "hôm nay")* | `pk-analyze` (dashboard) |
| "track" / "cập nhật" | `pk-analyze` + `pk-track` light |
| "review sâu" / "phân tích sâu" | `pk-analyze` deep + `pk-track` deep |
| "capture" / "ghi nhanh" | `pk-capture` |
| "retro" / "rút bài học" | `pk-reflect` |
| "hỏi kho" / "tra cứu" | `pk-consult` query |
| "chạy skill X" | `pk-consult` run |
| "đúc kết" / "distill" | `pk-distill` |
| "lint" / "kiểm tra" | `pk-lint` |

Bạn cũng có thể gọi trực tiếp từng skill (ví dụ `/pk-capture`, `/pk-plan`).

> [!TIP]
> Lần chạy đầu trong dự án, harness phát hiện chưa có thư mục `.cockpit/` và tự động route tới `pk-init` để khởi tạo workspace.

## Luồng dữ liệu

File [flows reference](skills/pk-harness/references/flows.md) chứa 11 sơ đồ sequence (Mermaid) mô tả mọi luồng tương tác chính: dashboard, tracking, deep review, capture, distill, consult, reflect, lint.

## Cấu trúc dự án

```
personal-assistant/
├── skills/
│   ├── pk-harness/          # Orchestrator + sơ đồ luồng
│   ├── pk-init/             # Khởi tạo
│   ├── pk-analyze/          # Phân tích & dashboard
│   ├── pk-plan/             # Lập kế hoạch
│   ├── pk-track/            # Tracking tiến độ
│   ├── pk-capture/          # Ghi nhanh
│   ├── pk-reflect/          # Rút bài học
│   ├── pk-distill/          # Đúc kết tri thức
│   ├── pk-consult/          # Tham vấn tri thức
│   ├── pk-lint/             # Kiểm tra & bảo trì
│   └── pk-shared/           # Tham chiếu chung (schemas, metrics, SOT, ...)
└── docs/
    └── superpowers/
        ├── specs/           # Tài liệu thiết kế
        └── plans/           # Kế hoạch triển khai
```

## Nguyên tắc thiết kế

1. **Cách ly per-project**: mỗi dự án một `.cockpit/`, không dùng chung toàn cục
2. **SOT ownership nghiêm ngặt**: một trường = một skill duy nhất được ghi
3. **Tri thức phục vụ mục tiêu**: kho tri thức độc lập, nhưng kết nối chặt với planning và tracking
4. **Phân tích read-only**: dashboard không tạo side effect
5. **Hỏi tối thiểu**: snapshot thu hẹp ngữ cảnh, tối đa 1 câu clarify trước khi hành động
