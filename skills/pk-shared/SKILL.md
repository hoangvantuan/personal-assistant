---
name: pk-shared
description: "Reference files dùng chung cho bộ pk-*: schemas, metrics, quality gate, cross-call rules, snapshot contract, SOT ownership. KHÔNG gọi trực tiếp."
---

# PK Shared: Reference files dùng chung

Thư viện tham chiếu canonical cho toàn bộ bộ pk-*. Không có flow riêng. Các skill khác đọc qua path `../pk-shared/references/`.

## References

| File | Nội dung | Skill tiêu thụ chính |
| --- | --- | --- |
| `schemas.md` | Inbox, action, knowledge page, log format | pk-capture, pk-track, pk-distill, pk-lint |
| `metrics.md` | Công thức KR/KI/trend/health | pk-analyze, pk-track |
| `quality-gate.md` | 3 câu hỏi nội bộ khi nhận input từ user | pk-init, pk-plan |
| `cross-call-rules.md` | 4 lớp skill, delegate protocol | pk-harness, pk-track |
| `snapshot-contract.md` | Context preload idempotent | Tất cả pk-* |
| `sot-ownership.md` | Phân vai quyền ghi | Tất cả pk-* |
