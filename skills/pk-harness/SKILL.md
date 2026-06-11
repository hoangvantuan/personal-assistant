---
name: pk-harness
description: "Orchestrator cho bộ pk-*. Dùng khi yêu cầu phức hợp cần phối hợp nhiều skill (vừa plan vừa track, vừa capture vừa distill), hoặc khi không rõ nên dùng skill pk-* nào. Cũng dùng khi user muốn xem bức tranh toàn cảnh dự án, hoặc yêu cầu workflow nhiều bước liên quan .cockpit/. KHÔNG dùng nếu yêu cầu rõ ràng khớp một skill pk-* cụ thể."
---

# PK Harness: Orchestrator (skill-only)

Entry point quản lý dự án + tri thức. Đọc state, route, chạy skill inline, tổng hợp kết quả.

## Bản đồ skill

| Skill | Lớp | Vai trò | Ghi |
| --- | --- | --- | --- |
| `pk-analyze` | 4 | Phân tích read-only: metrics, dashboard, issues | Không |
| `pk-init` | 4 | Tạo/sửa objective, tools, constraints | Có |
| `pk-plan` | 2 | Tạo/sửa plan, milestones, actions | Có |
| `pk-track` | 2 | Track progress, deep review, inbox execution | Có |
| `pk-capture` | 3 | Ghi nhanh vào inbox thống nhất | Có (inbox) |
| `pk-reflect` | 2 | Rút bài học (light/deep) | Có (qua capture) |
| `pk-distill` | 3 | Đúc kết inbox knowledge → wiki/skills/workflows | Có (knowledge) |
| `pk-consult` | 3 | Tham vấn tri thức (query/run/teach) | Có (log usage) |
| `pk-lint` | 3 | Rà soát, dọn dẹp, schema evolution | Có (sửa/evolve) |
| `pk-shared` | 4 | Quy tắc chung: 6 reference files | Không (tham khảo) |

## Phase 0: Context check

```
if .cockpit/ không tồn tại:
    → chạy pk-init mode new (inline)
    return
else:
    → Phase 1
```

## Phase 1: Snapshot + Intent routing

### Snapshot

Tự đọc context snapshot duy nhất (`../pk-shared/references/snapshot-contract.md`):

```
ls -1 .cockpit/           # Detect cấu trúc
objective.md              frontmatter
tools.md                  toàn bộ
plan.md                   frontmatter
actions/                  scan frontmatter
inbox/                    scan frontmatter
knowledge/index.md        toàn bộ
skills/registry.md        toàn bộ
workflows/registry.md     toàn bộ
Pinned pages              full body
```

### Intent routing

| State | User intent | Chạy skill |
| --- | --- | --- |
| Chưa có .cockpit/ | Bất kỳ | `pk-init` new |
| Có .cockpit/, không objective | Mặc định | `pk-capture` / `pk-consult` / `pk-distill` / `pk-lint` / `pk-reflect` (light) / `pk-analyze` (knowledge-only). Danh sách canonical: `../pk-shared/references/snapshot-contract.md` (Graceful Degradation) |
| Có objective, chưa plan | Mặc định | `pk-plan` new |
| Có objective + plan | **Mặc định / "hôm nay"** | `pk-analyze` (dashboard) |
| | "sửa mục tiêu / KR / KI / constraints" | `pk-init` update-objective |
| | "sửa tools" | `pk-init` update-tools |
| | "thêm action / sửa plan" | `pk-plan` update |
| | "track / cập nhật" | `pk-analyze` → `pk-track` light |
| | "review sâu / phân tích sâu" | `pk-analyze` deep → `pk-track` deep |
| | "capture / ghi nhanh / note" | `pk-capture` |
| | "inbox" | `pk-track` inbox-only (execution) HOẶC `pk-distill` (knowledge) |
| | "rút bài học / retro" | `pk-reflect` light |
| | "AAR / phỏng vấn sâu" | `pk-reflect` deep |
| | "hỏi kho / tra cứu" | `pk-consult` query |
| | "chạy skill / workflow" | `pk-consult` run |
| | "soạn onboarding / dạy lại X / lộ trình học" | `pk-consult` teach |
| | "distill / đúc kết" | `pk-distill` |
| | "lint / kiểm tra / dọn dẹp" | `pk-lint` check |
| | "sửa kho / rebuild / restore" | `pk-lint` fix |
| | "schema review / tiến hoá" | `pk-lint` evolve |

**Intent mơ hồ**: snapshot → thu hẹp → tối đa 1 câu clarify. Không đoán mò, không chạy sai skill.

**Collision**: state gợi ý khác user intent → ưu tiên user intent + xác nhận.

### Inbox routing đặc biệt

User nói "inbox" / "xử lý inbox":
- Kiểm tra inbox count by domain
- Chỉ có execution → `pk-track` inbox-only
- Chỉ có knowledge → `pk-distill`
- Cả hai → hỏi user: "Inbox có N execution + M knowledge. Xử lý bên nào trước?"

## Phase 2: Thực thi inline

Đọc SKILL.md skill đích, theo flow. Khi flow chuyển sang skill khác, đọc tiếp SKILL.md kia (cùng agent, inline).

### Dashboard (mặc định khi có plan)

`pk-analyze` (focus=full). Render dashboard hợp nhất + nhắc review nếu cần.

### Track light

1. `pk-analyze` (focus: track) → analysis
2. `pk-track` light, dùng analysis

### Deep review (chuỗi tuần tự inline)

1. `pk-analyze` deep → root cause
2. `pk-track` deep → đề xuất + confirm
3. Ghi progress
4. Delegate cấu trúc → `pk-init`/`pk-plan` (pre_confirmed)
5. Phát hiện tri thức → `pk-capture` (inbox knowledge)
6. Log review

## Phase 3: Tổng hợp

- Gom kết quả.
- Render tóm tắt: thay đổi gì, next step gì.
- Gợi ý rút bài học nếu flow có thực chất:
  > "Muốn rút bài học phiên này? (chạy pk-reflect)"
  KHÔNG tự chạy. User đồng ý mới chạy.

## Error handling

| Lỗi | Xử lý |
| --- | --- |
| `.cockpit/` không tồn tại | Route pk-init new |
| objective.md rỗng (knowledge-only) | Chỉ route knowledge skills + pk-analyze knowledge-only (danh sách canonical: snapshot-contract.md, Graceful Degradation) |
| File corrupt | Báo cụ thể file, đề xuất sửa |
| Skill flow lỗi giữa chừng | Báo bước lỗi, giữ thay đổi đã ghi |

## Tham khảo

- Sơ đồ luồng: `references/flows.md`
- Snapshot contract: `../pk-shared/references/snapshot-contract.md`
- Cross-call rules: `../pk-shared/references/cross-call-rules.md`
