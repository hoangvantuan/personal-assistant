# Phân vai SOT (Source of Truth)

Mỗi field SOT chỉ được sửa bởi skill được chỉ định. pk-track deep chỉ ĐỀ XUẤT điều chỉnh cấu trúc, delegate sang pk-init/pk-plan. Bảng canonical duy nhất.

`pk-reflect` và `pk-track` không tự tạo item trong inbox/ và raw/. Mọi item mới bàn giao qua `pk-capture` (một cửa inbox).

| Field/File | Skill được phép ghi |
| --- | --- |
| objective.md (text, KR/KI targets, period, status, constraints, type, review_cycle) | `pk-init` |
| tools.md | `pk-init` |
| plan.md (milestones) | `pk-plan` |
| plan.md (Roadmap: view dẫn xuất từ actions/) | `pk-plan`, `pk-track` (re-render) |
| actions/*.md (cấu trúc: title, due_date, deps, key_result, milestone) | `pk-plan` |
| actions/*.md (status, completed_date, current values) | `pk-track` |
| KR/KI current, plan counters | `pk-track` |
| inbox/ (tạo) | `pk-capture` |
| inbox/ domain=execution (xử lý) | `pk-track` |
| inbox/ domain=knowledge (xử lý) | `pk-distill` |
| knowledge/*.md, knowledge/index.md | `pk-distill` |
| knowledge/index.md usage_count | `pk-distill` (cache), `pk-lint` (rebuild-index) |
| skills/*.md, skills/registry.md | `pk-distill` |
| workflows/*.md, workflows/registry.md | `pk-distill` |
| log/ (append) | `pk-track`, `pk-consult` (usage), `pk-capture`, `pk-distill`, `pk-lint` |
| raw/ (tạo, immutable) | `pk-capture` |
| archive/ (actions) | `pk-track` |
| archive/ (knowledge, skills, workflows) | `pk-distill` |
| archive/ (inbox) | `pk-track`, `pk-distill` |
| archive/ (restore) | `pk-lint` |
| SCHEMA.md | `pk-init` (tạo lần đầu, mode new), `pk-lint` (evolve) |
| schema-signals.md (tạo) | `pk-init` (mode new) |
| schema-signals.md (emit tín hiệu) | `pk-distill`, `pk-consult` |
| schema-signals.md (move sang "Đã xử lý") | `pk-distill` (phiếu promote-candidate), `pk-lint` (evolve, các tín hiệu đã batch) |

Ghi chú:

- Roadmap trong plan.md KHÔNG phải SOT, là view dẫn xuất từ actions/. Không sửa tay, chỉ re-render.
- Tín hiệu schema-signals.md (query-miss, promote-candidate) emit TRỰC TIẾP vào file. Không đi qua inbox. Bất biến "một cửa inbox" chỉ áp cho inbox/ và raw/.
- **usage_count SOT là `log/`**: cột `Usage` trong `knowledge/index.md` chỉ là **cache-hint, CÓ THỂ CŨ**. Khi cần giá trị chính xác (ví dụ: quyết định promote), bắt buộc recompute từ log theo công thức tại `metrics.md` mục "Usage count (canonical)". Cột index dùng cho dashboard nhanh, không dùng cho quyết định.
- `pk-lint` mode fix được phép sinh lại các file dẫn xuất (index, registry).

> Bảng này là bản canonical. Mỗi SKILL.md trích subset liên quan. Sửa bảng này trước, subset theo sau.
