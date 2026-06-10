# Cross-call Rules

Skill được phép gọi nhau, 1 chiều theo lớp. Lớp trên gọi được lớp dưới hoặc cùng lớp. Lớp dưới KHÔNG gọi ngược lên.

## 4 lớp

```
Lớp 1 (orchestrator):  pk-harness
Lớp 2 (core):          pk-plan, pk-track, pk-reflect
Lớp 3 (support):       pk-capture, pk-consult, pk-distill, pk-lint
Lớp 4 (foundation):    pk-analyze, pk-init, pk-shared
```

## Cross-call hợp lệ

| Caller (lớp) | Callee (lớp) | Ví dụ |
| --- | --- | --- |
| pk-harness (1) | Mọi skill | Route sang bất kỳ skill nào |
| pk-plan (2) | pk-consult (3) | Tìm knowledge khi tạo action |
| pk-track (2) | pk-capture (3) | Deep review phát hiện pattern → tạo inbox |
| pk-track (2) | pk-consult (3) | Thực thi skill hỗ trợ tracking |
| pk-reflect (2) | pk-capture (3) | Bàn giao candidates sau phỏng vấn |
| pk-reflect (2) | pk-analyze (4) | Lấy metrics cho deep reflect |
| pk-distill (3) | pk-analyze (4) | Đọc metrics khi cần |
| pk-lint (3) | pk-analyze (4) | Sử dụng reachability audit |

## KHÔNG hợp lệ

| Caller | Callee | Lý do |
| --- | --- | --- |
| pk-consult (3) | pk-track (2) | Lớp dưới gọi ngược lên |
| pk-analyze (4) | pk-plan (2) | Lớp dưới gọi ngược lên |
| pk-capture (3) | pk-distill (3) | Cùng lớp: capture CHỈ ghi inbox, distill xử lý inbox (tách vai rõ) |
| pk-init (4) | pk-plan (2) | Foundation không gọi core |

## Delegate Protocol (pre_confirmed)

Khi pk-track deep đề xuất thay đổi cấu trúc:

1. pk-track gom proposals → trình user duyệt batch
2. User duyệt tất cả cùng lúc
3. pk-track gọi pk-plan/pk-init với payload `pre_confirmed: true`
4. pk-plan/pk-init thực thi ngay, không hỏi confirm lại

Payload:
- `apply_via`: "pk-init" / "pk-plan"
- `mode`: mode tương ứng
- `changes`: danh sách (field, from, to)
- `context.reason`: root cause text
- `pre_confirmed`: true

Khi có `pre_confirmed: true`:
1. SKIP ask "Xác nhận? (y/sửa/huỷ)" → ghi file ngay
2. VẪN HIỂN THỊ: block lý do + diff + kèm `(Đã được confirm tại track. Ghi ngay.)`
