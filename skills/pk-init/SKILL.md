---
name: pk-init
description: "Khởi tạo dự án mới hoặc sửa KR target/mục tiêu/constraints/tools (cấu trúc mục tiêu). Tạo cấu trúc .cockpit/ lần đầu. Dùng khi user bắt đầu dự án từ đầu, sửa OKR, thêm/bớt công cụ, thay đổi constraints, hoặc cần thiết lập lại mục tiêu. KHÔNG dùng khi cập nhật KR current tiến độ hàng ngày (dùng pk-track)."
---

# PK Init: Khởi tạo + cập nhật objective, tools, constraints

SOT chính: `objective.md` và `tools.md`.

## Modes

| Mode | Trigger | Mô tả |
| --- | --- | --- |
| `new` | Chưa có `.cockpit/` | Tạo toàn bộ cấu trúc + SCHEMA.md + registries |
| `update-objective` | Sửa objective/KR/KI/start_date/end_date/capacity/budget/gaps_risks | Sửa objective.md |
| `update-tools` | Sửa tools (thêm/bỏ công cụ) | Sửa tools.md |

## SOT quyền ghi

| File | Fields |
| --- | --- |
| objective.md | Objective text, KR/KI targets, start_date, end_date, status, capacity, budget, gaps_risks, type, review_cycle |
| tools.md | Bảng tools 4 cột |
| SCHEMA.md | Chỉ khi mode new (tạo lần đầu) |
| schema-signals.md | Chỉ khi mode new (tạo lần đầu) |
| AGENTS.md block | Tạo lần đầu (mode new); regenerate là việc pk-lint |

> Subset của bảng canonical `../pk-shared/references/sot-ownership.md`.

## Nguyên tắc

- Hỏi từng câu một, không hàng loạt.
- BẮT BUỘC confirm bảng trước ghi. **Ngoại lệ**: khi nhận payload `pre_confirmed: true` từ pk-track (delegate protocol), SKIP confirm, ghi ngay; vẫn hiển thị block lý do + diff + `(Đã được confirm tại track. Ghi ngay.)`. Xem Delegate Protocol tại `../pk-shared/references/cross-call-rules.md`.
- Quality Gate 3 câu trước mỗi follow-up (`../pk-shared/references/quality-gate.md`).
- **Snapshot Contract** (`../pk-shared/references/snapshot-contract.md`):
  - Mode `new`: KHÔNG ensure snapshot (.cockpit/ chưa tồn tại, đây là bước tạo cấu trúc). Marker `SNAPSHOT_LOADED` chưa cần.
  - Mode `update-objective` / `update-tools`: cần ensure snapshot trước khi đọc objective.md/tools.md (theo Snapshot Contract, mục "Nguyên tắc idempotent"). Phiên chưa có marker → tự nạp full rồi đặt marker.
- Solo only: 1 user, 1 objective.
- Đề xuất + lý do, user quyết.

## Flow: mode new

### Bước 1: Tạo cấu trúc thư mục

```bash
mkdir -p .cockpit/{actions,inbox,raw,log,archive/{actions,inbox,knowledge,skills,workflows},knowledge,skills,workflows}
```

### Bước 2: Sinh SCHEMA.md

Chứa quy ước dữ liệu:
- Cấu trúc thư mục `.cockpit/`
- Quy ước naming (actions: AXXX-slug.md, inbox: YYYY-MM-DD-HHmm-slug.md, knowledge: {type}-{slug}.md, log: YYYY-MM-DD.md)
- Wiki page types (6 types: decision, pattern, concept, troubleshooting, lesson, resource)
- Frontmatter bắt buộc theo loại file

Nội dung khởi tạo sinh theo quy ước canonical: `../pk-shared/references/schemas.md`.

### Bước 3: Sinh registries rỗng

- `knowledge/index.md`: bảng 8 cột (Slug, Type, Title, Tags, Status, Pinned, Usage, Updated)
- `skills/registry.md`: bảng 6 cột (Skill, Mô tả, Khi nào dùng, Version, Tags, Cập nhật)
- `workflows/registry.md`: bảng 7 cột (Workflow, Mô tả, Khi nào dùng, Skills dùng, Version, Tags, Cập nhật)
- `schema-signals.md`: header + 2 section (Đang chờ xử lý / Đã xử lý)

### Bước 4: Hỏi tools

Hỏi user: "Dự án dùng công cụ gì? (CLI, MCP, API, manual)". Ghi vào `tools.md` theo format canonical tại `../pk-shared/references/schemas.md` (tools.md format).

### Bước 5: Hỏi objective (OPTIONAL)

Hỏi user: "Mục tiêu dự án là gì? (Bỏ qua nếu chưa sẵn sàng, chạy knowledge-only mode)".

Nếu user cung cấp:
- Thu thập objective text + KR/KI
- Hỏi `start_date` và `end_date` (ngày bắt đầu và kết thúc kỳ)
- Hỏi `capacity` (giới hạn năng lực), `budget` (ngân sách), `gaps_risks` (khoảng trống/rủi ro), ghi vào 3 field riêng
- Hỏi `type`: project hoặc ongoing. Nếu ongoing, hỏi tiếp `review_cycle` (đề xuất 14 ngày)
- Ghi `objective.md`

Nếu user skip:
- Tạo `objective.md` rỗng (chỉ frontmatter `status: empty`)
- KHÔNG tạo `plan.md`
- Hệ thống hoạt động ở knowledge-only mode

Nội dung `objective.md` sinh theo format canonical tại `../pk-shared/references/schemas.md` (Objective file format).

### Bước 6: Attach hướng dẫn vào AGENTS.md

Ghi block hướng dẫn vào `AGENTS.md` tại root workspace theo format canonical `../pk-shared/references/schemas.md` (mục "AGENTS.md block format").

- Bao block bằng marker `<!-- personal-assistant:start -->` / `<!-- personal-assistant:end -->`.
- Idempotent: chưa có file → tạo mới; có file chưa marker → append; có marker → replace giữa hai marker. (Chi tiết quy tắc ghi: xem canonical.)
- Nội dung: mô tả nhiệm vụ tổng thể hệ Project Cockpit + 4 dòng `@.cockpit/...` auto-load. KHÔNG copy description per-skill.
- KHÔNG tạo/sửa CLAUDE.md. Việc nối `CLAUDE.md → AGENTS.md` (để Claude Code expand `@`) thuộc về user.

## Flow: mode update-objective

1. Đọc objective.md hiện tại
2. Hỏi user thay đổi gì
3. Quality Gate mỗi follow-up
4. Confirm bảng diff
5. Trước khi ghi đè body: đọc body hiện tại (bất biến: `../pk-shared/references/snapshot-contract.md`, mục "Bất biến đọc body trước khi ghi body"). Ghi file.

## Flow: mode update-tools

1. Đọc tools.md hiện tại
2. Hỏi user thay đổi gì
3. Confirm
4. Trước khi ghi đè body: đọc body hiện tại (bất biến: `../pk-shared/references/snapshot-contract.md`, mục "Bất biến đọc body trước khi ghi body"). Ghi file.

## Quy tắc

- KHÔNG ghi file trước confirm (trừ pre_confirmed: xem Ngoại lệ ở Nguyên tắc).
- KR không SMART → chỉ rõ thiếu gì + gợi ý sửa.
- Không tạo plan.md hay action files (việc pk-plan).
- Không sửa action status (việc pk-track).
- Không tạo/sửa knowledge pages (việc pk-distill).
