---
name: pk-shared
description: "Thư viện reference nội bộ cho bộ pk-* (schemas, metrics, quality gate). KHÔNG BAO GIỜ gọi trực tiếp. Skill này không có flow riêng. Các skill pk-* khác tự đọc file reference khi cần."
---

# PK Shared: Reference files dùng chung

Thư viện tham chiếu canonical cho toàn bộ bộ pk-*. Không có flow riêng. Các skill khác đọc qua path `../pk-shared/references/`.

## References

| File | Nội dung | Skill tiêu thụ chính |
| --- | --- | --- |
| `schemas.md` | Inbox, action, knowledge page, log format, objective, tools, plan, skill/workflow format, promote criteria | pk-capture, pk-track, pk-distill, pk-lint, pk-consult, pk-init, pk-plan, pk-analyze |
| `metrics.md` | Công thức KR/KI/trend/health | pk-analyze, pk-track, pk-lint |
| `quality-gate.md` | 3 câu hỏi nội bộ khi nhận input từ user | pk-init, pk-plan |
| `cross-call-rules.md` | 4 lớp skill, delegate protocol | pk-harness, pk-track |
| `snapshot-contract.md` | Context preload idempotent | Tất cả pk-* |
| `sot-ownership.md` | Phân vai quyền ghi | Tất cả pk-* |
