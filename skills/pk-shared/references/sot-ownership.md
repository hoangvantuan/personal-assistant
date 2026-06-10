# Phân vai SOT (Source of Truth)

Mỗi field SOT chỉ được sửa bởi skill được chỉ định. pk-track deep chỉ ĐỀ XUẤT điều chỉnh cấu trúc, delegate sang pk-init/pk-plan. Bảng canonical duy nhất.

| Field/File | Skill được phép ghi |
| --- | --- |
| objective.md (text, KR/KI targets, period, status, constraints) | `pk-init` |
| tools.md | `pk-init` |
| plan.md (milestones, roadmap) | `pk-plan` |
| actions/*.md (cấu trúc: title, deadline, deps) | `pk-plan` |
| actions/*.md (status, current values) | `pk-track` |
| KR/KI current, plan counters | `pk-track` |
| inbox/ (tạo) | `pk-capture`, `pk-reflect` |
| inbox/ domain=execution (xử lý) | `pk-track` |
| inbox/ domain=knowledge (xử lý) | `pk-distill` |
| knowledge/*.md, knowledge/index.md | `pk-distill` |
| knowledge/index.md usage_count | `pk-distill` (cache từ log) |
| skills/*.md, skills/registry.md | `pk-distill` |
| workflows/*.md, workflows/registry.md | `pk-distill` |
| log/ (append) | `pk-track`, `pk-consult` (usage), `pk-lint` |
| raw/ (tạo, immutable) | `pk-capture`, `pk-reflect` |
| archive/ (actions) | `pk-track` |
| archive/ (knowledge, skills, workflows) | `pk-distill` |
| archive/ (inbox) | `pk-track`, `pk-distill` |
| archive/ (restore) | `pk-lint` |
| SCHEMA.md | `pk-lint` (evolve mode) |
| schema-signals.md | `pk-distill`, `pk-consult` (emit) |

> Bảng này là bản canonical. Mỗi SKILL.md trích subset liên quan. Sửa bảng này trước, subset theo sau.
